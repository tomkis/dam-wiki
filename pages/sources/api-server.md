---
source: dam-agents/dam
commit: 9d1bc9990fba55f43d60b0aad453b188af5896a8
files:
  - packages/api-server/src/index.ts
  - packages/api-server/src/apps/api-server/app.ts
  - packages/api-server/src/apps/api-server/acp-relay.ts
  - packages/api-server/src/apps/api-server/auth.ts
  - packages/api-server/src/apps/harness-api-server/app.ts
  - packages/api-server/src/apps/harness-api-server/harness-router.ts
  - packages/api-server/src/apps/harness-api-server/mcp-endpoint.ts
  - packages/api-server/src/apps/harness-api-server/runtime-trpc.ts
  - packages/api-server/src/apps/harness-api-server/agent-auth.ts
  - packages/api-server/src/apps/ext-authz/grpc.ts
  - packages/api-server/src/modules/schedules/compose.ts
  - packages/api-server-api/src/index.ts
  - packages/api-server-api/src/router.ts
  - packages/api-server-api/src/context.ts
  - packages/api-server-api/src/auth-procedures.ts
  - packages/api-server-api/src/harness-router.ts
  - packages/api-server-api/src/harness-context.ts
  - packages/api-server-api/src/modules/connections/providers.ts
  - docs/architecture/platform-topology.md
updated: 2026-06-22
---

# api-server (package)

The TypeScript server that hosts DAM's user-facing surface and brokers all
traffic to agent pods. Clients never dial pods directly: the api-server proxies
tRPC, relays ACP/terminal/SSH WebSockets, exposes the harness-only MCP and
runtime endpoints, runs the schedule loop, and answers Envoy's ext_authz Check
for human-in-the-loop egress gating. See [platform-topology](../concepts/platform-topology.md)
for where it sits among the controller, agent-runtime, and gateway.

It ships as two workspace packages:

- **`api-server`** — the runnable server (`packages/api-server/`). Hono on
  `@hono/node-server`, tRPC, K8s client, BullMQ, Redis, Postgres.
- **`api-server-api`** — the shared contract/types package
  (`packages/api-server-api/`): tRPC router type, Zod input schemas, and the
  service interfaces + DTOs that both the server and the UI/CLI import. It is the
  client-facing boundary, so it deliberately keeps `@trpc/server`-loading code
  (the procedure builders) out of its barrel — see below.

## The two-port model

`src/index.ts` is the composition root: it loads config, runs DB migrations,
wires every module, and starts three listeners
(`packages/api-server/src/index.ts:497-549 @c307f40`).

- **Public port** (`config.port`) — user-authenticated tRPC + REST + the
  ACP/terminal/SSH relay WebSockets. Started by `startApiServerApp`
  (`packages/api-server/src/apps/api-server/app.ts:824 @c307f40`).
- **Harness port** (`config.harnessServerPort`) — internal-only, consumed by
  agent pods for MCP tool calls and the runtime channel; no user auth. Started by
  `startHarnessApiServerApp`
  (`packages/api-server/src/apps/harness-api-server/app.ts:64-71 @c307f40`).
- **ext-authz gRPC port** (`config.extAuthzPort`) — Envoy ext_authz Check for
  egress gating, started by `startExtAuthzGrpcApp`
  (`packages/api-server/src/apps/ext-authz/grpc.ts:46 @c307f40`,
  bound at `:140-156`).

The split matters for security: the public port authenticates a human or API key
and checks ownership, while the harness port trusts the mesh — an agent pod's
identity is proven cryptographically by the Istio waypoint before the request
arrives, so the handler only does a label lookup, never parses a header
(`packages/api-server/src/apps/harness-api-server/agent-auth.ts:14-37 @c307f40`).

## Directory layout

`packages/api-server/src/`:

- **`index.ts`** — composition root / DI wiring + graceful shutdown
  (`packages/api-server/src/index.ts:512-547 @c307f40`).
- **`config.ts`, `events.ts`** — env config loader and the domain event bus enum.
- **`apps/`** — the three runnable surfaces (see below).
- **`core/`** — cross-cutting infra: `logger.ts` (pino), `redis-bus.ts`
  (cross-replica domain bus), `security-log.ts` (structured audit lines),
  `acp-client.ts`, `sub-pseudonymizer.ts`, `unit-of-work.ts`, `result.ts`,
  `db-errors.ts`, `format-error.ts`.
- **`bootstrap/trusted-hosts.ts`** — loads the operator-editable seed list for
  the `trusted` egress preset at boot.
- **`sagas/agent-artifacts-sweeper.ts`** — cross-store orphan reaper that finds
  DB rows for agents no longer in the live K8s set and runs each module's cleanup
  (`packages/api-server/src/index.ts:444-475 @c307f40`).
- **`modules/`** — the bounded contexts (see below).
- **`proto/`, `proto-gen/`** — `external_auth.proto` and its ts-proto output for
  the ext_authz gRPC service.

### apps/

Each app composes a Hono router (or a gRPC server) from the modules:

- **`apps/api-server/`** — the public port. `app.ts` (982 lines) mounts the
  public REST routes, the auth + terms middleware, the OAuth callback routers,
  the tRPC handler, the per-agent tRPC proxy, the file-import proxy, and the WS
  relays. Siblings: `auth.ts` (JWT + API-key verification), `acp-relay.ts`,
  `terminal-relay.ts`, `ssh-relay.ts`, `session-presence.ts`, `oauth.ts`,
  `terms-gate.ts`, `brand-icon.ts`.
- **`apps/harness-api-server/`** — the harness port. `app.ts` builds a
  `createHarnessRouter` that mounts the MCP endpoint and the runtime tRPC
  (`packages/api-server/src/apps/harness-api-server/harness-router.ts:21-31 @c307f40`).
- **`apps/ext-authz/grpc.ts`** — the gRPC ext_authz server.

### modules/

One directory per bounded context, each typically `compose.ts` + `domain/` +
`services/` + `infrastructure/`. The set:
`agents`, `api-keys`, `approvals`, `audit`, `channels`, `connections`, `e2e`,
`egress-rules`, `files`, `forks`, `repos`, `runtime-delivery`, `schedules`,
`secret-store`, `secrets`, `skills`, `templates`, `terms`, `usage`. Most expose a
service that backs a matching tRPC router in `api-server-api`; the composition
root injects the K8s/DB/Redis ports. See [bounded-contexts](../concepts/bounded-contexts.md).

## Public surface (tRPC)

The public tRPC handler builds a per-request `ApiContext` — one service instance
per module, scoped to the authenticated `user.sub` — and dispatches through
`appRouter` (`packages/api-server/src/apps/api-server/app.ts:679-795 @c307f40`).
`appRouter` is assembled in the contract package from fourteen sub-routers:
templates, repos, agents, schedules, secrets, channels, connections, skills,
approvals, egressRules, files, terms, e2e, apiKeys
(`packages/api-server-api/src/router.ts:16-31 @c307f40`).

A handful of REST routes sit outside tRPC: `/api/health`, `/api/version`
(unauthenticated — powers the CLI compatibility check), `/api/auth/config`,
`/api/brand`, `/api/terms`, and the brand-icon/manifest routes, all registered
before the `auth` and `terms` middleware
(`packages/api-server/src/apps/api-server/app.ts:238-302 @c307f40`).

### Auth and ownership

The public port accepts either a Keycloak JWT or an API key in the same
`Authorization: Bearer` slot, dispatched by token prefix; `createAuth` wires both
verifiers (`packages/api-server/src/apps/api-server/app.ts:214-228 @c307f40`).
The resulting `UserIdentity` carries `sub`, effective `scopes`, an `agentIds`
allowlist (`"*"` = all owned agents), and an optional `keyId`
(`packages/api-server-api/src/context.ts:16-30 @c307f40`). tRPC procedures gate
on scope via the builders in `auth-procedures.ts` — `readAgentProcedure`,
`operateAgentsProcedure`, `manageAgentsProcedure`, etc.
(`packages/api-server-api/src/auth-procedures.ts:43-50 @c307f40`); the
non-tRPC surfaces (relay, import, WS) re-implement the same scope + binding
checks inline. See [security-and-credentials](../concepts/security-and-credentials.md).

## The ACP relay / WS proxy

The public Node server attaches a single `upgrade` handler that matches
`/api/agents/:id/(acp|terminal|ssh)`, verifies the query-string token, checks
ownership + `agents:operate` + per-key agent binding + terms acceptance, then
dispatches to the matching relay
(`packages/api-server/src/apps/api-server/app.ts:854-981 @c307f40`). The token
rides in the query string and is never logged.

`createAcpRelay` is a near-dumb WebSocket proxy: per upgrade it resolves the
instance identity once, ensures the agent is awake, dials the pod's upstream
`ws://<pod>/api/acp`, and pipes frames both ways
(`packages/api-server/src/apps/api-server/acp-relay.ts:104-200 @c307f40`). On top
of the pipe it mirrors HITL state: an outgoing `session/request_permission`
becomes a `pending_approvals` row, and the in-session response resolves it via a
deterministic row id, so no in-memory tracking map is needed
(`packages/api-server/src/apps/api-server/acp-relay.ts:140-175 @c307f40`). It also
subscribes a per-agent inject channel so the approvals service can push synthetic
ext_authz frames to the attached UI client
(`packages/api-server/src/apps/api-server/acp-relay.ts:135-139 @c307f40`). See
[channels](../concepts/channels.md) for the chat side and
[connections](../concepts/connections.md) for the runtime channel.

The plain tRPC proxy (`/api/agents/:id/trpc/*`) and the streamed file-import
proxy (`/api/agents/:id/import`) follow the same ownership/scope gate and forward
to the pod; import uses raw `node:http` (not undici) to preserve backpressure and
keep memory flat on multi-GB uploads
(`packages/api-server/src/apps/api-server/app.ts:376-417 @c307f40`,
`:454-718 @c307f40`).

## Harness port: MCP + runtime channel

The harness router mounts two surfaces
(`packages/api-server/src/apps/harness-api-server/harness-router.ts:21-31 @c307f40`):

- **MCP endpoint** — `POST/GET /api/agents/:id/mcp`, a Streamable-HTTP MCP server
  (`@modelcontextprotocol/sdk`) that exposes platform tools (channels, skills,
  schedules, files) to the running agent
  (`packages/api-server/src/apps/harness-api-server/mcp-endpoint.ts:500-501 @c307f40`;
  tools registered via `server.tool(...)` throughout the file). Sessions are keyed
  by `mcp-session-id` and pinned to one agent to detect hijack
  (`mcp-endpoint.ts:521-530 @c307f40`).
- **Runtime tRPC** — `/api/agents/:id/trpc/*` serving the `harnessRouter`
  (`runtime.*`) for the runtime-channel `hello` and `applyState` delivery
  (`packages/api-server/src/apps/harness-api-server/runtime-trpc.ts:13-41 @c307f40`,
  `packages/api-server-api/src/harness-router.ts:4-8 @c307f40`). Its context is
  the minimal `HarnessContext` — just `agentId` + the runtime-delivery service
  (`packages/api-server-api/src/harness-context.ts:3-6 @c307f40`).

Both resolve the caller with `resolveAgent`, which trusts the mesh-enforced URL
`:id` and only looks up the owner label
(`packages/api-server/src/apps/harness-api-server/agent-auth.ts:25-37 @c307f40`).

## Schedule loop

`composeSchedulesAtBoot` builds a BullMQ queue + worker and a `SchedulerRunner`;
on boot `restoreAll()` rehydrates every persisted schedule, and the fire handler
wakes a hibernated agent before triggering it
(`packages/api-server/src/modules/schedules/compose.ts:42-58 @c307f40`,
`packages/api-server/src/index.ts:477-490 @c307f40`). Per-owner schedule CRUD is
exposed through `composeSchedulesForOwner` on both ports. Cron and RRule logic
live in the contract package (`api-server-api/src/modules/schedules/rrule.ts`).

## ext_authz HITL handler

A single gRPC server backs both Envoy ext_authz filters (L7 HTTP and L4 network)
and delegates every Check to one `ExtAuthzGate`
(`packages/api-server/src/apps/ext-authz/grpc.ts:31-78 @c307f40`). It derives the
calling instance from the gRPC `:authority` of the per-instance ext-authz Service
the gateway dialled — identity is pinned by the Istio AuthorizationPolicy on that
Service, not by any header
(`packages/api-server/src/apps/ext-authz/grpc.ts:64-87 @c307f40`,
parsed by `parseInstanceFromAuthority` at `:163-176`). It strips the port from
host/SNI, calls `gate.gateRequest({agentId, host, method, path})`, and returns OK
or PERMISSION_DENIED; it **fails closed** on any error
(`packages/api-server/src/apps/ext-authz/grpc.ts:110-135 @c307f40`). The gate can
hold the request open while a human approves — keepalive is tuned to exceed the
hold deadline (`:50-54`). The decider is shared with the in-session ACP-native
approvals path, so egress denials and tool approvals converge on one approvals
module. See [security-and-credentials](../concepts/security-and-credentials.md).

## Postgres + K8s + Redis access

The composition root runs migrations then opens the DB
(`packages/api-server/src/index.ts:115-119 @c307f40`); modules receive the `db`
handle (postgres-js via the `db` workspace package) and write their own tables.
The chat-sdk channel state uses a separate `pg` pool with a scoped CA appended to
the URL (`packages/api-server/src/index.ts:286-296 @c307f40`). K8s access goes
through `createApi` / `createK8sClient` for Agent custom resources, ConfigMaps,
and Secrets; Redis backs both the BullMQ connection and the cross-replica
`redisBus` (`packages/api-server/src/index.ts:113-133 @c307f40`).

## api-server-api (the contract package)

`packages/api-server-api/src/` mirrors `modules/<name>/{router,schemas,types}.ts`
and re-exports them all from `index.ts` for browser/CLI consumers
(`packages/api-server-api/src/index.ts @c307f40`). Key exports: the `AppRouter`
and `HarnessRouter` type aliases, `ApiContext` / `HarnessContext`, `UserIdentity`,
every module's Zod input schema and service interface, the terminal binary frame
codec (`modules/terminal/protocol.ts`), the ACP synthetic-notification schemas
(`modules/acp/types.ts`), and the generated CRD types (`crd-types.gen.ts`).

One deliberate seam: `auth-procedures.ts` is **not** re-exported from the barrel,
because it calls `initTRPC.create()` at module load and would drag `@trpc/server`
into browser bundles; routers import it directly instead
(`packages/api-server-api/src/index.ts @9d1bc99`, see the comment block above the
`AgentBinding` re-export).

**Provider preset catalog** — `modules/connections/providers.ts` is the single
source of truth for model-provider definitions; the server, UI, and CLI all
derive from it (`packages/api-server-api/src/modules/connections/providers.ts:1
@9d1bc99`). It exports `PROVIDERS` — a keyed record of `ProviderPreset` objects
for `anthropic`, `ibm-litellm`, `openai`, and `bob` — each with a display name,
host/path pattern, and one or more `ProviderPresetMode` entries (key, label,
`templateId`, optional `tokenPrefix`, `defaultEnvMappings`, and
`injection`/`extraInjections` for Envoy credential injection). The
`PROVIDER_TEMPLATE_IDS` set (all mode `templateId` values) and
`providerTypeForTemplateId` / `templateIdForProvider` helpers let consumers
resolve the provider↔template relation without hand-maintaining a map
(`packages/api-server-api/src/modules/connections/providers.ts:195-233
@9d1bc99`). This catalog replaced per-package `provider-templates.ts` copies
that formerly lived in both the CLI and UI.

## Where to look

- Add/inspect a public API → `packages/api-server-api/src/modules/<name>/router.ts`
  + the matching `modules/<name>/` in the server.
- Trace a chat/permission frame → `apps/api-server/acp-relay.ts`.
- Trace an agent's tool call → `apps/harness-api-server/mcp-endpoint.ts`.
- Trace an egress allow/deny → `apps/ext-authz/grpc.ts` + `modules/approvals/`,
  `modules/egress-rules/`.
- Understand boot/shutdown and wiring → `src/index.ts`.

## Related

- [dam (overview)](./dam.md)
- [platform-topology](../concepts/platform-topology.md)
- [channels](../concepts/channels.md) · [connections](../concepts/connections.md)
- [security-and-credentials](../concepts/security-and-credentials.md)
- [agent (entity)](../entities/agent.md)
