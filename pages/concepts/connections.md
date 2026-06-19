---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files:
  - docs/architecture/connections.md
  - packages/api-server-api/src/modules/connections/types.ts
  - packages/agent-runtime-api/src/modules/runtime/types.ts
  - packages/agent-runtime-api/src/modules/runtime/router.ts
updated: 2026-06-19
---

# Connections

A **Connection** is everything an agent needs to talk to one external
integration — credentials, hosts to reach, config files to author, MCP entries
to expose, skills to install (`docs/architecture/connections.md:7 @c307f40`).
**Connection Templates** are code-level catalog entries shipping defaults;
granting a Connection to an Agent materializes its **Contributions** into the
right destinations (`docs/architecture/connections.md:7 @c307f40`).

The subsystem spans three bounded contexts
(`docs/architecture/connections.md:11-13 @c307f40`):

- **api-server — Connections context**: owns templates, Connections, and grants;
  computes per-agent Contribution sets and routes each Contribution to its rail.
- **api-server — Runtime Delivery context**: owns the outbox table, events
  table, delivery worker, the `runtime.applyState` call into agents, and the
  `runtime.hello` callback from agents.
- **agent-runtime — Runtime Channel context**: receives `applyState`, dispatches
  Contributions to per-kind drivers, processes events in order, reconciles
  on-disk state to match the snapshot, and calls back to `hello` on boot.

See [platform topology](platform-topology.md) for where these components sit,
[agent lifecycle](agent-lifecycle.md) for boot/wake, and
[security and credentials](security-and-credentials.md) for the credential rail.

## The two rails

A single grant produces Contributions of several kinds, and they do **not** all
travel the same path (`docs/architecture/connections.md:15 @c307f40`):

- **Egress rail** — `egress-allow` and `egress-inject` Contributions sync into
  Postgres `egress_rules` and are read live by Envoy `ext_authz`. The agent never
  sees these; `egress-inject` additionally carries a credential the gateway
  injects on the wire (`docs/architecture/connections.md:31,199-200 @c307f40`).
- **Runtime channel** — `env`, `file`, `mcp-entry`, and `skill-ref` travel the
  runtime channel as a state push. Moving `env` here (formerly a
  controller-render/pod-roll rail) means a grant change no longer rolls the agent
  pod (`docs/architecture/connections.md:31 @c307f40`).

The rail is a property of the **kind**, not the Connection: one GitHub Enterprise
grant produces `egress-allow` (egress rail) plus `env` + `file` (runtime
channel), flowing independently (`docs/architecture/connections.md:205 @c307f40`).

| Kind | Rail | Agent sees it? |
|---|---|---|
| `env` | runtime channel | yes (placeholder env merged at spawn) |
| `file` | runtime channel | yes (driver materializes file) |
| `mcp-entry` | runtime channel | yes (driver writes MCP config) |
| `skill-ref` | runtime channel | yes (per-version installer) |
| `egress-allow` | Postgres `egress_rules` → Envoy | no |
| `egress-inject` | `egress_rules` → Envoy + gateway wire-inject | no |

(`docs/architecture/connections.md:196-203 @c307f40`)

## Core model

### Connection Template

A code-level catalog entry. **Premade** templates (GitHub, Anthropic, Spotify,
Linear MCP, …) ship full defaults — auth flow, hosts, scopes, recommended
contributions. **Custom** templates (Custom MCP, Custom OAuth, Custom Header)
ship the *shape* but leave the integration's identity for the user to fill in
(`docs/architecture/connections.md:63 @c307f40`). Two display-axis attributes,
`category` (`app`/`mcp`/`other`) and `isCustom`, drive UI grouping
(`docs/architecture/connections.md:65-72 @c307f40`); the category enum is defined
at `packages/api-server-api/src/modules/connections/types.ts:6 @c307f40`. Adding
a new integration is one code entry; a template's `build()` projects user inputs
into the concrete `auth` + `contributions[]`
(`docs/architecture/connections.md:74 @c307f40`). Some templates (Spotify, Slack,
YouTube, Google) are hidden from regular users client-side and revealed by
testers via `platformConnections.showInternal()` or tapping the version string
five times (`docs/architecture/connections.md:80 @c307f40`).

### Connection

Every Connection has a uniform shape regardless of category or auth mode
(`docs/architecture/connections.md:84 @c307f40`), validated by the `connection`
Zod schema (`packages/api-server-api/src/modules/connections/types.ts:45-53
@c307f40`): `id`, `ownerId`, `templateId`, `name`, `inputs` (raw user values for
re-render), `auth`, and `contributions[]`.

`auth` is a discriminated union over three modes
(`packages/api-server-api/src/modules/connections/types.ts:37-41 @c307f40`):

- **`oauth`** — client identity, references to stored refresh/access tokens, and
  granted scopes (`.../types.ts:9-23 @c307f40`).
- **`header`** — a `valueRef` to the stored secret plus `headerName` and
  `valueFormat` to inject (`.../types.ts:25-30 @c307f40`).
- **`none`** (`.../types.ts:32-34 @c307f40`).

Token references point at the per-Connection K8s Secret, never inline secret
material; auth is kept separate from contributions because credentials have their
own acquisition and refresh lifecycle
(`docs/architecture/connections.md:98 @c307f40`). A **header** connection's
credential can be **updated in place** — it re-bakes the SDS files and rewrites
them with the new value onto the same Secret, preserving identity,
contributions, and grants, with no Agent-spec patch or pod roll because the live
value is read gateway-side via Envoy SDS
(`docs/architecture/connections.md:100 @c307f40`). OAuth connections rotate
through their refresh flow instead.

### Contribution

A typed unit a Connection emits when granted — a discriminated union over `kind`
(`packages/agent-runtime-api/src/modules/runtime/types.ts:79-87 @c307f40`). The
kinds are enumerated in the runtime contract
(`.../runtime/types.ts:3-10 @c307f40`):

- **`env`** — an env var (with a credential *placeholder*, not the secret)
  (`.../runtime/types.ts:30-34 @c307f40`).
- **`egress-allow`** — permission to reach a host, optionally path-scoped
  (`.../runtime/types.ts:36-40 @c307f40`).
- **`egress-inject`** — an allowed host plus a credential the gateway injects as
  a header or query param (`.../runtime/types.ts:42-55 @c307f40`).
- **`file`** — a config file with a `format` and a `mergeMode`
  (`.../runtime/types.ts:57-63 @c307f40`).
- **`mcp-entry`** — an MCP server to expose, with optional headers
  (`.../runtime/types.ts:65-69 @c307f40`).
- **`skill-ref`** — a skill source pinned to a `version`
  (`.../runtime/types.ts:71-76 @c307f40`).

New kinds are added by extending the union and gating on agent capabilities
(`docs/architecture/connections.md:113 @c307f40`).

### Event

A one-shot directive the agent executes through a per-kind handler. Each event
carries a stable `id` (the dedupe key), a `kind`, a kind-specific `payload`, the
agent-monotonic `version` slot it occupies, and an `expiresAt` TTL
(`docs/architecture/connections.md:117 @c307f40`;
`packages/agent-runtime-api/src/modules/runtime/types.ts:97-102 @c307f40`). The
kinds are `trigger`, `schedule-reset`, and `workspace-seed`
(`.../runtime/types.ts:13-17 @c307f40`). Unlike contributions, **all event kinds
are built-in to every agent** and advertised on `hello` from the schema enum;
capability filtering applies to contributions, not events
(`docs/architecture/connections.md:121 @c307f40`).

## The runtime channel

Two routes between api-server and agent-runtime, prefixed by protocol-major
version (`runtime.v1.*`). Adding a new contribution kind, event kind, or optional
field stays on `v1`; capability flags carry the gate, and new majors only on a
semantic break (`docs/architecture/connections.md:209 @c307f40`). The
agent-runtime router exposes `v1.applyState`
(`packages/agent-runtime-api/src/modules/runtime/router.ts:4-12 @c307f40`).

The wire payload carries (`docs/architecture/connections.md:51-56 @c307f40`):

- **`version`** — a per-agent monotonic counter, the single ack cursor; bumped on
  any contribution edit or event insert.
- **`state`** — the agent's full desired Contribution snapshot plus a
  deterministic `hash` that short-circuits no-op pushes
  (`.../runtime/types.ts:159-163 @c307f40`).
- **`events`** — ordered one-shot directives processed in order through per-kind
  handlers.

### `applyState` — server → agent

The server sends the cursor, full desired state, and currently pending events in
order; the agent reconciles contributions by diff and runs events through
per-kind handlers (`docs/architecture/connections.md:241 @c307f40`). The reply is
a **discriminated outcome**, not a bare ack
(`packages/agent-runtime-api/src/modules/runtime/types.ts:172-189 @c307f40`):

- **`ok`** — processed; returns `appliedVersion`, the resulting `appliedHash`
  (null until the first clean settle), `settledEvents`, and any per-driver
  `failures`. A failure leaves that driver's slice unsettled for redelivery
  without blocking the rest (`.../runtime/types.ts:174-181 @c307f40`).
- **`stale`** — contributions already at or beyond the requested version, so
  reconciliation was skipped; the agent still applies unseen events and reports
  which settled (`.../runtime/types.ts:183-188 @c307f40`).

Concurrent dispatches from different replicas race naturally — the agent rejects
versions older than its applied cursor (last-version-wins), surfacing as `stale`
(`docs/architecture/connections.md:248 @c307f40`).

### `hello` — agent → api-server catch-up

Called on boot, on wake from hibernation, and on any agent-side reconnect
(`docs/architecture/connections.md:252 @c307f40`). Its input reports
`lastAppliedVersion`/`lastAppliedHash`, `protocolVersion`, `agentRuntimeVersion`,
and `capabilities` (`.../runtime/types.ts:191-198 @c307f40`). `hello` never
carries state itself — if the reported cursor is behind it enqueues a worker
dispatch, and the catch-up arrives as an ordinary `applyState`; the returned
`events` array is always empty today
(`docs/architecture/connections.md:252,266 @c307f40`). It is read-only with
respect to the outbox (`docs/architecture/connections.md:283 @c307f40`).

### Per-kind event handlers (agent-side)

Each event kind has a built-in handler in the agent-runtime event loop. The loop
owns `id`-based dedupe *before* the handler runs (applied-version cursor plus a
per-key last-run timestamp in the local state store); the handler does the work
(open an ACP session for `trigger`, clone the seed repo for `workspace-seed`) and
does **not** touch `runtime_events`
(`docs/architecture/connections.md:287-291 @c307f40`). A handler failure leaves
the event unsettled for redelivery until it succeeds or expires
(`docs/architecture/connections.md:291 @c307f40`).

## Transactional outbox + worker delivery

State changes write to Postgres first, then a worker dispatches over the channel.
Two tables back this (`docs/architecture/connections.md:339-340 @c307f40`):

- **`runtime_state_outbox`** — one row per agent (last-write-wins, coalesced by
  agent), carrying `version`, `last_enqueued_at`, `last_applied_version`,
  `last_applied_hash`, `last_applied_at`.
- **`runtime_events`** — one row per pending event, each with its own `version`
  slot, `expires_at`, and `dispatched_at`.

### Mutation transaction

Every state-affecting handler commits the domain mutation, bumps the agent's
version, and upserts the outbox row **atomically**, then enqueues a BullMQ job
with a stable `jobId` (`state:<agentId>`) for natural coalescing
(`docs/architecture/connections.md:344-358 @c307f40`). The user-facing response
returns immediately and does **not** depend on agent reachability; if the enqueue
or Redis job is lost, the cron sweep re-enqueues the row
(`docs/architecture/connections.md:357,373 @c307f40`).

### Worker

A BullMQ Worker on every api-server replica consumes the single `state` queue;
BullMQ owns the dispatch loop, retry/backoff, and stalled-job recovery, while the
platform code is the handler (`docs/architecture/connections.md:377 @c307f40`).
The handler loads the outbox row, checks agent running-state from an in-memory
cache fed by the ConfigMap watch (never a direct K8s call), computes the state
slice + non-dispatched events, POSTs `applyState`, and on the apply outcome
stamps `last_applied` and `dispatched_at` up to the acked cursor
(`docs/architecture/connections.md:379-402,415 @c307f40`). When the agent is not
running, a **plain** dispatch exits clean (cron sweep re-enqueues later); a
**`hello`-triggered** dispatch instead throws to fast-retry on backoff, so fresh
config lands in ~a second instead of waiting a sweep tick
(`docs/architecture/connections.md:404,415 @c307f40`).

### Cron sweep and Redis-down behavior

A job runs every minute and (1) re-enqueues stale outbox rows where
`last_enqueued_at > last_applied_at` and older than the sweep interval, and (2)
deletes expired, never-dispatched `runtime_events` rows as `dropped-expired`
(`docs/architecture/connections.md:408-411 @c307f40`). Redis is only the signal
path with relaxed durability; a Redis outage may drop pending jobs, but no events
are lost because the outbox + events tables are in Postgres — latency just
degrades from sub-second to ≤ sweep-interval
(`docs/architecture/connections.md:419 @c307f40`).

### Event lifecycle and crash safety

Idempotency lives in two places: the handler's uniqueness constraint (prevents a
double side effect) and the worker's cursor stamp (prevents redelivery once
acked) (`docs/architecture/connections.md:297 @c307f40`). If the agent runs a
handler but crashes before acking, no rows are stamped; on redelivery the event
loop consults its PVC-persisted local state store and settles the already-run
event without re-firing. Re-fire is possible only if a crash lands between the
side effect and the state-store write
(`docs/architecture/connections.md:319-321 @c307f40`).

## Agent-side: drivers and the manifest

Every agent image ships a `runtime-manifest.yaml` declaring which impl handles
each Contribution kind, plus any custom impls under `extensions.impls`. The kinds
advertised on `hello` are derived from this — contribution kinds from the driver
keys, event kinds from the built-in set — not declared separately; the manifest is
validated against a versioned schema at boot (fail-fast)
(`docs/architecture/connections.md:425,463 @c307f40`). Custom impl names may not
collide with built-in names (`file`, `skill-install`, …); collision fails loud at
boot (`docs/architecture/connections.md:465 @c307f40`).

Two **built-in impls** (`docs/architecture/connections.md:469-472 @c307f40`):

- **`file`** — drives the `file` kind directly and `mcp-entry` via composition; a
  Format (`yaml`/`json`/`text`/`ini`) × MergeMode
  (`overwrite`/`section-marker`/`key-targeted`/`yaml-fill-if-missing`) matrix
  (enums at `packages/agent-runtime-api/src/modules/runtime/types.ts:20-28
  @c307f40`).
- **`skill-install`** — drives `skill-ref`; resolves the source URL, fetches at
  version through the gateway, materializes into skill paths, and removes vanished
  skills on reconciliation.

### Driver reconciliation

`applyState` delivers the *full* snapshot; the dispatcher groups contributions by
kind and calls each driver's `apply(contributions, ctx)`, which diffs the desired
set against on-disk (or its own per-kind state file), adds/updates/removes, and
returns a per-driver outcome (`docs/architecture/connections.md:476-481
@c307f40`). Removal semantics depend on merge mode: `overwrite`,
`section-marker`, and `key-targeted` remove cleanly, while `yaml-fill-if-missing`
is an additive-only legacy carve-out; new file producers must pick a remove-safe
mode (`docs/architecture/connections.md:482 @c307f40`).

## Versioning and capability negotiation

Four versions coexist (`docs/architecture/connections.md:502-507 @c307f40`):
`protocolVersion` (route prefix + hello, bumps on a wire break),
`manifestVersion` (manifest schema, independent), `agentRuntimeVersion`
(diagnostic-only image identity), and the per-agent `version` (the single ack
cursor). **Forward-compat** — older agent on newer server — is the supported
direction: the server keeps every `runtime.v1.*` route operational across
additive changes and the agent parses leniently
(`docs/architecture/connections.md:511 @c307f40`). A newer agent on an older
server is rare and fails loud on a 404
(`docs/architecture/connections.md:513 @c307f40`).

**Capability negotiation**: the agent's `hello` declares which Contribution and
Event kinds it supports (`capabilities`,
`packages/agent-runtime-api/src/modules/runtime/types.ts:149 @c307f40`); the
api-server drops unsupported items at send time (`dropped-unsupported` metric).
The UI surfaces the gap at grant time — e.g. granting GitHub to an agent without
`skill-ref` support warns that the connection grants envs + hosts but not skills
(`docs/architecture/connections.md:517-519 @c307f40`).

## Key invariants

- **Mutation handlers never wait on agent reachability** — the response returns
  after the local transaction + enqueue
  (`docs/architecture/connections.md:539 @c307f40`).
- **Postgres is the source of truth** — every change has a durable
  representation before any wire activity; BullMQ and channel calls are
  signal/delivery only (`docs/architecture/connections.md:540 @c307f40`).
- **Snapshots are idempotent**, and the agent's `lastAppliedVersion` rejects
  older pushes so replay cannot regress state
  (`docs/architecture/connections.md:541 @c307f40`).
- **Events fire once per dedupe key and version**
  (`docs/architecture/connections.md:542 @c307f40`).
- **One cursor for both slices** — `dispatched_at` is stamped in the same
  transaction that advances `last_applied_version`
  (`docs/architecture/connections.md:543 @c307f40`).
- **api-server is the only caller of `applyState`** — the harness port admits
  ingress only from api-server pods; the agent's only outbound channel is the
  paired gateway, which routes back to `hello`
  (`docs/architecture/connections.md:544 @c307f40`).
- **Every Contribution kind has exactly one rail**, with no overlap between
  drivers, controller-render, and Envoy
  (`docs/architecture/connections.md:545 @c307f40`).

## See also

- [Platform topology](platform-topology.md) — component layout and gateways.
- [Agent lifecycle](agent-lifecycle.md) — boot, hibernation, and wake (when
  `hello` fires).
- [Security and credentials](security-and-credentials.md) — the egress-inject
  gateway rail and per-Connection Secrets.
- [Channels](channels.md) — related delivery/messaging surfaces.
- [Persistence](persistence.md) — the Postgres tables and agent PVC substrates.
