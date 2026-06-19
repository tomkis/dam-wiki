---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [docs/architecture/agent-lifecycle.md, packages/controller/pkg/reconciler/idlechecker.go, packages/controller/pkg/reconciler/hibernation.go, packages/controller/pkg/reconciler/fork_resources.go]
updated: 2026-06-19
---

# Agent Lifecycle

An **Agent** is DAM's durable, owned, runnable resource: the unit a user creates, configures, and deletes. Its defining trait is that it scales between **zero and one** pod replica over its lifetime, hibernating when quiet and waking when work arrives — so an idle Agent costs nothing but a PVC, yet stays addressable. This page traces the states (create → wake → trigger → hibernate → delete), the transitions between them, and the actors that drive each one. For where the Agent sits in the cluster, see [../concepts/platform-topology.md](../concepts/platform-topology.md); the Agent resource itself is detailed on [../entities/agent.md](../entities/agent.md).

## The three drivers

The lifecycle is not a single state machine but the interplay of three independent actors `docs/architecture/agent-lifecycle.md:7-12 @c307f40`:

- **The UI (management surface).** The *only* writer of Agent spec. Create, configure, hibernate, and delete all flow through tRPC on the api-server's public port. Sessions may be driven from the UI or from a connected channel, but channels never touch management endpoints — they dial the ACP relay only `docs/architecture/agent-lifecycle.md:9 @c307f40`. See [../concepts/channels.md](../concepts/channels.md).
- **The api-server's scheduler.** Fires triggers on RRULE/cron occurrences, delivers them durably over the runtime outbox, and pokes a hibernated Agent awake so a fire lands even on a sleeping Agent `docs/architecture/agent-lifecycle.md:10 @c307f40`.
- **The controller's idle checker.** Hibernates running Agents that go quiet `docs/architecture/agent-lifecycle.md:11 @c307f40`.

## No stored desired state

The most important design choice: **there is no "desired state" field.** Running-vs-hibernated is *observed status* the controller derives from activity, not a flag a user sets `docs/architecture/agent-lifecycle.md:57 @c307f40`. The api-server stamps two activity annotations on the Agent — `agent-platform.ai/active-session` and `agent-platform.ai/last-activity` `packages/controller/pkg/reconciler/hibernation.go:8-9 @c307f40` — and a single shared decision function `shouldRun` reads them: the reconciler scales *up* when it returns true, and the idle checker treats false as a scale-*down* candidate, so the two can never disagree `packages/controller/pkg/reconciler/hibernation.go:12-22 @c307f40`.

`shouldRun` **fails open**: an Agent runs when auto-hibernation is disabled or when the last-activity stamp is missing or unparseable, so hibernation only ever results from a *positive* idle signal, never from absent data `packages/controller/pkg/reconciler/hibernation.go:18-25 @c307f40`.

## Create

The api-server writes a new Agent custom resource whose spec carries the image / mount declarations (copied from a Template at create time, if any), env, secret refs, and allowed users `docs/architecture/agent-lifecycle.md:57 @c307f40`. The controller then reconciles a paired set of owned resources: two StatefulSets (the agent and its paired gateway), two headless Services (the agent's ACP service and the gateway's proxy DNS), an agent-egress NetworkPolicy, and a per-Agent Envoy bootstrap ConfigMap plus a leaf TLS Certificate `docs/architecture/agent-lifecycle.md:57 @c307f40`. The StatefulSet starts at `replicas=0` — creation does not boot a pod `docs/architecture/agent-lifecycle.md:28 @c307f40`.

**Pull credentials.** When the create request carries a private-registry credential, the api-server writes an agent-scoped `dockerconfigjson` pull Secret *before* the Agent CR and rolls it back if that write fails. The controller lists that Secret first on the pod's `imagePullSecrets`, ahead of any install-wide default; the kubelet consumes it so the credential never enters the pod `docs/architecture/agent-lifecycle.md:59 @c307f40`.

**Env composition.** Pod env at start is the composition of three layers, last occurrence wins, with `PORT` server-enforced: platform envs (proxy + auth wiring rendered by the controller), `credentialEnvVars` (derived from mounted credential Secrets), and `agent.env` (the single env list on the spec, of which the api-server is sole writer) `docs/architecture/agent-lifecycle.md:63-67 @c307f40`. Template env contributes at *create time only* — `assembleSpecFromTemplate` copies it into `agent.env`; the controller never re-reads the Template at pod start, so editing a Template never re-flows into a running Agent `docs/architecture/agent-lifecycle.md:69 @c307f40`.

**The workspace PVC** is provisioned on first wake and survives later hibernations — unless a warm-pool spare matching the mount size is claimed at create time, skipping the first-start provisioning wait (invisibly: a claimed spare becomes an ordinary per-Agent PVC) `docs/architecture/agent-lifecycle.md:61 @c307f40`. See [../concepts/persistence.md](../concepts/persistence.md).

## Wake

Every caller that sends work to a pod — the ACP relay, channel adapters, skills management — routes through a single **reachability primitive** in the api-server. Its contract: the controller-published `Ready` condition is the authoritative answer to "can I call this pod?" The primitive pokes activity by bumping `last-activity` (the reconciler scales up any Agent with recent activity), single-flights concurrent waits per Agent, and re-bumps the annotation on every successful call, so any caller implicitly keeps the pod warm `docs/architecture/agent-lifecycle.md:75 @c307f40`.

Three paths trigger a wake `docs/architecture/agent-lifecycle.md:78-84 @c307f40`:

- **Connect-driven** — the api-server is about to forward an ACP frame to a hibernated Agent and ensures readiness before the relay completes. The frame originates from a UI tab attaching or a channel worker routing an inbound message to its bound session.
- **Schedule-driven** — a fire commits a `trigger` event to the outbox, then pokes the Agent awake *without* waiting for readiness; the boot-time `hello` catch-up delivers the event once the pod is `Ready` (see [Trigger fire](#trigger-fire)).
- **Skills-management-driven** — install / uninstall / scan / publish route through the same primitive before reaching the agent.

Wake is **bounded**: the primitive polls pod readiness with backoff and gives up after two minutes, surfacing a loud error (WS close code, channel log, or skills call error). The schedule-driven poke is the exception — it doesn't wait, so there is no bounded wait to fail `docs/architecture/agent-lifecycle.md:84 @c307f40`.

## Trigger fire

Schedules are Postgres rows owned by the api-server, each armed as a delayed job on a Redis-backed queue — one pending job per schedule, re-armed after every fire. The next occurrence is computed from the schedule's cron/RRULE in its timezone, skipping any occurrence inside an enabled quiet-hours window; suppressed fires are *dropped, not deferred*, and a schedule whose every occurrence is quiet is rejected at save time `docs/architecture/agent-lifecycle.md:88 @c307f40`.

When a fire is due `docs/architecture/agent-lifecycle.md:90-95 @c307f40`:

1. The api-server inserts a `trigger` event into the runtime outbox in the same transaction that bumps the Agent's version, then signals the delivery worker. **The fire is durable from this point**; the schedule re-arms for its next occurrence regardless of delivery outcome.
2. The api-server pokes the activity annotation so the reconciler scales a hibernated Agent up. The poke never waits on readiness; a poke that errors is recorded as a failed fire, but the committed event still delivers if the Agent comes `Ready` within its TTL.
3. The delivery worker pushes the event over `applyState` — only once the Agent is `Ready`. A waking Agent picks pending events up on its boot-time `hello` catch-up. Every event carries a TTL, so an Agent down through several occurrences doesn't replay a backlog of stale fires.
4. agent-runtime's trigger handler opens an ACP session against the harness over an in-process channel and submits the task as a prompt. The event settles once the prompt is submitted; the turn runs asynchronously.

The agent keeps a last-fire timestamp per schedule on the PVC and skips any fire at or before it, so a redelivered or superseded fire never runs twice `docs/architecture/agent-lifecycle.md:97 @c307f40`. Outbox versioning, sweep, and expiry are owned by [../concepts/persistence.md](../concepts/persistence.md) (event lifecycle).

### Per-schedule session continuity

The session model differs by schedule mode `docs/architecture/agent-lifecycle.md:101-106 @c307f40`:

- **Fresh schedule** — every fire creates a new session via `session/new`; the schedule accumulates a browseable list of sessions. Fresh fires open their own sessions and may run concurrently.
- **Continuous schedule** — the first fire creates a session; every subsequent fire calls `session/resume` against the same session id. One schedule, one session, history retained across fires; fires serialize naturally as prompts queue at the runtime.

The schedule↔session link is **agent-owned**: schedule sessions are typed (`schedule_cron`) through ACP session metadata, and the continuous binding is a per-schedule entry in a PVC state file. Resetting a continuous schedule rides the same outbox rail as fires — a `schedule-reset` event clears the binding on delivery. Unlike a fire, a reset does *not* poke the Agent awake, so a reset against an Agent that stays hibernated past the event's TTL expires undelivered and the next fire resumes the old session `docs/architecture/agent-lifecycle.md:106 @c307f40`.

## Sessions inside the pod

The harness child process runs for the *pod's* lifetime, not per-connection. Multiple ACP channels (UI tab WebSockets, the Slack worker, the in-process trigger handler) attach to the same runtime concurrently and engage with sessions implicitly via the `sessionId` on each frame `docs/architecture/agent-lifecycle.md:110 @c307f40`. A session is an append-only in-memory log (≤2 MB soft cap) with a per-channel cursor; `session/load` is served from the log on cache hit and falls through to the on-disk store on cold start `docs/architecture/agent-lifecycle.md:112 @c307f40`.

Sessions are short-lived relative to the pod: when one goes idle (no engaged channel, no active or queued prompt, no pending agent-initiated request) the runtime sends `session/close` to the harness, reaping the per-session subprocess; the next attach respawns it `docs/architecture/agent-lifecycle.md:116 @c307f40`. Terminal-mode and SSH sessions follow distinct models (single-viewer PTY; per-connection `sshd`) and, like chat relays, pin the Agent `active-session` so it will not hibernate while connected `docs/architecture/agent-lifecycle.md:118-120 @c307f40`.

## Hibernate

The controller's idle checker periodically scans running Agents `packages/controller/pkg/reconciler/idlechecker.go:37-39 @c307f40`. For each it probes agent-runtime's `/api/status` over the cluster network `packages/controller/pkg/reconciler/idlechecker.go:116-122 @c307f40`. **The runtime is authoritative about its own idleness**: it reports a single `idle` flag (false while a prompt turn runs, prompts are queued, an agent-initiated request awaits a client, or a terminal is open — connected viewers don't count), and the controller derives nothing on its own `docs/architecture/agent-lifecycle.md:148 @c307f40` `packages/controller/pkg/reconciler/idlechecker.go:134 @c307f40`. A probe that fails to report parses as busy, failing safe. If the runtime reports idle for long enough, the checker scales the StatefulSets to zero `docs/architecture/agent-lifecycle.md:148 @c307f40`.

The check interval is derived from the timeout (1/6 of it, clamped to [30s, 5m]), and the whole feature disables when `idleTimeout <= 0` `packages/controller/pkg/reconciler/idlechecker.go:40-47 @c307f40`.

On hibernation the pod terminates but the **PVC, Secret, Service, and NetworkPolicy persist** `docs/architecture/agent-lifecycle.md:150 @c307f40`. Workspace state survives — git checkout, `node_modules`, `.venv`, mise cache, and `$HOME` are all on the PVC and rejoin on the next wake. Anything written to the container's *ephemeral* filesystem (OS changes, tools outside `$HOME`) is lost — a deliberate constraint of the lifetime model `docs/architecture/agent-lifecycle.md:150 @c307f40`.

## Delete

The api-server deletes the Agent custom resource; the controller's reconciler tears down the owned StatefulSet, Service, NetworkPolicy, and Secret `docs/architecture/agent-lifecycle.md:152-154 @c307f40`. Sessions are agent-owned PVC files and disappear with the volume. The controller reclaims the workspace PVCs **explicitly** — StatefulSet `volumeClaimTemplate` PVCs are not cascade-deleted by Kubernetes. In-flight per-turn forks are owner-refed to the Agent CR, so Kubernetes garbage-collects them automatically `docs/architecture/agent-lifecycle.md:154 @c307f40`.

The api-server owns none of the storage teardown: it never touches PVCs, and only deletes the Secrets it wrote (per-channel credential Secrets and, via a cleanup hook, the agent-scoped pull Secret, with a label-scoped orphan sweep as backstop) `docs/architecture/agent-lifecycle.md:154 @c307f40`. **Schedules are independent Postgres rows and survive Agent deletion as orphans** unless the deletion path explicitly cascades `docs/architecture/agent-lifecycle.md:156 @c307f40`.

## Forks

A **fork** is the third durable concept in the bounded context, alongside Template and Agent. An `agent-fork` ConfigMap runs a derivative of an existing Agent with credential and env overrides `docs/architecture/agent-lifecycle.md:158-160 @c307f40`. Crucially, forks reconcile to a **Kubernetes Job** rather than a StatefulSet `packages/controller/pkg/reconciler/fork_resources.go @c307f40` — they **run to completion** and are never woken, hibernated, or kept warm, sidestepping the whole zero-to-one lifecycle above `docs/architecture/agent-lifecycle.md:160 @c307f40`. This already matches the run-to-completion shape DAM's *target* lifetime model intends for Agents (single-use Jobs per turn) `docs/architecture/agent-lifecycle.md:144 @c307f40`. A fork's Job inherits the parent Agent's image-pull Secret, so the kubelet pulls a private parent image without the fork ever seeing the credential `docs/architecture/agent-lifecycle.md:160 @c307f40`.

---

*See also:* [../entities/agent.md](../entities/agent.md) · [../sources/controller.md](../sources/controller.md) · [../concepts/platform-topology.md](../concepts/platform-topology.md) · [../concepts/persistence.md](../concepts/persistence.md) · [../concepts/channels.md](../concepts/channels.md)
