---
source: dam-agents/dam
commit: 9d1bc9990fba55f43d60b0aad453b188af5896a8
files: [packages/ui/package.json, packages/ui/vite.config.ts, packages/ui/index.html, packages/ui/tsconfig.json, packages/ui/src/main.tsx, packages/ui/src/app.tsx, packages/ui/src/api.ts, packages/ui/src/trpc.ts, packages/ui/src/auth.ts, packages/ui/src/store.ts, packages/ui/src/modules/platform/lib/routes.ts, packages/ui/src/modules/acp/acp.ts, packages/ui/src/modules/acp/session-projection.ts, packages/ui/src/modules/acp/utils.ts, packages/ui/src/modules/sessions/hooks/use-acp-session.ts, packages/ui/src/modules/sessions/hooks/use-acp-connection.ts, packages/ui/src/modules/sessions/components/terminal.tsx]
updated: 2026-06-22
---

# ui (package)

`packages/ui` (npm name `platform-ui`) is the React + Vite single-page app that
DAM serves to humans. It is the operator console for the platform: a place to
create and configure agents, run chat sessions against them, approve permission
prompts, attach to terminals, and manage platform resources (connections,
providers, secrets, API keys, schedules, repos, files, usage). It is statically
built and served by the [api-server](./api-server.md); all of its dynamic
behaviour flows back to that server over HTTP and WebSocket. See
[platform topology](../concepts/platform-topology.md) for where the UI sits.

> **Naming:** "Sandbox" is the UI-facing label for an [Agent](../entities/agent.md).
> The creation flow lives under `src/modules/sandboxes/` and routes like
> `/sandboxes/new` and `/sandboxes/:id`, but the underlying identifier is always
> an `agentId` — the sandbox views and routes operate directly on it
> (`packages/ui/src/modules/platform/lib/routes.ts:36-67 @c307f40`).

## Stack

- **React 19** + **Vite 6**, built with `@vitejs/plugin-react`, Tailwind v4
  (`@tailwindcss/vite`), and a PWA service worker (`vite-plugin-pwa`)
  (`packages/ui/package.json:1-65 @c307f40`,
  `packages/ui/vite.config.ts:8-37 @c307f40`).
- **tRPC v11** client + **TanStack Query** for server state
  (`@trpc/client`, `@trpc/tanstack-react-query`,
  `packages/ui/package.json:21-24 @c307f40`).
- **Zustand** for client/UI state (`packages/ui/package.json:51 @c307f40`).
- **`@agentclientprotocol/sdk` (ACP)** for the chat protocol; **`@xterm/xterm`**
  for terminals (`packages/ui/package.json:6,16-17 @c307f40`).
- **`oidc-client-ts`** for Keycloak OIDC login
  (`packages/ui/package.json:36 @c307f40`).
- Shared types come from sibling workspace packages `api-server-api` and
  `agent-runtime-api` — the UI imports the api-server's `AppRouter` type and ACP
  helpers directly, so the contract is compile-time checked
  (`packages/ui/package.json:33-34 @c307f40`,
  `packages/ui/tsconfig.json:1-14 @c307f40`).

## Bootstrapping

`src/main.tsx` is the entry point. It runs OIDC auth init and an unauthenticated
brand fetch in parallel, gates on accepted terms, then dynamically imports and
renders `App` inside `QueryClientProvider` + `TooltipProvider`
(`packages/ui/src/main.tsx:18-41 @c307f40`). `index.html` ships no title or icon
assets — both `<title>` and the PWA manifest/icons are filled at runtime from
`/api/brand*` so a single build can be re-skinned per deployment
(`packages/ui/index.html:10-16 @c307f40`,
`packages/ui/vite.config.ts:13-17 @c307f40`).

## Routing & state

The app is a manual client router, not a router library. `App`/`MainApp` in
`src/app.tsx` switch on a `view` field in the Zustand store and keep it in sync
with the URL via `history` + `popstate` listeners
(`packages/ui/src/app.tsx:42-108 @c307f40`). The chat view is full-screen; every
other view shares an `IconRail` shell (`packages/ui/src/app.tsx:110-126 @c307f40`).
URL parsing/serialisation is centralised in
`src/modules/platform/lib/routes.ts` (`viewSchema`, `viewToPath`, `pathToState`),
covering `/`, `/chat/:agent`, `/settings[/tab]`, `/inbox`, `/terms`,
`/sandboxes/new`, `/sandboxes/:id`
(`packages/ui/src/modules/platform/lib/routes.ts:3-68 @c307f40`).

The Zustand store (`src/store.ts`) is composed from per-domain slices —
dialog, theme, navigation, agents, sessions, session-config, files, permissions —
merged into one `PlatformStore` (`packages/ui/src/store.ts:39-57 @c307f40`).

## Directory layout (`src/`)

| Path | Role |
|------|------|
| `main.tsx`, `app.tsx` | entry point and root view switcher |
| `api.ts`, `trpc.ts`, `query-client.ts` | tRPC client, typed proxy, query client |
| `auth.ts` | OIDC/Keycloak login, access-token accessor |
| `brand.ts` | runtime branding (title, theme, icons) |
| `store.ts` | composed Zustand store |
| `types.ts`, `lib/` | shared types and helpers (toast, utils, api-health, query-helpers) |
| `components/` | app-wide shared components (`IconRail`, banners, editors, modal) |
| `components/ui/` | shadcn/Radix primitives (`components.json` config) |
| `hooks/` | cross-cutting hooks (auto-resize, unsaved-guard) |
| `modules/<domain>/` | feature modules, one per bounded context |

Each `modules/<domain>` folder is self-contained, typically with `api/`
(tRPC query/mutation wrappers), `components/`, `views/`, `hooks/`, and sometimes
`store/`, `forms/`, `lib/`. The domains present are: `acp`, `agents`, `api-keys`,
`approvals`, `connections`, `egress-rules`, `files`, `platform`, `providers`,
`repos`, `sandboxes`, `schedules`, `secrets`, `sessions`, `settings`,
`templates`, `terms`, `usage`. These mirror the platform's
[bounded contexts](../concepts/bounded-contexts.md).

**Provider preset filtering.** The `connections` and `sandboxes` modules import
`PROVIDER_TEMPLATE_IDS` directly from `api-server-api` to exclude provider
connections (Anthropic, IBM LiteLLM, OpenAI, Bob Shell) from the generic
connection catalog row — those connections are managed separately in the
`providers` module and must not appear twice
(`packages/ui/src/modules/connections/components/templates-section.tsx:4
@9d1bc99`,
`packages/ui/src/modules/sandboxes/components/connections-section.tsx:4
@9d1bc99`). The local `provider-templates.ts` that previously duplicated this
catalog in the UI was deleted; the canonical definition now lives in
`api-server-api/src/modules/connections/providers.ts`. See
[api-server](./api-server.md) for the provider preset catalog.

## Talking to the api-server

The UI uses three distinct transports to the api-server, all under `/api`.

### 1. tRPC over HTTP — resource & permission management

The tRPC client (`src/api.ts`) posts batched calls to `/api/trpc` with an
`Authorization: Bearer <token>` header, typed against the api-server's
`AppRouter` (`packages/ui/src/api.ts:1-36 @c307f40`). Its custom `fetch`
double-duties as a health probe: `502/503` flips a connection-lost banner via
`onFetchError`, and a `412` with `error: "terms_stale"` triggers the
re-acceptance gate (`packages/ui/src/api.ts:12-30 @c307f40`). `src/trpc.ts`
wraps the client in a TanStack-Query options proxy so components consume it as
typed queries/mutations (`packages/ui/src/trpc.ts:7-10 @c307f40`). This is the
path for almost everything: agents, connections, secrets, schedules, usage, and
the inbox-side resolution of egress approvals.

### 2. ACP WebSocket — one per active chat session

Chat runs over the Agent Client Protocol on a WebSocket per active session.
`openConnection` (`src/modules/acp/acp.ts`) dials
`/api/agents/:agentId/acp?token=<jwt>` (the bearer token is passed as a query
param because browsers can't set WS headers), adapts the socket into ACP
readable/writable streams, and wraps it in a `ClientSideConnection`
(`packages/ui/src/modules/acp/acp.ts:55-100 @c307f40`). The client implements
the ACP callbacks: `sessionUpdate` streams assistant content/tool calls into the
UI, and `requestPermission` hands the request to the store and blocks until a
human picks an option — there is no client-side auto-approve or timeout
(`packages/ui/src/modules/acp/acp.ts:61-107 @c307f40`). The runtime's custom
`platform/turnEnded` notification is surfaced through the same update channel as
a synthetic `sessionUpdate` so non-originating viewers can close their bubbles
(`packages/ui/src/modules/acp/acp.ts:115-131 @c307f40`).

Permission requests carrying a synthetic `_egress:` sessionId are diverted to
the **inbox** surface (resolved via tRPC) rather than the session-bound
permission queue; their WS promise is intentionally left pending
(`packages/ui/src/modules/acp/acp.ts:68-94 @c307f40`).

The session lifecycle is orchestrated in `src/modules/sessions/hooks/`.
`useAcpSession` is a thin orchestrator composing the connection, engagement,
history, prompt, config-cache, and update-handler sub-hooks
(`packages/ui/src/modules/sessions/hooks/use-acp-session.ts:20-30 @c307f40`).
`useAcpConnection` owns the connection state machine
(`idle | live | reloading | reconnecting`), reconnecting with a fixed backoff
schedule and replaying history before reconnect so the two paths can't drift
(`packages/ui/src/modules/sessions/hooks/use-acp-connection.ts:18-60 @c307f40`,
`packages/ui/src/modules/acp/utils.ts:5 @c307f40`). Incoming ACP updates are
folded into the message list by a pure projection,
`src/modules/acp/session-projection.ts`, shared by live streaming and history
replay (`packages/ui/src/modules/acp/session-projection.ts:16-28 @c307f40`).

### 3. Binary terminal WebSocket

Terminal attach is a separate WebSocket with `binaryType = "arraybuffer"`,
dialed at `/api/agents/:agentId/terminal?token=<jwt>&sessionId=<id>[&reset=1]`
(`packages/ui/src/modules/sessions/components/terminal.tsx:112-116 @c307f40`).
Frames use a compact binary opcode protocol (`OP_OUTPUT`, `OP_INPUT`, `OP_EXIT`,
plus resize), encoded/decoded by helpers imported from the shared
`api-server-api` package so client and server share one wire format
(`packages/ui/src/modules/sessions/components/terminal.tsx:5-12 @c307f40`).
Output frames are written into an xterm.js instance; keystrokes and resize events
are sent back as input/resize frames
(`packages/ui/src/modules/sessions/components/terminal.tsx:125-163 @c307f40`).
Terminal (PTY) sessions never engage ACP
(`packages/ui/src/modules/sessions/hooks/use-acp-connection.ts:33-34 @c307f40`).

## Auth & branding

Login is OIDC against Keycloak via `oidc-client-ts`. `initAuth` fetches
`/api/auth/config` (issuer, client id), constructs the `UserManager`, and must
run before render; `getAccessToken` supplies the bearer token to both tRPC and
the WebSocket URLs (`packages/ui/src/auth.ts:21-50 @c307f40`,
`packages/ui/src/api.ts:31-33 @c307f40`,
`packages/ui/src/modules/acp/acp.ts:55-58 @c307f40`). The current theme is handed
to the Keycloak login page as a `kc_theme` OIDC param so the login screen matches
without a flash (`packages/ui/src/auth.ts:11-19 @c307f40`).

## Build & serving

The app is built to static assets and shipped in a container (`Dockerfile`,
`default.conf`). In dev, Vite runs on port 5173 and proxies `/api` (HTTP and WS)
to the api-server on `localhost:4444`
(`packages/ui/vite.config.ts:43-48 @c307f40`). In production the api-server hosts
the built bundle and terminates all three transports above; see
[api-server](./api-server.md) and the overall [DAM overview](./dam.md).

## Key files

- `src/main.tsx` — bootstrap (auth + brand + terms gate, then render).
- `src/app.tsx` — root view switch and URL ↔ store sync.
- `src/api.ts` / `src/trpc.ts` — tRPC HTTP client and typed query proxy.
- `src/auth.ts` — OIDC/Keycloak login and token accessor.
- `src/store.ts` — composed Zustand store (per-domain slices).
- `src/modules/acp/acp.ts` — ACP WebSocket connection + permission handling.
- `src/modules/acp/session-projection.ts` — pure ACP-update → message projection.
- `src/modules/sessions/hooks/use-acp-connection.ts` — chat connection state machine.
- `src/modules/sessions/components/terminal.tsx` — binary terminal WS + xterm.
- `src/modules/platform/lib/routes.ts` — route/view schema and parsing.
