# Wiki index

Content catalog. One line per page, grouped by category. Maintained by `ingest`
and `lint`. Each line is a bullet linking to a page under `pages/CATEGORY/`,
followed by an em-dash and a one-line hook.

This wiki distils **`dam-agents/dam`** — a Kubernetes platform for running AI
agent harnesses headless in the cloud. Start at the [repo overview](pages/sources/dam.md)
or the [platform topology](pages/concepts/platform-topology.md).

## Sources

- [dam-agents/dam — repository overview](pages/sources/dam.md) — what DAM is, the monorepo layout, the architecture-doc map, and build tooling.
- [controller (package)](pages/sources/controller.md) — the Go K8s reconciler + scheduler: CRDs, control loop, idle checker, warm pool.
- [api-server (package)](pages/sources/api-server.md) — the TypeScript server: two-port model, tRPC routers, ACP relay, MCP endpoint, schedule loop, ext_authz.
- [agent-runtime (package)](pages/sources/agent-runtime.md) — the in-pod ACP WebSocket server: harness spawn, runtime channel, file ops, terminal/SSH.
- [ui (package)](pages/sources/ui.md) — the React + Vite SPA: tRPC, one ACP WebSocket per session, binary terminal frames.
- [cli (package)](pages/sources/cli.md) — the npm-distributed `dam` command-line client: agent management, chat/ssh attach, version floor check.
- [db (package)](pages/sources/db.md) — Postgres schema & migrations (Drizzle) plus the hand-written `usage_*` reporting views.

## Entities

- [Agent](pages/entities/agent.md) — DAM's central domain resource: the Agent CRD, its spec/status single-writer split, and its relation to Template and Fork.

## Concepts

- [Platform Topology](pages/concepts/platform-topology.md) — the four long-lived components, the protocols between them, and the K8s resource model.
- [Agent Lifecycle](pages/concepts/agent-lifecycle.md) — create → wake → trigger → hibernate → delete; the no-desired-state model; sessions and forks.
- [Persistence & the Substrate Split](pages/concepts/persistence.md) — Postgres vs ConfigMap vs PVC; the warm pool; what survives each lifecycle event.
- [Security & Credentials](pages/concepts/security-and-credentials.md) — Keycloak identity, the Envoy credential gateway, ext_authz HITL, the network boundary.
- [Channels](pages/concepts/channels.md) — Slack & Telegram adapters, inbound ACP relay, the outbound MCP tool, foreign-replier forks.
- [Connections](pages/concepts/connections.md) — the unified Connection/Contribution model and the transactional-outbox runtime channel.
- [Skills](pages/concepts/skills.md) — connectable git skill sources, install onto the PVC, publish-back-as-PR, and the two-sided skills context.
- [Usage Tracking](pages/concepts/usage-tracking.md) — append-only activity log, SQL views, HMAC-pseudonymized identifiers, inspector-role gating.
- [Logging & Audit Trail](pages/concepts/logging.md) — Pino structured logging and the real-identity security audit trail.
- [Bounded Contexts & Ubiquitous Language](pages/concepts/bounded-contexts.md) — the DDD glossary and orientation hub for every domain term.
