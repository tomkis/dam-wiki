---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [packages/controller/api/v1/agent_types.go, docs/architecture/platform-topology.md, docs/ubiquitous-language.md]
updated: 2026-06-19
---

# Agent

The **Agent** is DAM's central domain entity — "the durable, owned, runnable resource: definition, runtime state, and lifecycle in one custom resource" (`packages/controller/api/v1/agent_types.go:153 @c307f40`). It is the aggregate root of the Agents bounded context: a user owns Agents, the api-server defines them, and the controller runs them. The merged design retired a separate `Instance` resource — the Agent now carries definition, runtime state, and lifecycle by itself (`docs/ubiquitous-language.md:20 @c307f40`, `agent_types.go:7-8 @c307f40`).

Across the redesigned UI an Agent is surfaced to users as a **Sandbox**; "Agent" remains the domain and code term (`docs/ubiquitous-language.md:26 @c307f40`).

## Resource shape

The Agent is a namespaced Custom Resource in the `agent-platform.ai/v1` API group, with a `status` subresource and short name `agt` (`agent_types.go:143-161 @c307f40`). It is one of two controller-reconciled CRDs in the platform; the other is [`Fork`](#relationship-to-template-and-fork) (`docs/architecture/platform-topology.md:96-104 @c307f40`).

```go
type Agent struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    Spec   AgentSpec
    Status AgentStatus
}
```
(`agent_types.go:155-161 @c307f40`)

### `spec` — desired state (api-server writer)

`AgentSpec` is the sole durable per-agent intent. The **api-server is the sole writer** (`agent_types.go:7-9 @c307f40`). Fields:

| Field | Meaning |
|---|---|
| `image` (required) | The agent container image (`agent_types.go:16-17 @c307f40`) |
| `name`, `description` | Optional human-readable labels (`agent_types.go:19-24 @c307f40`) |
| `init` | Optional one-shot init script run before the agent starts (`agent_types.go:25-27 @c307f40`) |
| `mounts` | Declares volumes; a persisted mount becomes a PVC (`agent_types.go:28-30 @c307f40`) |
| `env` | Plain name/value env vars projected into the agent container (`agent_types.go:31-33 @c307f40`) |
| `resources` | Container requests/limits as K8s Quantity strings (`agent_types.go:34-36 @c307f40`, `:134-141 @c307f40`) |
| `imagePullPolicy`, `storageSize` | Per-agent overrides of chart-wide defaults; empty = inherit (`agent_types.go:38-43 @c307f40`) |
| `agentHome` | Resolved `HOME` inside the container; `$HOME` literals in mounts are pre-resolved at write time so the controller never sees `$HOME` (`agent_types.go:44-48 @c307f40`) |
| `secretRef` | A K8s Secret whose keys are `envFrom`-projected into the agent container (operator-supplied envs) (`agent_types.go:50-53 @c307f40`) |
| `imagePullSecretRef` | A `dockerconfigjson` Secret the kubelet uses to pull a private image; never projected into the container, so a foreign-replier [Fork](#relationship-to-template-and-fork) can pull without seeing it; takes precedence over the install-wide default pull secret (`agent_types.go:55-62 @c307f40`) |
| `grantedSecretIds` | Credential Secret IDs granted to this agent's egress; intent the controller reconciles into the credential set mounted on the gateway (`agent_types.go:64-69 @c307f40`) |
| `grantedConnectionIds` | Connection IDs granted to this agent (`agent_types.go:70-72 @c307f40`) |

Two things are deliberately **not** in spec:

- **No `desiredState` field.** Running-vs-hibernated is not stored intent but observed status the controller derives from activity (`agent_types.go:11-12 @c307f40`, `platform-topology.md:103 @c307f40`). Hibernation scales the pod pair to zero; the wake/hibernate cycle is covered in [agent-lifecycle](../concepts/agent-lifecycle.md).
- **No security context or scheduling.** These are chart-only (`config.AgentBase`) and cannot be set per-agent by design (`agent_types.go:12-14 @c307f40`).

### `status` — observed state (controller writer)

`AgentStatus` is the observed state; the **controller is the sole writer**, via the status subresource (`agent_types.go:100-102 @c307f40`). It holds only:

- `conditions` — the source of truth for operational state (`agent_types.go:103-108 @c307f40`)
- `observedGeneration` — the spec generation last reconciled (`agent_types.go:109-111 @c307f40`)

There is no `phase` field — the conditions are the only status the api-server reads (`agent_types.go:76-77 @c307f40`).

#### Conditions

| Condition | Meaning |
|---|---|
| `Ready` | Overall readiness — the intersection of agent-pod and gateway-pod readiness. The api-server treats this as the authoritative "can I route to this agent?" signal (`agent_types.go:79-83 @c307f40`) |
| `AgentPodReady` | Mirrors the agent pod's observed `Ready` condition (`agent_types.go:84-85 @c307f40`) |
| `GatewayPodReady` | Mirrors the paired gateway pod's `Ready` condition; required for credentialed egress, so a required input to `Ready` (`agent_types.go:86-89 @c307f40`) |
| `Reconciled` | Whether the controller accepted and rendered the spec; its message carries the last reconcile error, if any (`agent_types.go:90-92 @c307f40`) |

A hibernated agent reports `Ready=False` with reason `Hibernated`, distinguishing it from one that is still starting (also `Ready=False`) (`agent_types.go:95-98 @c307f40`). The api-server routes on `Ready` alone and never inspects pods itself (`platform-topology.md:99 @c307f40`).

## The spec / status single-writer split

The Agent is reconciled across two processes that never talk to each other directly — they coordinate only through the K8s API, using the `spec`/`status` subresource split so writes never contend (`platform-topology.md:7 @c307f40`):

- **`spec`** — user intent, owned exclusively by the api-server, validated by the K8s API server at admission (`platform-topology.md:98 @c307f40`).
- **`status`** — observed state, owned exclusively by the controller, written through the status subresource (`platform-topology.md:99 @c307f40`).

This is a structural invariant: the controller never writes `spec`; the api-server never writes `status`. The status subresource makes write contention between the two impossible (`platform-topology.md:112 @c307f40`). See [persistence](../concepts/persistence.md) for the substrate split and [platform-topology](../concepts/platform-topology.md) for the broader topology.

> Note: `docs/ubiquitous-language.md:25 @c307f40` still describes the Agent as "a single ConfigMap whose `spec.yaml` / `status.yaml`" carries this state and mentions a `desiredState` field. That entry lags the current model: the authoritative Go types (`agent_types.go`) and topology doc define the Agent as a CRD with `spec`/`status` subresources and **no** `desiredState`. The single-writer split (api-server → spec, controller → status) is identical in both descriptions.

## What the controller reconciles per Agent

For each `Agent`, the controller renders and reconciles a fixed fan-out of Kubernetes resources (`platform-topology.md:107-108 @c307f40`, `:37 @c307f40`):

- **Two paired StatefulSets** — the agent pod and the gateway pod, both at replicas 0 when hibernated and 1 when running.
- **Two pair-scoped Services** — the agent's ACP Service and the gateway's `<agent>-gateway` proxy DNS.
- **A per-agent ServiceAccount** (in the agent namespace).
- **A per-agent ext-authz Service** (`<release>-extauthz-<id>`, in the release namespace).
- **Two per-agent Istio AuthorizationPolicies** — the harness path-prefix policy at the waypoint and the ext-authz Service principal policy.
- **A per-agent Envoy bootstrap ConfigMap + leaf-TLS Certificate.**

The controller also computes `Ready`/`AgentPodReady`/`GatewayPodReady` from the pod pair onto status, and hibernates idle agents by scaling the pair to zero (`platform-topology.md:37 @c307f40`). It owns `status` only — never `spec` (`platform-topology.md:37 @c307f40`). See [controller](../sources/controller.md).

## Relationship to Template and Fork

- **Template** — a read-only catalog blueprint defining a base image, mounts, env, and resources for creating an agent (`docs/ubiquitous-language.md:24 @c307f40`). An Agent is **optionally derived from a Template at create-time** (`docs/ubiquitous-language.md:25 @c307f40`). Templates are deliberately **not** CRDs — they are chart-rendered ConfigMaps the api-server loads at boot, read-only and never reconciled (`platform-topology.md:106 @c307f40`). The Agent is the durable runnable thing; the Template is just the blueprint stamped at birth.

- **Fork** — an ephemeral, per-turn execution environment **derived from an Agent** that impersonates a foreign user for one Slack turn; it is Job-shaped and run-to-completion, distinct from the Agent's StatefulSet shape (`docs/ubiquitous-language.md:49 @c307f40`). `Fork` is the second controller-reconciled CRD: a forked run carrying a parent-agent ref plus overrides (`platform-topology.md:104 @c307f40`). The Agent's `imagePullSecretRef` is structured so a foreign-replier Fork can pull the image without ever seeing the secret (`agent_types.go:55-62 @c307f40`).

## The reserved `agent-` ID prefix

`agent-` is a reserved prefix the controller mints onto **every** Agent ID. Its consequences span three subsystems (`docs/ubiquitous-language.md:32 @c307f40`):

- The **api-server** forbids Agent names that begin with `agent-` at create-time.
- The **CLI** uses it as the syntactic ID-vs-name split signal: an Agent Ref starting with `agent-` is treated as an Agent ID, otherwise as a name (`docs/ubiquitous-language.md:32 @c307f40`, `:170 @c307f40`).

Because no user-chosen name can collide with the prefix, an Agent Ref is unambiguously classifiable without a server round-trip.

## See also

- [agent-lifecycle](../concepts/agent-lifecycle.md) — create, hibernate/wake (derived running state), delete.
- [persistence](../concepts/persistence.md) — Infra State vs Application State; the workspace volume.
- [platform-topology](../concepts/platform-topology.md) — the four subsystems and the K8s resource model.
- [bounded-contexts](../concepts/bounded-contexts.md) — the Agents context and its neighbours.
- [controller](../sources/controller.md) — the reconciler that owns Agent `status`.
