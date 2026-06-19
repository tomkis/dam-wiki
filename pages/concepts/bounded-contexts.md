---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [docs/ubiquitous-language.md]
updated: 2026-06-19
---

# Bounded Contexts & Ubiquitous Language

DAM is organised as a domain-driven design (DDD) codebase: the system is carved
into **bounded contexts**, each owning a self-consistent **ubiquitous
language** — a vocabulary where every term has one precise meaning *within that
context*. Terms are scoped to their context; the same word can mean different
things in two contexts, and that is by design, not an accident to be
reconciled (`docs/ubiquitous-language.md:3 @c307f40`).

This page is the glossary and orientation hub. It distils
`docs/ubiquitous-language.md` — the authoritative term registry — into a
per-context summary. For the *mechanics* behind a context, follow the
cross-links into the deeper concept and entity pages.

## The DDD discipline in DAM

- **One vocabulary per context.** Every term below is scoped to its bounded
  context, with one deliberate exception: the **Substrate** vocabulary is
  cross-cutting and shared by every context
  (`docs/ubiquitous-language.md:3 @c307f40`).
- **Same words, different responsibilities.** The clearest expression of the
  discipline is the **Skills** domain, which is split into two contexts —
  *api-server side* and *agent-runtime side* — that reuse the same nouns
  (`Skill`, `Skill Source`, `Install`, `Publish`, `Scan`) for different
  responsibilities (`docs/ubiquitous-language.md:53-78 @c307f40`). See
  [Two-sided Skills](#two-sided-skills-the-canonical-split) below.
- **Names carry contracts.** Where a name is *not* a stable contract, the
  registry says so explicitly — e.g. a Workspace Volume is identified by its
  owning-Agent + mount label, **not** by a reconstructed name
  (`docs/ubiquitous-language.md:13 @c307f40`).
- **Disambiguate against infrastructure nouns.** Domain verbs are kept distinct
  from their Kubernetes namesakes — e.g. *Claim (verb)* is distinct from the
  Kubernetes noun *PersistentVolumeClaim* (`docs/ubiquitous-language.md:16
  @c307f40`).

The contexts, at a glance:

| Bounded context | Owns | Deep page |
|-----------------|------|-----------|
| Substrate (cross-cutting) | Where state lives: Infra State vs Application State, volumes, warm pool | [Persistence](../concepts/persistence.md) |
| Agents | The durable runnable resource, its lifecycle, sessions, schedules | [Agent lifecycle](../concepts/agent-lifecycle.md), [Agent entity](../entities/agent.md) |
| Channels | External pathways (Slack) connecting users to Agents | [Channels](../concepts/channels.md) |
| Forks | Ephemeral per-turn impersonation of a foreign Slack user | [Channels](../concepts/channels.md) |
| Skills — api-server side | Catalog/orchestration: sources, installs, publishes | [Skills](../concepts/skills.md) |
| Skills — agent-runtime side | Pod-side files: materialise, scan, publish | [Skills](../concepts/skills.md) |
| Approvals | Gating egress + harness tool calls on a user verdict | [Security & credentials](../concepts/security-and-credentials.md) |
| Egress Rules | Persistent allow/deny decisions on outbound traffic | [Security & credentials](../concepts/security-and-credentials.md) |
| Secrets | User-owned credentials injected at the Envoy sidecar | [Security & credentials](../concepts/security-and-credentials.md) |
| Connections (proposed) | Unified registry of auth + contributions (in design) | — |
| Terms | Terms-of-use acceptance gate | — |
| Usage Tracking | Append-only activity events + operator metrics | — |
| Platform CLI | The `dam` terminal client | — |

---

## Substrate (cross-cutting)

The persistence vocabulary shared by every context. The split it encodes — what
the Controller reconciles vs what only the API Server touches — is the backbone
of the whole system; see [Persistence](../concepts/persistence.md).

| Term | Definition |
|------|-----------|
| Infra State | State the Controller reconciles into running infrastructure; a ConfigMap with `spec.yaml` (api-server writer) and `status.yaml` (controller writer) (`docs/ubiquitous-language.md:11 @c307f40`). |
| Application State | State only the API Server reads/writes; the Controller never touches it. Stored in PostgreSQL (`docs/ubiquitous-language.md:12 @c307f40`). |
| Workspace Volume | The persistent volume holding an Agent's workspace and `$HOME`; identified by owning-Agent + mount label, not a reconstructed name (`docs/ubiquitous-language.md:13 @c307f40`). |
| Warm Pool | Controller-managed, leader-only buffer of pre-provisioned, bound Spare volumes (per-size pools) claimed at Agent create to skip first-start latency; disabled by default (`docs/ubiquitous-language.md:14 @c307f40`). |
| Spare | An unclaimed Workspace Volume in the Warm Pool — provisioned, bound, carrying a pool label and available marker but deliberately no owning-Agent label (`docs/ubiquitous-language.md:15 @c307f40`). |
| Claim (verb) | To assign a Spare to a new Agent via one atomic relabel; distinct from the Kubernetes *PersistentVolumeClaim* noun (`docs/ubiquitous-language.md:16 @c307f40`). |

## Agents

The merged `Agent` carries definition, runtime state, and lifecycle —
`Instance` is retired (`docs/ubiquitous-language.md:20 @c307f40`). See the
[Agent entity](../entities/agent.md) and
[Agent lifecycle](../concepts/agent-lifecycle.md).

| Term | Definition |
|------|-----------|
| Template | Read-only catalog blueprint for the base image, mounts, env, resources of a new agent (`docs/ubiquitous-language.md:24 @c307f40`). |
| Agent | The durable, owned, runnable resource (definition + runtime state + lifecycle): one ConfigMap with `spec.yaml` (api-server) and `status.yaml` (controller) (`docs/ubiquitous-language.md:25 @c307f40`). |
| Sandbox | User-facing name for an Agent in the redesigned UI; "Agent" stays the domain/code term (`docs/ubiquitous-language.md:26 @c307f40`). |
| Session | One conversation with the agent harness, with its own lifecycle and metadata (`docs/ubiquitous-language.md:27 @c307f40`). |
| Schedule | A time-triggered task attached to an Agent — cron-based or heartbeat (`docs/ubiquitous-language.md:28 @c307f40`). |
| Desired State | The target lifecycle state: running or hibernated (`docs/ubiquitous-language.md:29 @c307f40`). |
| Wake | Transitioning an Agent from hibernated to running (`docs/ubiquitous-language.md:30 @c307f40`). |
| Heartbeat | A recurring schedule defined by interval, internally converted to cron (`docs/ubiquitous-language.md:31 @c307f40`). |
| Reserved ID Prefix (`agent-`) | Prefix the controller mints onto every Agent ID; forbidden in user-chosen names; the CLI's ID-vs-name split signal (`docs/ubiquitous-language.md:32 @c307f40`). |
| Keycloak User Directory | Infrastructure port resolving between user emails and Keycloak `sub` ids, backed by the Keycloak admin API (`docs/ubiquitous-language.md:33 @c307f40`). |

## Channels

External communication pathways. See [Channels](../concepts/channels.md).

| Term | Definition |
|------|-----------|
| Channel | An external communication pathway connecting users to an Agent (e.g. Slack) (`docs/ubiquitous-language.md:39 @c307f40`). |
| Channel Binding | The 1:1 linkage between a Slack channel and an Agent (at most one Agent globally per channel); released on Agent delete or Slack disconnect (`docs/ubiquitous-language.md:40 @c307f40`). |
| Channel Worker | A long-running process that bridges an external service to an Agent (`docs/ubiquitous-language.md:41 @c307f40`). |
| Thread | A Slack thread identified by `thread_ts`; maps 1:1 to at most one Session per Agent (`docs/ubiquitous-language.md:42 @c307f40`). |
| Foreign Replier | A linked Slack user in `allowedUsers` whose identity differs from the owner; triggers a Fork for the turn (`docs/ubiquitous-language.md:43 @c307f40`). |

## Forks

Ephemeral, per-turn impersonation. A Fork is Job-shaped and run-to-completion —
distinct from the Agent's StatefulSet shape (`docs/ubiquitous-language.md:49
@c307f40`). See [Channels](../concepts/channels.md).

| Term | Definition |
|------|-----------|
| Fork | A per-turn execution environment derived from an Agent that impersonates a foreign user for one Slack turn (`docs/ubiquitous-language.md:49 @c307f40`). |
| Foreign Sub | The Keycloak `sub` of a Slack replier who is not the Agent owner (`docs/ubiquitous-language.md:50 @c307f40`). |
| Fork Phase | The Fork's lifecycle state: Pending, Ready, Failed, or Completed (`docs/ubiquitous-language.md:51 @c307f40`). |

---

## Two-sided Skills: the canonical split

The Skills domain is split into **two bounded contexts** that share the same
vocabulary but answer to different responsibilities — the textbook DDD move and
the reason this page exists. When you read "Skill Source" or "Publish", you must
know *which side* you are on. See [Skills](../concepts/skills.md).

- **api-server side** owns *which sources are connected, which skills are
  installed where, and what was published from which Agent*. It never touches
  files on a pod directly; every concept is Application State in Postgres or
  api-server config (`docs/ubiquitous-language.md:53-55 @c307f40`).
- **agent-runtime side** owns *what files are where on this pod and how to mutate
  them*. It never reasons about source catalogs or drift
  (`docs/ubiquitous-language.md:64-66 @c307f40`).

Same words, contrasted across the two sides:

| Shared word | api-server side | agent-runtime side |
|-------------|-----------------|--------------------|
| Skill Source | A connected, id-addressable source of one of three kinds — user / system / template (`docs/ubiquitous-language.md:59 @c307f40`). | A git repository URL containing Skills under `skills/*` or top-level `*` (`docs/ubiquitous-language.md:73 @c307f40`). |
| Skill (noun) | Tracked indirectly via Refs/records, not as a file unit. | A directory with `SKILL.md` (`name`/`description` frontmatter) — the unit of installation (`docs/ubiquitous-language.md:70 @c307f40`). |
| Install | A *record* (Installed Skill Ref) that a Scanned Skill is installed at a Version on an Agent (`docs/ubiquitous-language.md:60 @c307f40`). | The *act* of materialising a Skill from a Source at a Version into Skill Paths (`docs/ubiquitous-language.md:76 @c307f40`). |
| Publish | A *record* that a Local Skill was published as a PR to a Skill Source, denormalized to survive source rename/delete (`docs/ubiquitous-language.md:61 @c307f40`). | The *act* of lifting a Local Skill to a GitHub repo as branch + PR via the REST API (`docs/ubiquitous-language.md:77 @c307f40`). |

### Skills — api-server side (catalog/orchestration)

| Term | Definition |
|------|-----------|
| Skill Source | A connected, id-addressable source: user (Postgres row), system (Seed List entry), or template (synthesised from a Template's `skillSources`) (`docs/ubiquitous-language.md:59 @c307f40`). |
| Installed Skill Ref | A record that a Scanned Skill is installed at a Version on an Agent; identity `(agentId, source, name)` (`docs/ubiquitous-language.md:60 @c307f40`). |
| Skill Publish Record | A denormalized record that a Local Skill was published as a PR to a Skill Source (`docs/ubiquitous-language.md:61 @c307f40`). |
| Seed List | Cluster-admin-declared system Skill Sources injected as JSON (`SKILL_SOURCES_SEED`) at startup; merged with `system: true`, protected from user deletion (`docs/ubiquitous-language.md:62 @c307f40`). |

### Skills — agent-runtime side (pod-side files)

| Term | Definition |
|------|-----------|
| Skill | A directory with `SKILL.md` (`name`/`description` frontmatter) — the unit of installation (`docs/ubiquitous-language.md:70 @c307f40`). |
| Skill Path | An absolute on-pod directory under which Skills are materialised; identity within a path is the directory name (`docs/ubiquitous-language.md:71 @c307f40`). |
| Local Skill | A Skill present in some Skill Path on this pod, whether installed or authored in place (`docs/ubiquitous-language.md:72 @c307f40`). |
| Skill Source | A git repository URL containing Skills under `skills/*` or top-level `*` (`docs/ubiquitous-language.md:73 @c307f40`). |
| Scanned Skill | A Skill discovered in a Source: `(source, name, description, version, contentHash)`, version = Source HEAD SHA at scan time (`docs/ubiquitous-language.md:74 @c307f40`). |
| Content Hash | Deterministic SHA-256 over a Skill directory's contents — the drift signal *produced but not compared* on this side (`docs/ubiquitous-language.md:75 @c307f40`). |
| Install / Publish / Scan | The on-pod *acts*: materialise, lift-to-PR, enumerate Scanned Skills (`docs/ubiquitous-language.md:76-78 @c307f40`). |

---

## Approvals

Gating decisions on a user verdict; see
[Security & credentials](../concepts/security-and-credentials.md).

| Term | Definition |
|------|-----------|
| Approval | A user-pending decision gating a credentialed egress request (ext_authz) or a harness tool call (acp_native); in `pending_approvals` (`docs/ubiquitous-language.md:84 @c307f40`). |
| Pending Approval | An approval whose verdict isn't yet decided; lives in the Inbox (`docs/ubiquitous-language.md:85 @c307f40`). |
| Inbox | The user surface listing pending approvals — page, sidebar bell, per-Agent tray (`docs/ubiquitous-language.md:86 @c307f40`). |
| Verdict | `allow_once`, `allow`, `deny_once`, or `deny`; the `*_once` forms resolve only the held call, `allow`/`deny` also write a permanent egress rule (`docs/ubiquitous-language.md:87 @c307f40`). |
| Held Call | An ext_authz request blocking on the API Server up to `approvalHoldSeconds` (default 30 min); the durable pending row outlives the hold (`docs/ubiquitous-language.md:90 @c307f40`). |
| ext_authz Gate | The service running Envoy's ext_authz check: rule lookup, pending-row creation, synth-frame fan-out, hold, wake-up, expiry (`docs/ubiquitous-language.md:91 @c307f40`). |

(Also defined: Action Outcome, Synth Frame, Wrapper Response, Approvals Relay
Service — `docs/ubiquitous-language.md:88-93 @c307f40`.)

## Egress Rules

| Term | Definition |
|------|-----------|
| Egress Rule | A persistent allow/deny decision keyed on `(agent, host, method, path_pattern)`, matched on every ext_authz check before any prompt (`docs/ubiquitous-language.md:99 @c307f40`). |
| Rule Verdict | `allow` or `deny` (`docs/ubiquitous-language.md:100 @c307f40`). |
| Rule Match | Lookup of the most-specific active rule; misses fall through to the ext_authz Gate (`docs/ubiquitous-language.md:101 @c307f40`). |

## Secrets

User-owned credentials; see
[Security & credentials](../concepts/security-and-credentials.md).

| Term | Definition |
|------|-----------|
| Secret | A user-owned credential stored as a K8s Secret labelled with the owner's `sub`, mounted into the agent pod's Envoy sidecar for wire-level injection (`docs/ubiquitous-language.md:126 @c307f40`). |
| Secret Type | Provider taxonomy: `anthropic` (fixed hostPattern) or `generic` (user host/path patterns) (`docs/ubiquitous-language.md:127 @c307f40`). |
| Host Pattern | The hostname pattern that selects which outbound requests get this secret injected (`docs/ubiquitous-language.md:128 @c307f40`). |
| Secret Assignment | Linkage between a Secret and an Agent, via `secret-mode` + `granted-secret-ids` annotations on the Agent ConfigMap (`docs/ubiquitous-language.md:129 @c307f40`). |
| Provider | The external service a secret authenticates against; determines default routing for typed secrets (`docs/ubiquitous-language.md:130 @c307f40`). |

## Terms

| Term | Definition |
|------|-----------|
| Terms of Use | The legal contract a user accepts before driving Platform; text/version from Helm values (`docs/ubiquitous-language.md:136 @c307f40`). |
| Acceptance | A per-(user, version) record proving acceptance; append-only in `terms_acceptances` (`docs/ubiquitous-language.md:139 @c307f40`). |
| Acceptance Gate | Middleware that refuses requests from a sub whose latest Acceptance ≠ current Terms Version (412 with `{currentVersion, currentHash}`) (`docs/ubiquitous-language.md:140 @c307f40`). |

(Also: Terms Version, Terms Hash, Stale Acceptance —
`docs/ubiquitous-language.md:137-141 @c307f40`.)

## Usage Tracking

| Term | Definition |
|------|-----------|
| Activity Event | An append-only `activity_events` row for one meaningful interaction, carrying actor, agent, surface, outcome (`docs/ubiquitous-language.md:147 @c307f40`). |
| Actor Sub | Pseudonymized user id: `HMAC-SHA256(ACTIVITY_HMAC_KEY, keycloak_sub)`, joinable across tables (`docs/ubiquitous-language.md:148 @c307f40`). |
| Agent Mirror | The Postgres `agents` table — a projection of agent ConfigMaps for SQL `agent_id → owner_sub` resolution (`docs/ubiquitous-language.md:151 @c307f40`). |
| Usage View | A named `usage_*` SQL view aggregating Activity Events; view names are the public read API (`docs/ubiquitous-language.md:153 @c307f40`). |

(Also: Sub Pseudonymizer, Activity Outcome, Inspector, Pilot Metric Filter —
`docs/ubiquitous-language.md:149-154 @c307f40`.)

## Platform CLI

The `dam` command-line client at `packages/cli/`
(`docs/ubiquitous-language.md:160 @c307f40`).

| Term | Definition |
|------|-----------|
| Platform CLI | The `dam` client talking to a hosted Platform deployment (`docs/ubiquitous-language.md:160 @c307f40`). |
| Active Host | The Server URL the CLI targets — resolved from `--server`, env, `config.toml`; also the Auth Store key (`docs/ubiquitous-language.md:165 @c307f40`). |
| Auth Store | Machine-managed credential file at `$XDG_STATE_HOME/dam/auth.toml` (0600, atomic), keyed by Host URL (`docs/ubiquitous-language.md:167 @c307f40`). |
| Compat Verdict | CLI-vs-server version compare: `Ok`, `BehindMinClient` (hard-refuse), `BehindCurrent` (soft-warn) (`docs/ubiquitous-language.md:164 @c307f40`). |
| Agent Ref / Agent Resolver | The user string addressing an Agent (ID or name) and the service resolving it to the owner's Agent (`docs/ubiquitous-language.md:170-171 @c307f40`). |

(Also: Config, Config Source, Server URL, Host Auth, Token Provider, CLI Client
— `docs/ubiquitous-language.md:161-169 @c307f40`.)

## Connections (proposed, in-flight design)

> **In active design.** This context generalises today's split between
> `OAuthAppDescriptor` and `ProviderPreset` into one model; structure is still
> being grilled (`docs/ubiquitous-language.md:103-105 @c307f40`). Treat these
> terms as provisional.

| Term | Definition |
|------|-----------|
| Connection Template | A code-level catalog entry shipping defaults (`AuthConfig` + `Contribution[]` + input fields); replaces `OAuthAppDescriptor` + `ProviderPreset` (`docs/ubiquitous-language.md:109 @c307f40`). |
| Connection | A uniform shape `{ auth, contributions, inputs, templateId? }` with no `kind` discriminator — identity is its contributions + auth (`docs/ubiquitous-language.md:110 @c307f40`). |
| Contribution | One typed unit a Connection emits per Agent: `env`, `egress-host`, `file`, `mcp-entry`, `skill-ref` (provisional, extensible) (`docs/ubiquitous-language.md:111 @c307f40`). |
| AuthConfig | Discriminated union of how a Connection authenticates: `oauth`, `header`, `none` (`docs/ubiquitous-language.md:112 @c307f40`). |
| State Slice / Event / Version | The `applyState` reconciliation payload: a desired-Contributions snapshot, one-shot directives, and a per-Agent monotonic counter (`docs/ubiquitous-language.md:113-115 @c307f40`). |
| Settled / Applied Version, Apply Failures | The agent's reconciliation progress, last fully-applied state, and failed drivers driving background retry (`docs/ubiquitous-language.md:116-120 @c307f40`). |

---

## See also

- [Persistence — Infra vs Application State](../concepts/persistence.md)
- [Agent lifecycle](../concepts/agent-lifecycle.md)
- [Channels (and Forks)](../concepts/channels.md)
- [Skills — both sides](../concepts/skills.md)
- [Agent (entity)](../entities/agent.md)
