---
source: dam-agents/dam
commit: 9d1bc9990fba55f43d60b0aad453b188af5896a8
files:
  - packages/controller/main.go
  - packages/controller/Dockerfile
  - packages/controller/api/v1/groupversion_info.go
  - packages/controller/api/v1/agent_types.go
  - packages/controller/api/v1/fork_types.go
  - packages/controller/pkg/reconciler/crd.go
  - packages/controller/pkg/reconciler/agent_reconciler.go
  - packages/controller/pkg/reconciler/idlechecker.go
  - packages/controller/pkg/reconciler/hibernation.go
  - packages/controller/pkg/reconciler/warmpool.go
  - packages/controller/pkg/reconciler/resources.go
  - packages/controller/pkg/reconciler/envoy.go
  - packages/controller/pkg/reconciler/gateway.go
  - packages/controller/pkg/reconciler/fork.go
  - packages/controller/pkg/reconciler/status.go
  - packages/controller/pkg/reconciler/agent.go
  - packages/controller/pkg/config/config.go
updated: 2026-06-22
---

# controller (package)

`packages/controller/` is DAM's Kubernetes control plane: a stateless Go
reconciler, built on `client-go`, that turns [Agent](../entities/agent.md) and
Fork custom resources into the running pods, Services, NetworkPolicies, Secrets,
and Envoy config that make up an agent. It is one of the four long-lived
subsystems in the [platform topology](../concepts/platform-topology.md); the
api-server writes resource `spec`, the controller writes only `status`, and the
two coordinate exclusively through the K8s API — they never call each other.

The controller is the actor behind the scale-to-zero half of the
[agent lifecycle](../concepts/agent-lifecycle.md): it derives run-vs-hibernate
from activity (there is no stored desired state) and hibernates quiet agents by
scaling their pod pair to zero.

## Custom resources (`api/v1/`)

Group/version is `agent-platform.ai/v1`; the scheme registers `Agent` and `Fork`
(`api/v1/groupversion_info.go:18` @c307f40, `:27` @c307f40). The api-server is
the sole writer of each resource's `spec`; the controller is the sole writer of
its `status` subresource (`api/v1/groupversion_info.go:1` @c307f40).

- **Agent** — the durable, owned, runnable resource: definition, runtime state,
  and lifecycle in one CR (`api/v1/agent_types.go:153` @c307f40). `AgentSpec`
  carries image, mounts (a persisted mount becomes a PVC), env, resource
  requests, secret refs, and the granted credential / connection IDs that scope
  egress (`api/v1/agent_types.go:15` @c307f40). There is **no** `desiredState`
  field — running-vs-hibernated is observed status the controller derives from
  activity (`api/v1/agent_types.go:7` @c307f40). Status is a set of
  conditions (`Ready`, `AgentPodReady`, `GatewayPodReady`, `Reconciled`); the
  api-server routes on `Ready` (`api/v1/agent_types.go:78` @c307f40). A
  hibernated agent is distinguished from a still-starting one by the
  `Hibernated` reason stamped on its conditions (`api/v1/agent_types.go:95`
  @c307f40).
- **Fork** — a per-turn, ephemeral impersonation run derived from a parent Agent;
  `ForkSpec` names the parent and the foreign identity it runs as, and
  `ForkStatus` tracks a phase `Pending|Ready|Failed|Completed`
  (`api/v1/fork_types.go:11` @c307f40, `:21` @c307f40).

`api/v1/zz_generated.deepcopy.go` and `api/v1/schema_generation.go` are codegen
support; `pkg/crdcheck/` asserts at startup that the cluster's CRD schema matches
this build (`pkg/crdcheck/crdcheck.go:23` @c307f40).

## Entrypoint and control loop (`main.go`)

The binary configures a JSON `log/slog` handler on **stderr** at the `LOG_LEVEL`
level (`debug|info|warn|error`, default `info`) via `setupLogger()`, then loads
config from env, builds in-cluster typed + dynamic clients (raising client-go's
default QPS/Burst to 50/100), and fails fast on a stale CRD schema before doing
anything else (`packages/controller/main.go:99-112` @9d1bc99, `:30-70` @9d1bc99).
There is no audit trail — the controller acts only under its own ServiceAccount
and carries no real actor. The binary registers no in-process OpenTelemetry SDK
so the OTel Operator's eBPF auto-instrumentation sidecar can capture log records
with trace correlation (`docs/architecture/logging.md:53-57 @9d1bc99`). It then
runs under **leader election** (a 15s Lease), so only one replica reconciles at a
time (`packages/controller/main.go:74` @9d1bc99).

The elected leader's `run` wires the loop (`main.go:112` @c307f40):

- **Informers.** Agents and Forks are watched via *dynamic* informers off a
  shared filtered factory (they surface as `unstructured`); a separate typed Pod
  informer pins the `agent-platform.ai/agent` label selector
  (`main.go:117` @c307f40, `:124` @c307f40).
- **Work queues.** Two rate-limited string queues (agent, fork). CR add/update
  events enqueue by name; CR delete events call the reconciler's `Delete`
  teardown directly; pod readiness transitions re-enqueue the *owning* agent so
  its `Ready` conditions recompute (`main.go:158` @c307f40, `:182` @c307f40,
  `:292` @c307f40).
- **Workers.** `runAgentWorker` / `runForkWorker` drain their queues, resolving
  each object from the informer cache and calling `Reconcile`. On error they
  `AddRateLimited` (never `Forget`, which would reset to fast retries); after
  `maxReconcileRetries` (15) consecutive failures an agent is marked
  `BackoffLimitExceeded` for visibility while retries continue at the capped
  cadence (`main.go:204` @c307f40, `:224` @c307f40).
- **Background goroutines.** The leader also starts the idle checker, the warm
  PVC pool, and a 10-minute orphan sweep that GCs PVCs and leaf TLS Secrets
  whose Agent was removed out-of-band (`main.go:137` @c307f40, `:143` @c307f40,
  `:151` @c307f40, `:322` @c307f40).

The image entrypoint is the built `controller` binary (`Dockerfile:25` @c307f40).

## Reconciliation (`pkg/reconciler/`)

This is the heart of the package. The owned-resource conversion helpers and
GVRs/GVKs live in `crd.go`; every child the reconciler renders carries a
controller owner-reference to its CR so K8s GC cascade-deletes the whole set
(`pkg/reconciler/crd.go:18` @c307f40, `:54` @c307f40).

### Agent reconcile (`agent_reconciler.go`)

`AgentReconciler.Reconcile` renders an Agent into a **paired pod set** plus its
security surface, in a fixed order, then publishes status
(`pkg/reconciler/agent_reconciler.go:65` @c307f40). The owned resources, in
render order:

1. **Envoy bootstrap ConfigMap** — the per-agent credential-injecting proxy
   config, built from the agent's granted credential Secrets
   (`pkg/reconciler/agent_reconciler.go:86` @c307f40).
2. **Envoy leaf TLS Certificate** (cert-manager) — applied via the dynamic
   client; because cert-manager doesn't owner-ref the Secret it produces, the
   reconciler patches an owner-reference onto it so GC reaps it
   (`pkg/reconciler/agent_reconciler.go:95` @c307f40).
3. **Per-agent ServiceAccount** — must exist before the pods (kubelet rejects a
   missing SA; Istio stamps its SPIFFE cert from it)
   (`pkg/reconciler/agent_reconciler.go:115` @c307f40).
4. **Ext-authz Service + two AuthorizationPolicies** — the gateway pod's Envoy
   dials the per-agent ext-authz Service for HITL approvals; the AuthZ policies
   pin the gateway's SPIFFE identity to the harness path-prefix and ext-authz
   principal (`pkg/reconciler/agent_reconciler.go:124` @c307f40, `:135` @c307f40).
5. **Agent-egress NetworkPolicy** — agent pods opt out of the ambient mesh, so
   kernel NP is the only thing gating their egress; it admits DNS and the paired
   gateway's Envoy port only (`pkg/reconciler/agent_reconciler.go:148` @c307f40).
6. **Gateway StatefulSet + ClusterIP Service**, rendered *before* the agent so
   the agent's `HTTPS_PROXY` target exists when it starts. The gateway
   ClusterIP is captured synchronously; if not yet assigned the reconcile
   requeues *quietly* (a transient wait, not a `ReconcileError`)
   (`pkg/reconciler/agent_reconciler.go:169` @c307f40, `:199` @c307f40).
7. **Agent StatefulSet + headless Service**, with any matching warm-pool PVC
   claims applied to the volume templates
   (`pkg/reconciler/agent_reconciler.go:205` @c307f40, `:210` @c307f40).

Run state is computed by `shouldRun` from activity annotations
(`pkg/reconciler/agent_reconciler.go:157` @c307f40); the StatefulSets are scaled
to 1 when running, 0 when not. **The reconciler only scales up** — scale-down is
the idle checker's job, so a reconcile fired for any other reason can never
hibernate a busy agent (`pkg/reconciler/agent_reconciler.go:153` @c307f40). A
roll-rev annotation set by the api-server is stamped into both pod templates to
force a rolling restart of the pair without a spec/status write
(`pkg/reconciler/agent_reconciler.go:164` @c307f40, `resources.go:64` @c307f40).

Status is split: `Reconciled` ("last render accepted") and readiness ("pods
running") are orthogonal — a running agent stays `Ready=True` even while a later
re-render fails. A running agent gets `publishReadiness` (Ready = AgentPodReady ∧
GatewayPodReady, both required because the agent cannot make credentialed egress
without its gateway); an idle agent gets `publishReconciled` and leaves its
readiness conditions to the idle checker
(`pkg/reconciler/agent_reconciler.go:226` @c307f40, `:243` @c307f40). Status
writes go through `updateAgentStatus` against the status subresource
(`pkg/reconciler/status.go:24` @c307f40).

### Idle checker / hibernation (`idlechecker.go`, `hibernation.go`)

The idle checker is a leader-only goroutine that periodically lists agents and
hibernates idle ones (`pkg/reconciler/idlechecker.go:39` @c307f40). It is
disabled when `idleTimeout <= 0`, and its sweep interval is 1/6 of the timeout
clamped to `[30s, 5m]` (`pkg/reconciler/idlechecker.go:41` @c307f40,
`:62` @c307f40).

The idle/run decision is the *same* `shouldRun` function the reconciler uses, so
the two writers can never disagree about whether an agent is idle; it fails open
(an agent runs when the activity stamp is missing or unparseable, so hibernation
is only ever a positive idle signal) (`pkg/reconciler/hibernation.go:22` @c307f40).
Before scaling down, the checker probes the runtime's `/api/status` endpoint —
the runtime is authoritative about its own idleness, and an unreachable pod
parses as not-busy (fails safe toward allowing hibernation). This busy-guard is
*why* scale-down lives in the checker and not the reconciler
(`pkg/reconciler/idlechecker.go:98` @c307f40, `:121` @c307f40). `hibernate`
scales both StatefulSets of the pair to zero (idempotently, with conflict
retry) and stamps the `Hibernated` reason onto the readiness conditions
(`pkg/reconciler/idlechecker.go:148` @c307f40).

### Warm PVC pool (`warmpool.go`)

`WarmPoolManager` is another leader-only goroutine that maintains a background
buffer of pre-provisioned, already-Bound spare workspace PVCs so a new agent can
claim one instantly instead of waiting on dynamic provisioning
(`pkg/reconciler/warmpool.go:30` @c307f40). It is a no-op when disabled, so a
default deployment incurs zero work (`pkg/reconciler/warmpool.go:47` @c307f40).
The manager only *creates and GCs available spares*; the reconciler only
*claims* them — so the two never contend over the same PVC. A spare carries
`LabelPool` + `LabelPoolAvailable` and no `LabelAgent` (so the orphan sweep skips
it); on claim it gains `LabelAgent` + `LabelMount` and becomes an ordinary agent
PVC (`pkg/reconciler/resources.go:55` @c307f40). Stuck-Pending spares are reaped
after `maxProvisioningTime` (default a generous 30m, so a slow-but-healthy
provision is never churned) (`pkg/reconciler/warmpool.go:80` @c307f40). See
[persistence](../concepts/persistence.md) for the volume model.

### Owned-resource builders (`resources.go`, `envoy.go`, `gateway.go`)

The reconcilers stay thin by delegating rendering to pure `Build*` functions:

- `resources.go` — `BuildAgentStatefulSet`, `BuildAgentService` (headless), the
  shared label vocabulary (`LabelAgent`, `LabelPair`, `LabelRole`, `LabelMount`,
  pool labels), and mount→PVC resolution
  (`pkg/reconciler/resources.go:99` @c307f40, `:462` @c307f40, `:39` @c307f40).
- `gateway.go` — `GatewayName` (= `<agent>-gateway`), `BuildGatewayStatefulSet`
  (replicas track the agent, scaling as a unit), `BuildGatewayService`
  (ClusterIP) (`pkg/reconciler/gateway.go:29` @c307f40, `:33` @c307f40,
  `:126` @c307f40).
- `envoy.go` — the Envoy bootstrap renderer: it expands the granted credential
  Secrets into per-host credential chains and produces the bootstrap ConfigMap
  the gateway's Envoy sidecar consumes (`pkg/reconciler/envoy.go:1099` @c307f40,
  `:1042` @c307f40, `:382` @c307f40). The credential model is documented in
  [security and credentials](../concepts/security-and-credentials.md).

Other reconciler files map cleanly to one concern each: `service_account.go`,
`network_policy.go`, `authorization_policy.go`, `extauthz_service.go`,
`np_gate_init.go` / `iptables_init.go` (init containers), `pod_overrides.go`,
`pod_termination.go`, `status.go`, `timing.go` (per-phase reconcile timing,
visible at `LOG_LEVEL=debug`), and `leaf.go`.

### Fork reconcile (`fork.go`, `fork_resources.go`)

`ForkReconciler.Reconcile` mirrors the agent path for the ephemeral fork case
(`pkg/reconciler/fork.go:46` @c307f40). Terminal forks (`Failed`/`Completed`) are
skipped (`pkg/reconciler/fork.go:51` @c307f40). It resolves the parent Agent via
an `AgentResolver` backed by the shared informer cache
(`pkg/reconciler/agent.go:27` @c307f40, `crd.go:62` @c307f40), then renders
fork-scoped copies of the bootstrap CM, leaf cert, ServiceAccount, AuthZ
policies, and egress NetworkPolicy. Two security distinctions matter: the fork's
credential Secrets are scoped to the *foreign* identity (the parent owner's
secrets must not appear), and the fork gets its **own** SA so a compromised fork
can only reach the narrow `/api/agents/<parent>/mcp` path the per-fork harness
policy admits — not the parent's full surface
(`pkg/reconciler/fork.go:75` @c307f40, `:102` @c307f40, `:113` @c307f40). The
fork agent runs as a Job + paired gateway Pod rather than StatefulSets
(`pkg/reconciler/fork.go:139` @c307f40, `fork_resources.go`).

## Configuration (`pkg/config/`)

`config.LoadFromEnv` (`pkg/config/config.go`) builds the `Config` struct from
env vars threaded in by the Helm chart: the agent / release namespaces, the
`AgentBase` platform policy applied verbatim to every rendered pod, agent
template defaults, warm-pool settings, Envoy image/port and the cert-manager
MITM issuer, and the Istio trust domain used to render SPIFFE principals
(`pkg/config/config.go:15` @c307f40, `:29` @c307f40, `:67` @c307f40). Security
context and scheduling are chart-only by design and cannot be set per-agent
(`api/v1/agent_types.go:7` @c307f40).

## Public surface

- **CRDs** `Agent` and `Fork` under `agent-platform.ai/v1` — the contract with
  the api-server (`api/v1/groupversion_info.go:27` @c307f40).
- **Exported reconciler API** consumed by `main.go`: `NewAgentReconciler`,
  `NewForkReconciler`, `NewIdleChecker`, `NewWarmPoolManager`, `NewAgentLister`,
  `NewAgentResolver`, the `AgentsGVR`/`ForksGVR` GVRs, the `LabelAgent`
  selector, `GatewayName`, and the `Build*` resource renderers.
- **Status subresource** on each CR — the controller's only write surface; the
  api-server reads `Ready` to route (`api/v1/agent_types.go:78` @c307f40).

## Where to look

| Concern | File |
| --- | --- |
| Entrypoint, leader election, informers, queues, workers | `main.go` |
| CRD types & conditions | `api/v1/agent_types.go`, `api/v1/fork_types.go` |
| Agent reconcile loop | `pkg/reconciler/agent_reconciler.go` |
| Idle / hibernation | `pkg/reconciler/idlechecker.go`, `hibernation.go` |
| Warm PVC pool | `pkg/reconciler/warmpool.go` |
| Owned-resource builders | `resources.go`, `gateway.go`, `envoy.go` |
| Fork reconcile | `pkg/reconciler/fork.go`, `fork_resources.go` |
| Config from Helm env | `pkg/config/config.go` |
| Status writes & conditions | `pkg/reconciler/status.go` |

## Related pages

- [dam (source overview)](./dam.md)
- [agent (entity)](../entities/agent.md)
- [platform topology](../concepts/platform-topology.md)
- [agent lifecycle](../concepts/agent-lifecycle.md)
- [persistence](../concepts/persistence.md)
