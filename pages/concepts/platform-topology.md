---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [docs/architecture/platform-topology.md, docs/architecture.md, packages/controller/api/v1/groupversion_info.go, packages/controller/api/v1/agent_types.go]
updated: 2026-06-19
---

# Platform Topology

DAM runs as four long-lived subsystems on Kubernetes, plus a per-agent pod pair that comes and goes with the [agent lifecycle](../concepts/agent-lifecycle.md). The four durable components are a Go **controller**, a TypeScript **api-server**, the per-agent **agent-runtime** (paired with a **gateway**), and a React **ui** served by the api-server (`docs/architecture/platform-topology.md:7 @c307f40`). The cluster boundary is the trust boundary: browsers, the CLI, and Slack reach the platform only through the api-server, and agent egress to LLM/GitHub always exits through the paired gateway (`docs/architecture.md:53 @c307f40`).

The defining structural choice: the controller and the api-server never talk to each other directly. They coordinate entirely through the K8s API, using the `spec`/`status` subresource split so their writes can never contend (`docs/architecture/platform-topology.md:7 @c307f40`).

## Shape

```
browser ──HTTP+WS──▶ ui ──tRPC / ACP·WS──▶ api-server ──ACP relay / tRPC proxy──▶ agent-runtime
                                              │                                        │
                                              │──REST──▶ K8s API ◀──watch+status── controller
                                              │                                        │
                                          ext_authz ◀──gRPC── gateway ◀──HTTPS_PROXY──┘
```

Agent pods are stateless and ephemeral; durable state lives on per-agent persistent volumes, which is what makes the hibernate/wake cycle (scale to zero, scale back up) safe (`docs/architecture/platform-topology.md:7 @c307f40`). See [persistence](../concepts/persistence.md).

## The four components

### controller

A stateless Go reconciler built on client-go. It watches the `Agent` and `Fork` custom resources (API group `agent-platform.ai/v1`) plus agent-labelled pods, and reconciles the StatefulSet, Service, NetworkPolicy, and per-agent Secret for each agent (`docs/architecture/platform-topology.md:37 @c307f40`; group/version confirmed at `packages/controller/api/v1/groupversion_info.go:18 @c307f40`). It computes agent readiness from the pod pair onto the Agent status and hibernates idle agents by scaling the pair to zero. The controller writes **only** the `status` subresource on resources it owns; it never writes `spec`. The schedule loop lives in the api-server, not here (`docs/architecture/platform-topology.md:37 @c307f40`). See [controller](../sources/controller.md).

### api-server

A TypeScript server hosting the user-facing surface and the ACP relay. It runs **two listeners** that share no routes (`docs/architecture/platform-topology.md:41-44 @c307f40`):

- **Public port** — user-authenticated tRPC, REST (OAuth callbacks, health, version), and the ACP relay WebSocket. The `version` endpoint is unauthenticated and powers the CLI compatibility-floor check (`docs/architecture/platform-topology.md:43 @c307f40`).
- **Harness port** — cluster-internal only, consumed by agent pods for trigger handoff and MCP tool calls. Not exposed outside the cluster; carries no user authentication (`docs/architecture/platform-topology.md:44 @c307f40`).

All ACP traffic is proxied through the api-server — clients never dial pods directly — and it wakes hibernated agents on demand before forwarding the first message of a session (`docs/architecture/platform-topology.md:46 @c307f40`). Both the ACP relay and the tRPC proxy verify the caller (a Keycloak JWT or an API key, dispatched by token prefix in the same `Authorization: Bearer` slot), check ownership at the public port, then rewrite `Authorization` to the per-agent runtime token before forwarding — so agent-runtime never sees user identity directly (`docs/architecture/platform-topology.md:46 @c307f40`). See [security-and-credentials](../concepts/security-and-credentials.md) and [api-server](../sources/api-server.md).

The public port also streams bundled file imports per agent through to the target agent-runtime without buffering, ownership-checked and size-capped at the proxy boundary (`docs/architecture/platform-topology.md:48 @c307f40`). A session's mode is agent-owned metadata, persisted over ACP (`session/resume` carrying `_meta.platform.mode`); there is no server-side mode-change side effect and no cross-client broadcast (`docs/architecture/platform-topology.md:50 @c307f40`).

### agent-runtime (+ gateway)

The per-agent pod that runs the ACP WebSocket server and spawns the underlying agent binary via the harness-script contract (`docs/architecture/platform-topology.md:54 @c307f40`). Its responsibilities:

- Accept one relayed ACP WebSocket connection and speak JSON-RPC 2.0 to the agent process; chat-mode sessions spawn `/usr/local/bin/harness-chat` (`docs/architecture/platform-topology.md:56 @c307f40`).
- Accept terminal-mode WebSocket connections on `/api/terminal` — each gets a PTY running `/usr/local/bin/harness-terminal`, with a binary input/output/resize frame protocol and scrollback that reattaches within a 30 s grace window (`docs/architecture/platform-topology.md:57 @c307f40`).
- Accept SSH WebSocket connections on `/api/ssh` — each spawns a per-connection `sshd -i` as the agent user, relaying raw bytes verbatim; this is `dam ssh`'s transport, available only on images shipping `sshd` (`docs/architecture/platform-topology.md:58 @c307f40`).
- Hold the agent side of the runtime channel: call the api-server's `hello` on boot/reconnect, accept `applyState` deliveries over its tRPC surface, and dispatch runtime events (schedule triggers, workspace seeding) to in-pod handlers (`docs/architecture/platform-topology.md:59 @c307f40`). See [connections](../concepts/connections.md).
- Expose a scoped tRPC router (via the api-server's tRPC proxy) for in-pod file operations surfaced to the UI (`docs/architecture/platform-topology.md:60 @c307f40`).
- Accept bundled file imports on the harness port, extracting the tarball to PVC staging then atomically moving top-level entries into `<homeDir>/work`; one import per agent at a time, with a boot sweeper reclaiming orphaned staging dirs (`docs/architecture/platform-topology.md:61 @c307f40`). See [persistence](../concepts/persistence.md).

Crucially, the agent-runtime pod holds **zero** credential Secrets and has no admitted route to TCP 80/443 except its paired gateway. Its `HTTPS_PROXY` points at the gateway Service DNS, but that value is decorative — Kubernetes admits no other route (`docs/architecture/platform-topology.md:63 @c307f40`). See [agent-runtime](../sources/agent-runtime.md) and the [agent](../entities/agent.md) entity.

The **gateway** is a per-agent Envoy pod paired with agent-runtime. It mounts the owner's credential Secrets, cert-manager-issued leaf TLS material, and the rendered Envoy bootstrap ConfigMap; terminates the agent's egress TLS, injects credentials on the wire, and gates each credentialed request through the api-server's ext_authz handler (`docs/architecture/platform-topology.md:67 @c307f40`). See [security-and-credentials](../concepts/security-and-credentials.md).

### ui

A React + Vite SPA served by the api-server. It uses tRPC over HTTP for resource management and permission flows, and opens one ACP WebSocket per active session — permission prompts, tool calls, and streaming output all flow over that same ACP connection (`docs/architecture/platform-topology.md:71 @c307f40`). See [ui](../sources/ui.md).

## Protocols between components

| Edge | Protocol | Purpose |
|------|----------|---------|
| ui → api-server | tRPC over HTTP | Resource CRUD and single-file uploads (`docs/architecture/platform-topology.md:77 @c307f40`) |
| ui → api-server | WebSocket (ACP, JSON-RPC 2.0) | Live chat, permission prompts, streaming, plus session list/create/delete and mode changes — sessions are agent-owned (`docs/architecture/platform-topology.md:78 @c307f40`) |
| ui → api-server | WebSocket (binary terminal frames) | Live terminal: input / output / resize / exit (`docs/architecture/platform-topology.md:79 @c307f40`) |
| cli → api-server | tRPC over HTTP + WS frames | Agent resolution, auth, and `dam chat` terminal attach (`docs/architecture/platform-topology.md:80-81 @c307f40`) |
| api-server → agent-runtime | WebSocket (ACP, JSON-RPC 2.0) | Chat-mode relay target — one hop, no fan-out (`docs/architecture/platform-topology.md:82 @c307f40`) |
| api-server → agent-runtime | WebSocket (binary terminal frames) | Terminal relay — one hop, single client per session (`docs/architecture/platform-topology.md:83 @c307f40`) |
| api-server → agent-runtime | HTTP (tRPC proxy / `applyState`) | In-pod file ops, and outbox-worker runtime-channel delivery (`docs/architecture/platform-topology.md:84,90 @c307f40`) |
| ui/cli → api-server → agent-runtime | HTTP (multipart, streamed) | Bundled file import (`docs/architecture/platform-topology.md:85 @c307f40`) |
| agent-runtime → api-server (harness, via gateway → Istio waypoint) | HTTP | MCP tool access, runtime-channel `hello` (`docs/architecture/platform-topology.md:86 @c307f40`) |
| gateway → api-server | gRPC | HITL ext_authz `Check`, per-agent Service pinned by AuthorizationPolicy to the gateway SA principal (`docs/architecture/platform-topology.md:87 @c307f40`) |
| controller → K8s API | watch / list / status write | Resource reconciliation and status writes (`docs/architecture/platform-topology.md:88 @c307f40`) |
| api-server → K8s API | REST | Resource CRUD, spec writes, pod wake (`docs/architecture/platform-topology.md:89 @c307f40`) |

ACP frames are JSON-RPC 2.0, one logical message per WebSocket frame (`docs/architecture/platform-topology.md:92 @c307f40`). The api-server additionally reaches Keycloak (JWKS validate), Postgres (metadata), and Redis (BullMQ jobs / pub-sub) (`docs/architecture.md:41-43 @c307f40`).

## K8s resource model

The controller-reconciled domain resources are CRDs under the `agent-platform.ai/v1` API group, each with a status subresource (`docs/architecture/platform-topology.md:96 @c307f40`):

- **`spec`** — user intent. Owned exclusively by the api-server; validated at admission (`docs/architecture/platform-topology.md:98 @c307f40`).
- **`status`** — observed state. Owned exclusively by the controller via the status subresource. Conditions `Ready`, `AgentPodReady`, `GatewayPodReady`, `Reconciled` are the source of truth; the api-server routes on `Ready` alone and never inspects pods itself (`docs/architecture/platform-topology.md:99 @c307f40`; condition names defined at `packages/controller/api/v1/agent_types.go:83-92 @c307f40`).

| Kind | Purpose |
|---|---|
| `Agent` | The sole resource per agent: image, mounts, env, secret refs, granted secret/connection IDs. No separate instance resource, no `desiredState` — running-vs-hibernated is derived from activity annotations (`docs/architecture/platform-topology.md:103 @c307f40`) |
| `Fork` | A forked run: parent agent ref + overrides (`docs/architecture/platform-topology.md:104 @c307f40`) |

Two domain resources are deliberately **not** CRDs: **Templates** are chart-rendered ConfigMaps loaded at boot (read-only, never reconciled), and **Schedules** are Postgres rows owned by the api-server (`docs/architecture/platform-topology.md:106 @c307f40`). See [persistence](../concepts/persistence.md).

For each `Agent`, the controller reconciles **two paired StatefulSets** (agent + gateway, replicas 0 when hibernated, 1 when running), **two pair-scoped Services** (the agent's ACP and the `<agent>-gateway` proxy DNS), a **per-agent ServiceAccount**, a **per-agent ext-authz Service** (`<release>-extauthz-<id>`), **two per-agent Istio AuthorizationPolicies**, and a per-agent Envoy bootstrap ConfigMap + leaf-TLS Certificate (`docs/architecture/platform-topology.md:108 @c307f40`). Installing the CRDs requires cluster-admin at install time — moving to CRDs deliberately gave up the namespace-scoped install the earlier ConfigMap model allowed (`docs/architecture/platform-topology.md:108 @c307f40`).

## Invariants

- **Spec/status ownership.** Controller never writes `spec`; api-server never writes `status`. The status subresource makes the split structural — write contention is impossible (`docs/architecture/platform-topology.md:112 @c307f40`).
- **Relay-only ACP.** All ACP traffic is proxied through the api-server; agent pods accept no external ACP connections and the UI never dials pods directly (`docs/architecture/platform-topology.md:113 @c307f40`).
- **Two-port api-server.** Public port is user-authenticated; harness port is cluster-internal with no user auth. They share no routes (`docs/architecture/platform-topology.md:114 @c307f40`).
- **Credential isolation.** Agent pods never hold real upstream credentials; the paired gateway injects them from a Secret mounted only on the gateway, and the agent pod has no path to TCP 80/443 except through that gateway (`docs/architecture/platform-topology.md:115 @c307f40`). See [security-and-credentials](../concepts/security-and-credentials.md).
- **SPIFFE identity per hop.** Three hops, the latter two gated by per-agent Istio AuthorizationPolicies; no app-layer header conveys identity (`docs/architecture/platform-topology.md:116 @c307f40`).
- **Durable triggers.** Schedule fires are Postgres rows in the runtime outbox, delivered over the runtime channel only when the agent is `Ready`; an undelivered fire survives pod and api-server restarts (`docs/architecture/platform-topology.md:117 @c307f40`). See [connections](../concepts/connections.md).
