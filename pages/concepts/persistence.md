---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [docs/architecture/persistence.md]
updated: 2026-06-19
---

# Persistence & the Substrate Split

DAM keeps durable state on **three substrates**, split cleanly between the
platform and the agent so that no two writers ever own the same surface
(`docs/architecture/persistence.md:7 @c307f40`). The split is what lets an Agent
hibernate to zero pods and wake later with its workspace intact, while the
platform's view of that Agent stays queryable the whole time.

| Substrate | Owner | Holds | Sole writer |
|---|---|---|---|
| **Postgres** | platform | application state queryable with no pod running | api-server |
| **Custom resources** (Agent/Fork CR `spec`/`status`) | platform | resource state the controller reconciles | api-server (`spec`) / controller (`status`) |
| **Per-Agent PVC** | agent | workspace + `$HOME` mounted into the pod | the agent process |

The agent's **only** durable surface is the PVC; everything the platform knows
*about* the agent is mirrored onto Postgres or the Agent CR by the api-server or
controller — never by the agent itself
(`docs/architecture/persistence.md:20 @c307f40`). Conversely the agent has no
direct access to Postgres or to the custom resources that describe it
(`docs/architecture/persistence.md:16 @c307f40`).

## The three substrates

### Postgres — Application State

Postgres carries application state the api-server owns end-to-end: anything that
has to be queryable when **no agent pod is running**, plus any domain resource the
controller does not reconcile (`docs/architecture/persistence.md:55 @c307f40`).
The api-server is the sole writer; the **controller never reads from or writes to
Postgres** — its bookkeeping lives on the CR `status` subresource instead
(`docs/architecture/persistence.md:11 @c307f40`,
`docs/architecture/persistence.md:62 @c307f40`).

What lives here (`docs/architecture/persistence.md:57-61 @c307f40`):

- **channel routing** — bindings between external chat surfaces and the
  Agent/session they map to.
- **identity and auth** — channel-identity↔user links, the auth allow-list, and
  API keys for headless CLI use.
- **skills catalog** — connected sources, per-Agent install records, publish
  history.
- **activity log + agent mirror** — append-only `activity_events`, per-sub
  `actor_roles`, and the K8s↔Postgres ownership mirror (`agents`), with
  pseudonymized `actor_sub` / `owner_sub` columns at the write boundary.
- **schedules** — RRULE, quiet hours, task payload, session mode, and firing
  bookkeeping (`schedules`); the api-server's schedule loop fires them, the
  controller plays no part.

Note that **session metadata is *not* in Postgres** — it is agent-owned and lives
on the PVC (`docs/architecture/persistence.md:11 @c307f40`; see below). The
authoritative schema and migrations live in
[`packages/db/`](../sources/db.md), running automatically on api-server startup
(`docs/architecture/persistence.md:63 @c307f40`).

### Custom resources — Infra State

Resources the controller reconciles are Kubernetes CRDs under the
`agent-platform.ai/v1` API group, each with a status subresource
(`docs/architecture/persistence.md:67 @c307f40`):

| Kind | What it declares | `spec` writer | `status` writer |
|---|---|---|---|
| `Agent` | image, mount declarations, env, secret refs, image-pull secret ref, granted secret/connection IDs — the sole resource per Agent | api-server | controller |
| `Fork` | parent Agent ref + overrides for a forked run | api-server | controller |

The former template/instance pair was collapsed into the single `Agent` kind
(`docs/architecture/persistence.md:71 @c307f40`).

**The single-writer split is structural, not conventional** — the status
subresource makes it so the controller and api-server can never write the same
surface (`docs/architecture/persistence.md:20 @c307f40`):

- **`spec`** is user intent, written exclusively by the api-server and validated
  by the K8s API server at admission; the controller consumes typed objects
  without re-validating shape (`docs/architecture/persistence.md:76 @c307f40`).
- **`status`** is observed state, written exclusively by the controller.
  Conditions are the source of truth (`Ready`, `AgentPodReady`,
  `GatewayPodReady`, `Reconciled`); `Ready` is the agent-and-gateway pod
  intersection and the api-server's sole routing signal
  (`docs/architecture/persistence.md:77 @c307f40`).

There is **no stored desired state** — the former `desiredState` latch is gone.
Wake is a one-off activity poke, the controller hibernates on idleness, and
running-vs-hibernated is recorded as observed status; see
[agent-lifecycle](../concepts/agent-lifecycle.md)
(`docs/architecture/persistence.md:79 @c307f40`).

High-frequency, out-of-band signals live on **annotations** rather than `status`,
so they are independently patchable without a spec or status write: the
last-activity timestamp and active-session marker that drive hibernation, plus
the roll trigger the api-server bumps to force a rolling restart
(`docs/architecture/persistence.md:81 @c307f40`).

### Per-Agent PVC — Workspace Volume

Each `agent` reconciles into a StatefulSet whose `volumeClaimTemplates` derive
from the Agent's declared mounts: a mount marked `persist: true` becomes a PVC, a
non-persisted mount becomes an `emptyDir` that dies with the pod. PVCs are
`ReadWriteMany` so the original owner and a foreign user running a fork can mount
the same volume concurrently (`docs/architecture/persistence.md:90 @c307f40`).

The default Claude Code template persists the **workspace** and **`$HOME`**
(`docs/architecture/persistence.md:92-95 @c307f40`):

- **workspace** — git checkouts, tool caches (`node_modules`, `.venv`, mise), and
  agent-produced artifacts.
- **`$HOME`** — agent memory, skills, MCP server caches, and the harness's
  on-disk session store (the cold-start source for `session/load` after a pod
  restart). The agent-runtime's `.platform/` directory lives here too, holding the
  **session-metadata state file** — the platform's sole source of truth for
  per-session mode, type, `scheduleId`, `threadTs`, and `createdAt`, surfaced over
  ACP `_meta.platform` — alongside trigger-binding and runtime-channel state files.
- **`.import-staging-*/`** — transient extraction dirs for the bundled
  file-import path; orphans from crashed imports are reclaimed by an agent-runtime
  boot sweeper (`docs/architecture/persistence.md:96 @c307f40`).

PVCs **survive hibernation**: when a StatefulSet scales to zero replicas the
volume detaches but is retained. The controller **explicitly deletes** PVCs on
Agent deletion — overriding the StatefulSet default of retaining them, because
Agent deletion is intentional (`docs/architecture/persistence.md:98 @c307f40`).

What does **not** survive hibernation: anything written to the container's
ephemeral filesystem outside the persisted mounts — OS-level changes, packages
installed at runtime, files in `/tmp`. Tools and dependencies must be baked into
the image at build time (`docs/architecture/persistence.md:100 @c307f40`).

## Choosing Postgres vs. the K8s API

A new resource belongs on a **CRD iff the controller reconciles it**. If only the
api-server reads and writes it, it belongs in **Postgres**. The spec/status
single-writer split exists purely to coordinate api-server and controller; without
a controller reader it has no purpose, and putting api-server-only state on the
K8s API is using it as a generic key-value store
(`docs/architecture/persistence.md:18 @c307f40`).

This is why two domain resources are deliberately **not** CRDs
(`docs/architecture/persistence.md:83-86 @c307f40`):

- **Templates** stay ConfigMaps — chart-rendered, read-only blueprints copied into
  an Agent at create time, never reconciled, loaded by the api-server at boot.
- **Schedules** live in Postgres — only the api-server reads and writes them.

## Warm PVC pool — Spare / Claim

First-start provisioning of a workspace PVC is slow on production storage — tens
of seconds to minutes — because the volume is allocated on demand when the first
pod mounts it. To hide that latency the controller can keep a **warm pool**: a
background buffer of pre-provisioned, already-bound spare PVCs organized into
per-workspace-size pools, that a new Agent claims instantly. The buffer refills in
the background, is operator-tunable, and is **disabled by default**
(`docs/architecture/persistence.md:104 @c307f40`).

- **Immediate binding.** A spare only helps if it holds real storage while idle,
  so pool PVCs use an **immediate-binding** StorageClass — the agents' own class
  defers allocation until mount, which would leave a pre-created spare empty
  (`docs/architecture/persistence.md:106 @c307f40`).
- **Claim at create.** For each persisted mount whose size matches a configured
  pool, the controller claims one spare and mounts it **by name** rather than
  through the StatefulSet's `volumeClaimTemplate`. A mount with no matching pool,
  or an exhausted pool, **falls back to on-demand provisioning** so Agent creation
  never blocks (`docs/architecture/persistence.md:106 @c307f40`).
- **Tracked by labels.** An unclaimed spare carries a pool label but **no
  owning-agent label**, so the orphan-PVC sweep — which acts only on agent-labeled
  PVCs — leaves it untouched. On claim the controller stamps the agent label and
  removes the available marker in **one atomic update**; from then on the volume is
  an ordinary per-Agent PVC, reclaimed on Agent deletion and reattached on wake
  (`docs/architecture/persistence.md:108 @c307f40`).
- **Survives hibernate/wake.** The claim decision is made once at create and
  reconstructed from the live StatefulSet on every later reconcile, without ever
  re-rendering the pod's volumes — even if the claimed volume is deleted
  out-of-band, the agent keeps referencing it by name rather than degrading to a
  mount with no backing volume (`docs/architecture/persistence.md:108 @c307f40`).

## What survives each lifecycle event

(`docs/architecture/persistence.md:112-120 @c307f40`)

| Event | Postgres | Agent CR (spec/status) | PVC |
|---|---|---|---|
| Pod restart | survives | survives | survives |
| Hibernate (replicas → 0) | survives | survives | survives |
| Wake (replicas → 1) | survives | survives | survives |
| api-server restart | survives | survives | survives |
| Controller restart | survives | survives | survives |
| **Agent delete** | agent row marked deleted by api-server | CR removed | PVCs removed by controller |
| Schedule delete | schedule row removed | n/a | n/a |

Edge cases the table doesn't fully capture:

- **Schedules** are independent Postgres rows and survive Agent deletion as
  orphans unless the deletion path explicitly cascades
  (`docs/architecture/persistence.md:122 @c307f40`).
- **Sessions** are agent-owned files on the PVC, not Postgres rows — they follow
  the PVC column, not Postgres (`docs/architecture/persistence.md:122 @c307f40`).
- **Unclaimed warm-pool spares** are tied to no Agent and so are absent from the
  table — the pool manager reclaims them when it trims a pool below inventory or
  the size pool is removed, never via Agent deletion. Once claimed, a spare
  follows the PVC column (`docs/architecture/persistence.md:124 @c307f40`).
- An **agent-scoped image-pull Secret** (for private custom images) follows the
  Agent itself: the api-server writes it at create and removes it on delete via a
  cleanup hook, with a label-scoped orphan sweep as backstop — the opposite of the
  reusable owner-scoped credential Secrets the gateway injects for egress
  (`docs/architecture/persistence.md:126 @c307f40`).

## Security boundary

The PVC is a **shared mutable surface across every session, trigger, fork, and
channel-driven prompt that runs on the same Agent**. Anything one turn writes into
the workspace — model output, tool output, fetched files — is plain context for
the next turn, so workspace contents must be treated as **adversarial input**: a
scheduled job can plant a file that prompt-injects a later user-driven session
(`docs/architecture/persistence.md:130 @c307f40`).

The platform does **not** sandbox writes within the workspace. Mitigations live
elsewhere: NetworkPolicy restricts which upstreams the agent can reach (the agent
pod can only dial its paired gateway pod), the gateway pod gates credentialed
egress, and forks let you run with a narrowed credential set without polluting the
parent's workspace (`docs/architecture/persistence.md:132 @c307f40`).

## See also

- [Platform topology](../concepts/platform-topology.md) — how api-server,
  controller, and agent-runtime pods are wired.
- [Agent lifecycle](../concepts/agent-lifecycle.md) — wake, hibernation, and the
  activity-poke model that drives replica scaling.
- [db](../sources/db.md) — the Postgres schema and migration workflow.
- [controller](../sources/controller.md) — the reconciler that owns CR `status`,
  PVC provisioning, and the warm pool.
- [Agent](../entities/agent.md) — the single CR that declares mounts, env, and
  secret grants.
