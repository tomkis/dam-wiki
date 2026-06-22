---
source: dam-agents/dam
commit: 9d1bc9990fba55f43d60b0aad453b188af5896a8
files:
  - docs/architecture/logging.md
  - packages/api-server/src/core/logger.ts
  - packages/api-server/src/core/security-log.ts
  - packages/api-server/src/index.ts
  - packages/api-server/src/modules/audit/sagas/audit-log.ts
  - packages/controller/main.go
updated: 2026-06-22
---

# Logging & Audit Trail

The [api-server](../sources/api-server.md) logs through a single process-wide
**Pino** logger, configured once at startup. Its first and primary consumer is a
**security audit trail** — a structured record at every security-relevant
decision point, designed so a forensic investigation can reconstruct *who did
what, to what, and whether it was allowed* (`docs/architecture/logging.md:7-9
@c307f40`).

The audit trail is the orthogonal counterpart to
[usage-tracking](../concepts/usage-tracking.md): usage is **pseudonymized
analytics** in Postgres (a database leak yields only opaque HMAC hashes); the
audit trail is **real-identity forensics** on stdout (an investigator can
attribute directly). The two never share actor handling
(`docs/architecture/logging.md:9 @c307f40`,
`packages/api-server/src/core/security-log.ts:14-18 @c307f40`).

## The base logger

A single Pino instance is created at module load and configured once at startup
via `configureLogger(...)` (`packages/api-server/src/core/logger.ts:54-67
@c307f40`). Components reach it through `getLogger()`, which reads the current
instance at call time so reconfiguration applies, rather than threading it
through every service factory (`packages/api-server/src/core/logger.ts:69-72
@c307f40`).

- **One JSON object per line on stdout.** The common levels
  `error/warn/info/debug` are the only knob — there is no per-feature toggle.
  Visibility is the operator's level choice (`docs/architecture/logging.md:7
  @c307f40`, `packages/api-server/src/core/logger.ts:18,28-29 @c307f40`).
- **Build identity on every line.** Startup stamps `appVersion` as a base field
  (`packages/api-server/src/index.ts:104-107 @c307f40`), so a line attributes to
  a build across restarts and upgrades. Pino's pid/hostname noise is dropped —
  the log pipeline annotates pod identity
  (`packages/api-server/src/core/logger.ts:30-32 @c307f40`).
- **It is a general logger.** Any component may use it; the audit trail is just
  the substrate's first consumer
  (`packages/api-server/src/core/logger.ts:8-10 @c307f40`).

## The audit record

`securityLog(level, event, fields)` is a thin, typed wrapper over
`getLogger()[level](fields, event)` — **not** a new level or a separate stream
(`packages/api-server/src/core/security-log.ts:79-85 @c307f40`). The dotted
`event` (e.g. `egress.decision`, `secret.create`, `authn.deny`) is the log
message; the typed `fields` are merged into the line
(`docs/architecture/logging.md:13 @c307f40`):

- **`category`** — the coarse class, one of `authn`, `authz`, `egress`,
  `approval`, `authz-list`, `credential`, `channel`, `resource`, `privileged`
  (`packages/api-server/src/core/security-log.ts:27-36 @c307f40`). Because
  Kubernetes merges stdout and stderr into one pod log, a shipper isolates the
  trail by **filtering on this field**, not on the stream
  (`docs/architecture/logging.md:15 @c307f40`,
  `packages/api-server/src/core/logger.ts:12-13 @c307f40`).
- **`actor` / `actorKind`** — the raw, **un-pseudonymized** Keycloak `sub`, an
  agent id, `system:<component>`, an external id, or `null`; tagged
  `user | agent | system | external`
  (`packages/api-server/src/core/security-log.ts:38,54-56 @c307f40`). This raw
  actor is the deliberate difference from the usage subsystem's HMAC-hashed
  events (`packages/api-server/src/core/security-log.ts:14-18 @c307f40`).
- **`result`** (`success | failure`) and **`decision`**
  (`allow | deny | hold | expired`) are **separate axes** — an execution failure
  is not a policy deny, so the canonical "show me every deny" query stays
  unambiguous (`packages/api-server/src/core/security-log.ts:58-62 @c307f40`,
  `docs/architecture/logging.md:17 @c307f40`).
- **`correlationId`** ties a multi-site flow together: a credentialed-egress
  hold (`egress.hold`), the human verdict (`approval.verdict`), and the resolved
  decision (`egress.decision`) all carry the same pending-approval id
  (`packages/api-server/src/core/security-log.ts:68-69 @c307f40`,
  `docs/architecture/logging.md:18 @c307f40`).
- **`agentId`, `target`, `sourceIp`, `reason`, `detail`** round out the line;
  `detail` is shallow and value-free
  (`packages/api-server/src/core/security-log.ts:63-74 @c307f40`).

**Level mapping:** deny/fail → `warn`, allow/success/mutation → `info`, internal
failure on a security path → `error`
(`packages/api-server/src/core/security-log.ts:9-12 @c307f40`,
`docs/architecture/logging.md:21 @c307f40`).

## Redaction

**Redaction is the caller's contract.** Records never carry token/secret/PAT
values, refresh tokens, raw JWTs, or raw prompts — only metadata
(`hasRefresh`, `secretId`, env key *names*, byte counts, a resolved file path).
The bus saga projects explicit fields per event rather than spreading a whole
domain event, which would leak the raw prompt carried by `ForeignReplyReceived`
(`packages/api-server/src/core/security-log.ts:20-23 @c307f40`,
`packages/api-server/src/modules/audit/sagas/audit-log.ts:26-28 @c307f40`).

As **defense-in-depth**, the base logger also censors a few well-known
credential keys (`token`, `authorization`, `password`, `secret`, `refreshToken`
at top level and one nesting deep) to `[REDACTED]`, so a careless field can't
leak a token onto the forensic stream
(`packages/api-server/src/core/logger.ts:33-50 @c307f40`). The WS edge strips
`?token=` from a path before logging it
(`packages/api-server/src/core/security-log.ts:66 @c307f40`,
`docs/architecture/logging.md:23 @c307f40`).

The audit stream is therefore PII and is governed at the log sink — the inverse
trade-off from [usage analytics](../concepts/usage-tracking.md), which is
hashed-at-rest precisely so it need not be
(`packages/api-server/src/core/security-log.ts:14-18 @c307f40`). See also
[security & credentials](../concepts/security-and-credentials.md).

## How the trail is produced

Two **disjoint** mechanisms feed the one logger, so no event is double-logged
(`docs/architecture/logging.md:27 @c307f40`,
`packages/api-server/src/modules/audit/sagas/audit-log.ts:22-24 @c307f40`):

- **Bus saga** (`startAuditLogSaga`) — subscribes the in-process domain event
  bus for the discrete success/observation events that already carry a real
  actor: `ChannelTurnRelayed`, `ScheduleFired`, `FilesImported`,
  `ForeignReplyReceived` (cross-identity turn, prompt omitted), and the `Fork*`
  events (`packages/api-server/src/modules/audit/sagas/audit-log.ts:34-44,66-152
  @c307f40`). Each handler **projects** explicit fields and never spreads the
  event (`packages/api-server/src/modules/audit/sagas/audit-log.ts:26-28
  @c307f40`); a projection bug logs `audit.saga_error` rather than tearing down
  the subscription (`packages/api-server/src/modules/audit/sagas/audit-log.ts:45-53
  @c307f40`). It deliberately does **not** subscribe `UserAuthenticated`, which
  fires on every authenticated request — the usage saga consumes that one,
  collapsing it to a single row per day; a successful login is recorded
  authoritatively by Keycloak's own authentication-event log
  (`packages/api-server/src/modules/audit/sagas/audit-log.ts:58-64 @c307f40`).
- **Direct calls** at every decision/denial/mutation site not on the bus — the
  majority, and all denials. Each site logs at the application/service layer or
  the transport edge, where the actor is in scope; never in the pure domain layer
  (`docs/architecture/logging.md:30 @c307f40`).

## Coverage

| Surface | Representative events |
|---|---|
| Auth edge | `authn.deny` (bad/missing token), `authz.deny` (missing role). Successful logins are not logged here — Keycloak owns that record. |
| HTTP / WS edge | `authz.owner_mismatch`, `ws.authn_deny` / `ws.owner_mismatch` / `ws.terms_block`, `relay.attach` |
| Credentialed egress (HITL) | `egress.decision` (allow/deny/expired), `egress.hold`; identity-unresolved and ext-authz transport denials |
| Approvals | `approval.verdict` (approve/deny once/permanent/host) |
| Authorization lists | `agent.allowed_users_set` (added/removed diff), `egress_rule.create\|update\|revoke\|preset`, `secret.grants_set`, `connection.grants_set` |
| Credentials | `secret.create\|update\|delete`, `oauth.token_mint`, `connection.create\|delete`, `secret.orphan_cleanup_failed` |
| Channels | `channel.authz` / `channel.authz_deny`, `channel.turn` (prompt omitted), `channel.foreign_turn.begin` (prompt omitted), `identity.link`, `channel.outbound`, `channel.thread_revoked` |
| Privileged | `skill.install` / `skill.uninstall` / `skill.publish`, `schedule.create\|toggle\|delete`, `usage.inspect` / `usage.inspect.deny`, `agent.create\|update\|delete\|restart\|wake` |

(`docs/architecture/logging.md:34-43 @c307f40`)

## Invariants

- **Single process, single stdout.** The public API, harness, and ext-authz
  gRPC apps run in one process, so one logger configuration and one stream cover
  the whole api-server (`docs/architecture/logging.md:47 @c307f40`).
- **Once per replica.** The domain bus is in-process; each replica's saga logs
  only its own events. Moving domain events onto the cross-replica Redis bus
  would require dedup, or every replica would duplicate every line. The
  api-server runs single-replica today
  (`docs/architecture/logging.md:48 @c307f40`,
  `packages/api-server/src/modules/audit/sagas/audit-log.ts:30-32 @c307f40`).
- **Non-blocking.** The egress gate logs on the proxy's request-blocking hop;
  the writer must never block or throw into a request path
  (`docs/architecture/logging.md:49 @c307f40`).

## Controller logging

The controller logs through Go's standard-library **`log/slog`**, configured
once at startup in `setupLogger()` (`packages/controller/main.go:99-112
@9d1bc99`). Output is one JSON object per line on **stderr** at the `LOG_LEVEL`
level (`debug|info|warn|error`, default `info`); `debug` surfaces per-reconcile
phase timing (`packages/controller/main.go:218,259 @9d1bc99`). As with the
api-server, the level is the only knob — there is no per-feature toggle. Stderr
rather than stdout reflects that controller lines are pure diagnostics, not
program output; Kubernetes merges both streams into the one container log, so a
collector sees it either way (`docs/architecture/logging.md:53 @9d1bc99`).

**No audit trail.** The controller acts only under its own ServiceAccount
against the K8s API, never on behalf of a user, so there is no real actor to
attribute — the audit trail is solely an api-server concern
(`docs/architecture/logging.md:55 @9d1bc99`).

**Zero-code OTel instrumentation.** The controller registers no in-process
OpenTelemetry SDK and no global `TracerProvider`. This keeps it ready for the
OTel Operator's eBPF auto-instrumentation sidecar: the sidecar's `log/slog`
probe captures the JSON records and exports them with trace correlation. Wiring
an in-process `TracerProvider` would conflict with the injected auto-SDK and
break that correlation, so the binary deliberately stays OTel-free
(`docs/architecture/logging.md:57 @9d1bc99`).

## See also

- [Usage tracking](../concepts/usage-tracking.md) — the pseudonymized analytics
  counterpart in Postgres.
- [Security & credentials](../concepts/security-and-credentials.md) — the
  decision points the audit trail records.
- [api-server](../sources/api-server.md) — the host process.
- [controller](../sources/controller.md) — the Go reconciler that produces
  controller logs.
