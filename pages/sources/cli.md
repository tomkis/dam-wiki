---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files:
  - packages/cli/README.md
  - packages/cli/package.json
  - packages/cli/src/bin.ts
  - packages/cli/src/compose.ts
  - packages/cli/src/modules/cli/compose.ts
  - packages/cli/src/modules/cli/domain/config.ts
  - packages/cli/src/modules/cli/domain/compat.ts
  - packages/cli/src/modules/cli/commands/version.ts
  - packages/cli/src/modules/cli/services/compat-service.ts
  - packages/cli/src/modules/cli/infrastructure/version-probe.ts
  - packages/cli/src/modules/cli/infrastructure/config-path.ts
  - packages/cli/src/modules/cli/infrastructure/package-version.ts
  - packages/cli/src/modules/shared/preflight.ts
  - packages/cli/src/modules/shared/exit-codes.ts
  - packages/cli/src/modules/shared/trpc/trpc-client.ts
  - packages/cli/src/modules/auth/compose.ts
  - packages/cli/src/modules/auth/commands/login.ts
  - packages/cli/src/modules/auth/domain/flow.ts
  - packages/cli/src/modules/agent/compose.ts
  - packages/cli/src/modules/agent/services/agent-service.ts
  - packages/cli/src/modules/chat/compose.ts
  - packages/cli/src/modules/chat/commands/chat.ts
  - packages/cli/src/modules/chat/infrastructure/terminal-bridge.ts
  - packages/cli/src/modules/ssh/compose.ts
  - packages/cli/src/modules/ssh/commands/ssh.ts
  - docs/architecture/cli.md
updated: 2026-06-19
---

# cli (package)

The `dam` command-line client: a TypeScript Node package, published to npm as
`@dam-agents/cli`, that users install locally and point at a configured Platform
deployment. It never runs inside the cluster — it is the human/CI front door to
the [api-server](./api-server.md). See [DAM overview](./dam.md) and
[platform topology](../concepts/platform-topology.md) for where it sits.

- Package: `@dam-agents/cli`, version `0.2.5`, ESM, Apache-2.0
  (`packages/cli/package.json:1-6 @c307f40`).
- Single binary `dam` → `./dist/bin.js` (`packages/cli/package.json:19-21 @c307f40`).
- Requires Node ≥ 20 (`packages/cli/package.json:16-18 @c307f40`); install with
  `npm install -g @dam-agents/cli` (`packages/cli/README.md:5-9 @c307f40`).

## Entry point & composition

`bin.ts` builds the program via `compose()`, runs `program.parseAsync`, and
gives one special-case: a `TermsStaleAtTransportError` is caught and turned into
a "Terms of Use acceptance required" hint pointing at the host
(`packages/cli/src/bin.ts:1-16 @c307f40`).

`compose.ts` is the package-level wiring. Each bounded context exposes its own
`compose*Module()` that returns commander `Command`s (plus cross-module
services); the top-level `compose()` stitches them into one
`commander` program named `dam`
(`packages/cli/src/compose.ts:39-149 @c307f40`). Dependency injection is manual:
the `cli` module produces the shared `compatService` + `configService`, the
`auth` module produces the `tokenProvider`, and every other module receives them
as constructor args rather than importing internals
(`packages/cli/src/compose.ts:40-67 @c307f40`). The shared tRPC client is built
per-host via `buildTrpc = (host) => createTrpcClient({ host, tokenProvider })`
(`packages/cli/src/compose.ts:48 @c307f40`).

Each module follows a `commands/ · services/ · infrastructure/ · domain/`
layering (DDD-ish hexagonal split): domain is pure/I/O-free, infrastructure does
fetch/fs/spawn, services orchestrate, commands wire to commander.

## Command surface

Command groups are singular to match `gh`/`git`/`docker`
(`docs/architecture/cli.md` @c307f40). Top-level commands registered in
`compose.ts:134-146 @c307f40`:

- **cli module** — `config set`, `ping`, `version`
  (`packages/cli/src/modules/cli/compose.ts:41-48 @c307f40`).
- **auth** — `login`, `logout`, `status`, and the `token` sub-tree
  (`create`/`list`/`revoke`) for API keys
  (`packages/cli/src/modules/auth/compose.ts:133-177 @c307f40`).
- **agent** — `list` (default), `get`, `create`, `create-interactive`,
  `delete`, `restart`
  (`packages/cli/src/modules/agent/compose.ts:60-83 @c307f40`).
- **chat** + **session** — `dam chat <agent>` attaches your terminal to an
  agent TUI; `dam session list`
  (`packages/cli/src/modules/chat/compose.ts:51-60 @c307f40`).
- **ssh** — `connect`, `configure`, and a hidden internal `_proxy`
  (`packages/cli/src/modules/ssh/commands/ssh.ts:57-257 @c307f40`).
- Plus `template`, `import`, `file`, `egress` (the `network` group),
  `approval`, `connection`, `schedule`, `skill`
  (`packages/cli/src/compose.ts:137-145 @c307f40`).

Process exit codes are a flat, shared contract with the user's shell
(`packages/cli/src/modules/shared/exit-codes.ts:8-35 @c307f40`): `0` success,
`1` runtime failure, `2` invalid input, `3` below-floor, plus typed codes per
domain (e.g. `5` agent-not-resolved, `9` schedule-not-found, `130` SIGINT).

## How it talks to the api-server (tRPC + WS)

Two transports, both pointed at the [api-server](./api-server.md):

**tRPC over HTTP** — the bulk of verbs. `createTrpcClient` uses an
`httpBatchLink` against `<host>/api/trpc`, typed by the shared `AppRouter` from
the `api-server-api` contract package, so server type changes reach the CLI with
no codegen (`packages/cli/src/modules/shared/trpc/trpc-client.ts:1-88 @c307f40`).
Agent-scoped tRPC (e.g. SSH key authorization) goes through the relay at
`<host>/api/agents/<agentId>/trpc`, typed by the `agent-runtime-api`
`AppRouter` (`packages/cli/src/modules/shared/trpc/trpc-client.ts:90-106 @c307f40`).
The `agent-service` wraps these calls (`agents.list/get/delete/restart`) and
classifies tRPC errors into typed `Result`s
(`packages/cli/src/modules/agent/services/agent-service.ts:26-64 @c307f40`).

Two cross-cutting concerns ride on the fetch layer: every request attaches
`Authorization: Bearer <token>` from the `TokenProvider`
(`.../trpc-client.ts:25-50 @c307f40`), and a fetch wrapper turns an HTTP `412`
with body `{ error: "terms_stale" }` into a `TermsStaleAtTransportError`
(`.../trpc-client.ts:52-72 @c307f40`).

**WebSocket** — interactive sessions. `dam chat` and `dam ssh` open a WS to the
api-server's terminal relay rather than tRPC. `connectTerminalBridge` derives
`wss:`/`ws:` from the host scheme, appends the token as a `?token=` query param,
and frames stdin/stdout via the `OP_INPUT`/`OP_OUTPUT`/`OP_EXIT` opcodes from the
`api-server-api` package
(`packages/cli/src/modules/chat/infrastructure/terminal-bridge.ts:1-44 @c307f40`).

## `dam chat` / terminal attach

`dam chat <agent>` connects the local terminal to an agent's interactive TUI
(`packages/cli/src/modules/chat/commands/chat.ts:13-19 @c307f40`). Strategy
flags select the session: `--resume <id>`, `-c/--continue` (most recent
terminal session), or default `new`; `--reset` kills the existing PTY and starts
fresh (`packages/cli/src/modules/chat/commands/chat.ts:16-34 @c307f40`). Chat
requires a TTY; a non-tty fails with `not-a-tty`
(`packages/cli/src/modules/chat/commands/chat.ts:114-116 @c307f40`). On
disconnect it prints a copy-pasteable `dam chat <agent> --resume <id>` resume
hint (`packages/cli/src/modules/chat/commands/chat.ts:48-56 @c307f40`).
Sessions are agent-owned and spoken over ACP across the api-server relay, not
tRPC (`packages/cli/src/modules/chat/compose.ts:23-33 @c307f40`).

## `dam ssh`

`dam ssh connect <agent>` opens an SSH session, or launches an editor/IDE
against the agent — `--exec <bin[:mode]>` where mode is one of `ssh`, `code`,
`zed`, inferred from the binary name unless forced with a `:mode` suffix
(`packages/cli/src/modules/ssh/commands/ssh.ts:43-92 @c307f40`). `dam ssh
configure` writes/removes the dam-managed SSH host config without launching:
one agent, `--all` (reconciles — adds current agents, prunes deleted ones), or
`--clear` (`packages/cli/src/modules/ssh/commands/ssh.ts:126-216 @c307f40`).
The hidden `_proxy` subcommand acts as an `ssh` `ProxyCommand`: it resolves the
agent, ensures a local keypair, registers the public key with the agent via
`ssh.authorizeKey.mutate` over agent-scoped tRPC, then bridges stdin/stdout to
the agent over a raw WS
(`packages/cli/src/modules/ssh/commands/ssh.ts:218-256 @c307f40`). For `code`
mode it first ensures editor egress is allowed for the VS Code remote hosts
(`packages/cli/src/modules/ssh/commands/ssh.ts:109-115 @c307f40`).

See [agent](../entities/agent.md) for the resource these verbs address.

## Auth

`dam auth login` authenticates against the Active Host's Keycloak realm via the
OAuth 2.0 Device Authorization Grant (RFC 8628)
(`packages/cli/src/modules/auth/commands/login.ts:20-36 @c307f40`). Flags:
`--server` (persisted as active host), `--no-browser`, `--force`
(`packages/cli/src/modules/auth/commands/login.ts:24-36 @c307f40`). The polling
loop's rules are a pure decision function `nextFlowStep` —
`authorization_pending` → poll again, `slow_down` → +5s, `access_denied` /
`expired_token` → terminal — with all I/O (sleeps, HTTP) kept outside
(`packages/cli/src/modules/auth/domain/flow.ts:66-116 @c307f40`).

The `auth` module exposes a single application service, **`TokenProvider`**,
which every authenticated verb consumes via `getValidAccessToken(host)`
(`packages/cli/src/modules/auth/compose.ts:49-51,113-118 @c307f40`). Auth-config
and OIDC discovery are raw `fetch` (not tRPC), with a per-invocation token-
endpoint resolver cache used during refresh
(`packages/cli/src/modules/auth/compose.ts:65-93 @c307f40`). For headless/CI use,
set `DAM_TOKEN` (used verbatim, bypassing `auth.toml`); credentials otherwise
live keyed by host in `$XDG_STATE_HOME/dam/auth.toml`
(`docs/architecture/cli.md` @c307f40).

## Version compatibility-floor check

Before networked verbs run, the CLI probes the api-server's **unauthenticated**
`GET <server>/api/version` (plain HTTP, outside tRPC), parsing
`{ serverVersion, minClientVersion? }` with a 5s deadline
(`packages/cli/src/modules/cli/infrastructure/version-probe.ts:5-32 @c307f40`).
`CompatService.check()` combines the probe with the local CLI version to produce
a verdict (`packages/cli/src/modules/cli/services/compat-service.ts:30-47 @c307f40`).

`verdictFor` (pure, with an inlined semver comparator — no semver dep, to keep
the domain dependency-free) yields one of three verdicts
(`packages/cli/src/modules/cli/domain/compat.ts:1-48 @c307f40`):

- **below-floor** — local `<` `minClientVersion`. Gated verbs hard-fail with
  exit `3` (`packages/cli/src/modules/shared/preflight.ts:23-28 @c307f40`).
- **behind-current** — below server but at/above floor → stderr warning, proceed
  (`packages/cli/src/modules/shared/preflight.ts:29-33 @c307f40`).
- **ok** — at/ahead of server. If `minClientVersion` is absent, below-floor is
  never produced (`packages/cli/src/modules/cli/domain/compat.ts:32-38 @c307f40`).

`resolveActiveHost` runs this gate then resolves the host, and is the shared
preflight for gated verbs (`packages/cli/src/modules/shared/preflight.ts:11-39
@c307f40`). `dam version` is the **un-gated** counterpart to `dam ping`: it
surfaces the same verdict and warnings but always exits 0, even on probe failure
(`packages/cli/src/modules/cli/commands/version.ts:9-66 @c307f40`). The local
version is embedded at build time by tsup's `define` (`__CLI_VERSION__`),
falling back to `0.0.0-dev` when run unbundled
(`packages/cli/src/modules/cli/infrastructure/package-version.ts:1-12 @c307f40`).

## Config

Flat schema, currently one key: `server`, validated as an http(s) URL via Zod;
undeclared keys are a build error
(`packages/cli/src/modules/cli/domain/config.ts:9-27 @c307f40`). Resolution
precedence is flag → env (`DAM_SERVER`) → file → `missing-config` error, with no
silent default (`packages/cli/src/modules/cli/domain/config.ts:9,59-68
@c307f40`). The config file lives at `$XDG_CONFIG_HOME/dam/config.toml`
(default `~/.config/dam/config.toml`) per XDG
(`packages/cli/src/modules/cli/infrastructure/config-path.ts:13-19 @c307f40`);
credentials live separately under `$XDG_STATE_HOME/dam/`
(`packages/cli/README.md:20 @c307f40`).

## Key files

| Path | Role |
| --- | --- |
| `src/bin.ts` | Executable entry; runs the program, handles terms-stale |
| `src/compose.ts` | Package-level DI wiring of all module commands |
| `src/modules/cli/` | config, ping, version, compat-service, version-probe |
| `src/modules/shared/trpc/trpc-client.ts` | tRPC client (server + agent-scoped), auth headers, terms gate |
| `src/modules/auth/` | Device-flow login, TokenProvider, API keys |
| `src/modules/agent/` | Agent lifecycle verbs + agent-service |
| `src/modules/chat/` | `dam chat` terminal attach over WS/ACP |
| `src/modules/ssh/` | `dam ssh` connect/configure/_proxy |
| `src/modules/shared/exit-codes.ts` | Flat process exit-code contract |

## Related

- [api-server](./api-server.md) — the deployment the CLI targets (tRPC + relay).
- [dam (overview)](./dam.md) — the platform as a whole.
- [agent](../entities/agent.md) — the resource `agent`/`chat`/`ssh` address.
- [platform topology](../concepts/platform-topology.md) — where the CLI fits.
</content>
</invoke>
