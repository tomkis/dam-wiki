---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [packages/agent-runtime/src/server.ts, packages/agent-runtime/src/modules/config.ts, packages/agent-runtime/src/modules/acp/compose.ts, packages/agent-runtime/src/modules/acp/services/acp-runtime.ts, packages/agent-runtime/src/modules/acp/services/trigger-session-driver.ts, packages/agent-runtime/src/modules/runtime-channel/compose.ts, packages/agent-runtime/src/modules/runtime-channel/service.ts, packages/agent-runtime/src/modules/runtime-channel/hello.ts, packages/agent-runtime/src/modules/runtime-channel/dispatcher.ts, packages/agent-runtime/src/modules/runtime-channel/drivers/trigger-impl.ts, packages/agent-runtime/src/modules/runtime-channel/infrastructure/file-ops.ts, packages/agent-runtime/src/modules/runtime-channel/index.ts, packages/agent-runtime/src/modules/import/index.ts, packages/agent-runtime/src/modules/import/http.ts, packages/agent-runtime/package.json, packages/agent-runtime-api/src/index.ts, packages/agent-runtime-api/src/router.ts, packages/agent-runtime-api/src/context.ts, packages/agent-runtime-api/src/trpc.ts, packages/agent-runtime-api/src/modules/runtime/router.ts, packages/agent-runtime-api/src/modules/runtime/service.ts, packages/agent-runtime-api/src/modules/runtime/types.ts, packages/agent-runtime-api/src/modules/files/router.ts, packages/agent-runtime-api/package.json]
updated: 2026-06-19
---

# agent-runtime (package)

`agent-runtime` is the **in-pod server that runs inside every agent pod**. It is
the only DAM process the api-server talks to across the pod boundary: it fronts
the harness subprocess (Claude Code, Codex, etc.), terminal, and SSH on a single
HTTP/WebSocket port, and holds the agent side of the runtime channel that applies
desired state (env, files, MCP entries, skills) and fires triggers. See
[platform topology](../concepts/platform-topology.md) for where the pod sits in
the cluster and [agent lifecycle](../concepts/agent-lifecycle.md) for boot/wake
ordering; this page is a map of the package itself.

`agent-runtime-api` is its **shared contract package** — the tRPC router type,
context interface, and the Zod schemas / TypeScript types for the runtime
channel. The api-server imports it to type its client; the server imports it to
mount the router. Nothing in it executes pod logic; it is types + schemas only
(`packages/agent-runtime-api/package.json:4-12 @c307f40` exports `.` and
`./router`).

## What it does

A single Node HTTP server (`packages/agent-runtime/src/server.ts:316-359 @c307f40`)
multiplexes everything the api-server proxies into the pod:

- **`/api/acp`** — WebSocket upgrade to the ACP relay; each socket becomes a
  client channel attached to the harness
  (`server.ts:365-372 @c307f40`).
- **`/api/terminal`** — WebSocket upgrade that spawns/attaches a
  `harness-terminal` PTY per `sessionId`
  (`server.ts:375-380 @c307f40`).
- **`/api/ssh`** — WebSocket upgrade that bridges to an in-pod `sshd`
  (`server.ts:381-390 @c307f40`).
- **`/api/trpc/*`** — the scoped tRPC router (files / skills / ssh / runtime),
  path-stripped and handed to the standalone tRPC handler
  (`server.ts:351-356 @c307f40`).
- **`/api/import`** — multipart bundle upload, extracted into the work dir
  (`server.ts:337-340 @c307f40`).
- **`/api/status`**, **`/healthz`**, and `POST /api/sessions/:id/reset`
  (`server.ts:322-349 @c307f40`).

No in-process auth check guards these routes. The pod's NetworkPolicy admits
ingress on this port only from the api-server (which has already verified the
user JWT and agent ownership), so the kernel is the auth boundary
(`server.ts:152-166 @c307f40`; `packages/agent-runtime-api/src/trpc.ts:34-41 @c307f40`).
Config is a tiny Zod-parsed env block — `PORT` (default 8080), `HOME_DIR`,
`WORK_DIR`, `API_SERVER_URL`, `PLATFORM_DEV`
(`packages/agent-runtime/src/modules/config.ts:3-14 @c307f40`).

## Directory / module layout

`packages/agent-runtime/src/`:

- **`server.ts`** — composition root + the HTTP/WS multiplexer and PTY manager.
- **`core/`** — cross-module primitives: `document-store.ts` (file-backed state
  backend), `runtime-env.ts` (`RuntimeEnvReader` port + `mergedSpawnEnv`),
  `result.ts`, `expand-home.ts`, `import-staging.ts`.
- **`modules/acp/`** — the ACP relay (see below).
- **`modules/runtime-channel/`** — the runtime channel: `applyState`, hello,
  contribution drivers, trigger/seed event loop.
- **`modules/import/`** — bundled-file import (multipart → extract → finalize).
- **`modules/skills/`** — skill install/publish/scan service (backs `skills.*`).
- **`modules/files.ts`**, **`modules/ssh.ts`**, **`modules/git.ts`**,
  **`modules/pod-service.ts`** — services for the files router, sshd, git
  credential helper, and an optional `pod-service` supervisor.

`packages/agent-runtime-api/src/` mirrors the router shape: `router.ts`,
`context.ts`, `trpc.ts`, `result.ts`, and a `modules/<files|skills|ssh|runtime|import|plugin>/`
folder of types + schemas per router segment.

## The ACP relay (`modules/acp/`)

The relay speaks **JSON-RPC 2.0 / ACP** between client channels (WebSocket
sockets, or in-memory channels for triggers) and a single harness subprocess. It
is composed at boot by `composeAcp`, which wires the spawn function, the
session-metadata store, and an env-ready gate
(`packages/agent-runtime/src/modules/acp/compose.ts:22-43 @c307f40`). The harness
command is `/usr/local/bin/harness-chat` in production
(`server.ts:101-104 @c307f40`).

`createAcpRuntime` (`packages/agent-runtime/src/modules/acp/services/acp-runtime.ts:205 @c307f40`)
is the heart of the package. Its public surface is small —
`attach`, `status`, `resetSession`, `refreshEnv`, `shutdown`
(`acp-runtime.ts:50-72 @c307f40`) — but it implements a fan-out relay with
several non-obvious behaviours:

- **Engagement, not just attach.** Multiple channels may attach the same
  harness; a channel only receives a session's updates and agent-initiated
  requests once it has *engaged* that session by sending/receiving a
  session-scoped ACP frame (`acp-runtime.ts:50-66 @c307f40`). This is what lets
  multiple browser tabs watch the same agent while a cross-session call like
  `session/list` stays private.
- **Per-session append-only log + cursors.** `session/update` notifications are
  logged and fanned out by per-channel cursor, so a reconnecting client catches
  up from where it left off; a soft byte cap evicts oldest entries and emits a
  `platform_clipped_replay` sentinel
  (`acp-runtime.ts:289-353 @c307f40`). Agent→client *requests* (permission
  prompts) are deliberately never logged — only live-fanned and replayed from
  `pendingFromAgent` to fresh engagers (`acp-runtime.ts:824-842 @c307f40`).
- **Runtime-mediated `session/resume` and `session/load`.** Resume never reaches
  the harness; the runtime serves it from the in-memory log on the hot path, or
  on a cold miss issues its own `session/load` to rehydrate the subprocess and
  parks waiters (`acp-runtime.ts:1055-1137 @c307f40`). This shields harnesses
  that don't implement resume and survives subprocess respawns.
- **Prompt queue + turn boundaries.** Prompts for a busy session queue (cap 32,
  `acp-runtime.ts:19-20 @c307f40`); on response the runtime emits a synthetic
  "turn ended" notification (ACP has no on-wire turn boundary) so non-originating
  viewers can close their bubble (`acp-runtime.ts:956-977 @c307f40`).
- **Env recycle.** `refreshEnv` recycles the harness so it respawns with new env —
  immediately if idle, otherwise at the next turn boundary or after a forced
  bound (`acp-runtime.ts:1281-1297 @c307f40`).
- **Idle-session reaping.** When no channel is engaged and nothing is in flight,
  the session is torn down (`session/close`) to free the per-session CLI
  subprocess, gated on the harness advertising `sessionCapabilities.close`
  (`acp-runtime.ts:700-716 @c307f40`).

**Trigger sessions** run through the same relay via an in-memory channel.
`createTriggerSessionDriver` opens a channel, drives `session/new`/`prompt` (or
`session/resume` for continuous schedules), and resolves a `sessionId`
(`packages/agent-runtime/src/modules/acp/services/trigger-session-driver.ts:5-28 @c307f40`).
The runtime-channel's `trigger-impl` decides fresh vs continuous and stamps
`ScheduleCron` session metadata
(`packages/agent-runtime/src/modules/runtime-channel/drivers/trigger-impl.ts:18-40 @c307f40`).

## The runtime channel (`modules/runtime-channel/`)

This is the agent side of the desired-state protocol the api-server pushes. See
[channels](../concepts/channels.md) for the full protocol; the package's role:

- **hello on boot.** After listen, the server calls `runtimeChannel.helloOnBoot`,
  which advertises the agent's capabilities — every contribution kind from the
  manifest and every event kind from the schema — and retries with backoff until
  it lands, since readiness hard-depends on it
  (`packages/agent-runtime/src/server.ts:422-426 @c307f40`;
  `packages/agent-runtime/src/modules/runtime-channel/compose.ts:83-120 @c307f40`;
  `packages/agent-runtime/src/modules/runtime-channel/hello.ts:6-30 @c307f40`).
- **applyState.** The single tRPC mutation the channel exposes
  (`packages/agent-runtime-api/src/modules/runtime/router.ts:4-8 @c307f40`,
  `service interface at packages/agent-runtime-api/src/modules/runtime/service.ts:3-5 @c307f40`).
  The server-side implementation serialises applies, dispatches contributions to
  drivers when the state hash changed, and runs events; on driver failure it
  leaves the cursor behind so the next push re-dispatches
  (`packages/agent-runtime/src/modules/runtime-channel/service.ts:21-120 @c307f40`).
- **Contribution drivers (plugins).** A manifest binds each contribution kind to
  a plugin impl; the dispatcher routes by kind
  (`packages/agent-runtime/src/modules/runtime-channel/dispatcher.ts:43-50 @c307f40`).
  The server registers env, file, mcp-entry, and skill-install plugins
  (`server.ts:119-133 @c307f40`); the env plugin's `onChange` re-triggers the
  ACP env recycle and git credential setup (`server.ts:120-129 @c307f40`). The
  file plugin merges contributed files into `$HOME` with formats
  yaml/json/text/ini and several merge modes
  (`packages/agent-runtime/src/modules/runtime-channel/infrastructure/file-ops.ts:13-30 @c307f40`).
- **Event handlers.** `trigger`, `schedule-reset`, and `workspace-seed` events
  carry their own version and apply even when contributions are stale
  (`packages/agent-runtime-api/src/modules/runtime/types.ts:13-18 @c307f40`;
  `service.ts:51-67 @c307f40`).

The contribution and event kinds are single-sourced from the contract package's
Zod enums: `env`, `egress-allow`, `egress-inject`, `file`, `mcp-entry`,
`skill-ref` for contributions
(`packages/agent-runtime-api/src/modules/runtime/types.ts:3-10 @c307f40`).

## File ops (tRPC) and bundle import

The tRPC `appRouter` mounts four segments — `files`, `skills`, `ssh`, `runtime`
(`packages/agent-runtime-api/src/router.ts:7-12 @c307f40`) — over a context the
server builds per request from the boot-time services
(`server.ts:157-166 @c307f40`;
`packages/agent-runtime-api/src/context.ts:6-11 @c307f40`). The **files** router
is a scoped editor over the agent's home: `listDirs`, `read`, `write`, `create`,
`mkdir`, `rename`, `remove`, `upload`, each returning a `Result` mapped to tRPC
error codes (Forbidden/NotFound/Conflict/PayloadTooLarge)
(`packages/agent-runtime-api/src/modules/files/router.ts:40-118 @c307f40`). The
backing `FilesService` is rooted at `homeDir` and caps decoded uploads
(`server.ts:73 @c307f40`; transport guard `TRPC_MAX_BODY_SIZE = 32MB` at
`server.ts:146-150 @c307f40`).

**Bundle import** is a separate raw HTTP path (not tRPC) because it streams a
multi-GB tar. `POST /api/import` runs through busboy → `extractBundle` →
`finalize`, single-flight (a concurrent import gets 409), with its own 30s
inactivity and 30min wall-clock deadlines; the server disables its global
`requestTimeout` for this route and relies on those plus `headersTimeout`
(`packages/agent-runtime/src/modules/import/http.ts:12-58 @c307f40`;
`server.ts:396-413 @c307f40`). A boot-time `sweepStaging` cleans abandoned
staging dirs (`server.ts:418-420 @c307f40`).

## Idle status

`/api/status` reports the pod idle only when the ACP runtime has no active or
queued prompt and no pending agent request **and** there are no live terminal
PTYs (`server.ts:327-335 @c307f40`; `acp-runtime.ts:1265-1274 @c307f40`). The
controller uses this signal to decide when an agent pod can be scaled down — see
[agent lifecycle](../concepts/agent-lifecycle.md).

## Public surface of `agent-runtime-api`

The contract package re-exports from one barrel
(`packages/agent-runtime-api/src/index.ts:1-100 @c307f40`):

- `AppRouter` (the router *type*) and `AgentRuntimeContext`.
- `Result` / `ok` / `err`.
- Per-segment input schemas + domain types for files, skills, ssh, import.
- The runtime-channel vocabulary: `contribution`/`event` schemas, `Capabilities`,
  `ApplyStateInput`/`ApplyStateResult`, `HelloInput`/`HelloResult`,
  `RuntimeChannelService`, and the plugin contract
  (`PLUGIN_PROTOCOL_VERSION`, `Plugin`, `KindHandler`, `DispatchContext`, …).

Because it ships source TS (`main: ./src/index.ts`) it is a workspace-only,
build-free contract: a router-type change in the pod and a client change in the
api-server stay in lockstep through a single `workspace:*` dependency.

## Related pages

- [DAM (repo overview)](./dam.md)
- [platform topology](../concepts/platform-topology.md)
- [agent lifecycle](../concepts/agent-lifecycle.md)
- [channels](../concepts/channels.md)
- [agent (entity)](../entities/agent.md)
