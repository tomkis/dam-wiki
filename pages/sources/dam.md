---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [README.md, CLAUDE.md, docs/architecture.md, docs/architecture/platform-topology.md, pnpm-workspace.yaml]
updated: 2026-06-19
---

# dam-agents/dam — repository overview

**DAM** is a Kubernetes platform for running AI agent harnesses (Claude Code, Codex, Pi Agent, Bob) headless in the cloud (`README.md:10 @c307f40`). Agents execute continuously in the cloud and keep running after you close your laptop (`README.md:25 @c307f40`); each agent runs in an isolated container with all egress routed through a policy-enforced gateway (`README.md:26 @c307f40`), and credentials are connected to agents without ever being exposed to the runtime — "zero-trust credentials" (`README.md:27 @c307f40`). See [security-and-credentials](../concepts/security-and-credentials.md) for how that isolation is enforced.

This page is the entry point for the wiki. For the runtime shape of the system, start with [platform topology](../concepts/platform-topology.md); for the domain language, see [bounded contexts](../concepts/bounded-contexts.md).

## What it is

DAM exposes four ways to drive an agent (`README.md @c307f40`): a browser **Web UI** (chat + streamed terminal + file management), a **CLI** (`dam`), **Slack** (teammates interact with their own credentials), and **Schedules** (recurring timer-driven runs). Supported harnesses include Claude Code, Pi Agent, Bob, and Codex; any ACP-compatible runtime can run on DAM (`README.md:43-48 @c307f40`).

The central domain object is the [**Agent**](../entities/agent.md) — the durable, owned, runnable resource that scales between zero and one pod and hibernates when idle. Its full create→wake→trigger→hibernate→delete story is on [agent lifecycle](../concepts/agent-lifecycle.md).

## Monorepo layout

DAM is a pnpm-workspace monorepo plus a standalone Go module (`pnpm-workspace.yaml:2 @c307f40`). Concept depth lives in `docs/architecture/`; the package map is (`CLAUDE.md:9-16 @c307f40`):

| Package | Language | Role | Wiki page |
|---|---|---|---|
| `packages/controller/` | Go | K8s reconciler + scheduler | [controller](controller.md) |
| `packages/api-server/` + `packages/api-server-api/` | TS | API server (tRPC, ACP relay) + contract | [api-server](api-server.md) |
| `packages/agent-runtime/` + `packages/agent-runtime-api/` | TS | in-pod ACP WebSocket server + contract | [agent-runtime](agent-runtime.md) |
| `packages/agents/` | mixed | per-harness agent images (`claude-code`, `pi-agent`, `codex`, `bob`) | — |
| `packages/ui/` | TS/React | React chat interface (Vite) | [ui](ui.md) |
| `packages/platform-base/` | — | shared base image/utilities | — |
| `packages/db/` | TS/SQL | Postgres schema & migrations | [db](db.md) |
| `packages/cli/` | TS | the `dam` command-line client | [cli](cli.md) |
| `packages/keycloak-theme/` | — | Keycloak login theme | — |
| `packages/e2e/` | TS | Playwright end-to-end tests + mock agents | — |
| `packages/dev-config/` | — | shared dev/lint config | — |
| `deploy/helm/platform/` | Helm | chart for all components + PostgreSQL | — |

The four long-lived components (controller, api-server, agent-runtime, ui) plus the per-agent gateway are described in [platform topology](../concepts/platform-topology.md) (`docs/architecture/platform-topology.md:7 @c307f40`).

## Architecture documentation

`docs/architecture.md` is the authoritative system map and the source of truth for behavior — the repo's own guidance is *not* to infer architecture from code when an architecture page exists (`docs/architecture.md:55 @c307f40`). Each subsystem doc maps to a wiki concept page:

- [platform-topology](../concepts/platform-topology.md) — components, protocols, K8s resource model
- [agent-lifecycle](../concepts/agent-lifecycle.md) — create → wake → trigger → hibernate → delete; sessions; forks
- [persistence](../concepts/persistence.md) — the three substrates and what survives each lifecycle event
- [security-and-credentials](../concepts/security-and-credentials.md) — Keycloak, the Envoy credential gateway, ext_authz HITL
- [channels](../concepts/channels.md) — Slack & Telegram adapters, inbound relay, outbound MCP tool
- [connections](../concepts/connections.md) — the Connection/Contribution model and the runtime channel
- [skills](../concepts/skills.md) — connectable git skill sources, install, publish-as-PR
- [usage-tracking](../concepts/usage-tracking.md) — append-only activity log, pseudonymized identifiers
- [logging](../concepts/logging.md) — Pino structured logs and the real-identity audit trail
- [bounded-contexts](../concepts/bounded-contexts.md) — the ubiquitous-language glossary and DDD discipline

> **Note (off-limits sources):** the source repo treats `docs/adrs/` as off-limits to agents. This wiki does **not** read, cite, or link ADRs; the architecture docs above and `docs/ubiquitous-language.md` are used as the source of truth instead.

## Build & tooling

**mise** is the task runner; all tasks live in `tasks.toml` files, and the repo's rule is to always use `mise run` rather than invoking `go`/`pnpm`/`helm`/`kubectl` directly (`CLAUDE.md:20 @c307f40`). `mise run check` lint- and type-checks all packages and also runs as the pre-commit hook (`CLAUDE.md:23 @c307f40`). The local cluster runs on k3s via lima; a family of `cluster:*` and `e2e:*` tasks build images and manage the dev/test clusters.

Database table/index/enum migrations are **generated** from `packages/db/src/schema.ts`, while the `usage_*` reporting views are **hand-written** raw SQL outside the schema (`CLAUDE.md:80 @c307f40`). See [db](db.md) and [usage-tracking](../concepts/usage-tracking.md).

> **Note (brand vs codename):** the codename `platform` is permanent in code; the user-visible brand ("DAM") flows through Helm `brand.*` → api-server `config.brand` → UI `getBrand()`, and must never be hardcoded (`CLAUDE.md:115 @c307f40`). This is why architecture docs and identifiers say "Platform" where the product says "DAM".
