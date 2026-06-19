---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [docs/architecture/skills.md, packages/db/src/schema.ts, packages/agent-runtime/src/modules/skills/services/publish.ts]
updated: 2026-06-19
---

# Skills

A **skill** is a directory holding a `SKILL.md` manifest (YAML frontmatter — `name`, `description`) plus supporting files. DAM does not interpret skills; it **transports** them between external git repositories and a per-Agent PVC, where the harness reads them from configured paths. There is no DAM-hosted catalog — sources are external git repos (`docs/architecture/skills.md:7 @c307f40`).

The whole subsystem is best understood through one split: **what DAM records** vs **what lives on a pod**. Holding that line is what keeps the design simple.

## Two bounded contexts

Skills live across two contexts that never blur (`docs/architecture/skills.md:9-14 @c307f40`):

- **api-server side — the catalog.** Which sources are connected, which skills are installed where, what was published from which Agent. This is all Application State in Postgres or api-server config. The api-server **never touches a pod's filesystem directly** (`docs/architecture/skills.md:11 @c307f40`).
- **agent-runtime side — the pod files.** Scanning a source, materializing a skill into the configured paths, walking the disk to enumerate local skills, and publishing a local skill upstream. It never reasons about catalogs, drift, or ownership (`docs/architecture/skills.md:12 @c307f40`).

The two share exactly two narrow channels: the **agent-runtime** is the only path that reaches a pod, and the paired **gateway pod** is the only path that reaches GitHub with credentials. Everything else between them is a tRPC call (`docs/architecture/skills.md:14 @c307f40`). See [api-server](../sources/api-server.md) and [agent-runtime](../sources/agent-runtime.md) for the wider package context, and [Agent](../entities/agent.md) for what a skill is installed onto.

## The catalog vocabulary (api-server side)

### Skill Source

A connection to an external git repo, addressable by id. Three kinds, merged into one list at read time and badged in the UI (`docs/architecture/skills.md:65-73 @c307f40`):

- **User source** — a row in `skill_sources` (Postgres), owner-scoped, created/deleted by the user via tRPC.
- **System source** — a Helm-declared, platform-wide entry loaded from the `SKILL_SOURCES_SEED` env at boot (the **Seed List**), never persisted to Postgres, marked `system: true`, protected from deletion. Badged "Platform".
- **Template source** — declared on a template's `spec.skillSources`, surfaced read-only on every Agent derived from that template. Badged "Agent".

Listing dedupes on `gitUrl` with first-wins precedence **user → template → system**, so a user source for the same URL shadows the system entry, and deleting that row re-exposes the system entry (`docs/architecture/skills.md:73 @c307f40`).

### Installed Skill Ref

A **Scanned Skill** is what a scan returns from a Source: `(name, description, version, contentHash)`, where `version` is the source's HEAD commit SHA and `contentHash` is a deterministic SHA-256 over the skill directory (`docs/architecture/skills.md:77 @c307f40`).

An **Installed Skill Ref** is a row in `agent_skills` keyed `(agentId, source, name)` recording which Scanned Skill is installed at which version on which Agent (`docs/architecture/skills.md:79 @c307f40`; schema at `packages/db/src/schema.ts:173-189 @c307f40`). Critically, the row is **a declarative record, not the source of truth** — the on-disk directory in the configured Skill Paths is authoritative, and the row self-heals on every `state` query.

### Local Skill, Standalone, and the Publish Record

A **Local Skill** is any directory present in a Skill Path on the pod, however it got there. The reconciled `state` view splits Locals into **Installed** (also tracked in `agent_skills`; drift surfaces when the Postgres `contentHash` differs from the upstream scan) and **Standalone** (on disk but untracked — authored in place via the Files panel) (`docs/architecture/skills.md:81-84 @c307f40`).

A **Skill Publish Record** (`agent_skill_publishes`) is the explicit log of a successful publish: `skillName`, `sourceId`, `prUrl`, plus `sourceName` and `sourceGitUrl` **denormalized** so the record outlives renaming or deletion of the source (`docs/architecture/skills.md:86 @c307f40`; schema at `packages/db/src/schema.ts:276-291 @c307f40`). It drives the "Published" badge, replacing a name-match heuristic that produced false positives.

## The pod vocabulary (agent-runtime side)

### Skill Path

An absolute on-pod directory the harness reads skills from — the `skill-ref` driver's `paths` in the Agent's runtime manifest. The agent-runtime resolves it for both install and the read-side views; **the api-server never passes paths over the wire** (`docs/architecture/skills.md:90 @c307f40`). Every image inherits a default path from platform-base's `runtime-manifest.yaml`, and each per-Agent Dockerfile symlinks its harness-native skills dir onto that canonical store — so an install writes once on disk regardless of harness, with no per-Agent manifest override (`docs/architecture/skills.md:92 @c307f40`).

Install writes the skill directory into **every** configured Skill Path; uninstall removes it from all of them. Disk scans walk every path in order and dedupe by directory name, first found wins (`docs/architecture/skills.md:94 @c307f40`).

## Install: REST tarball or shallow clone

`skills.install(agentId, source, name, version)` flows api-server → agent-runtime → gateway → GitHub → PVC, then the api-server upserts the `agent_skills` row (`docs/architecture/skills.md:156-166 @c307f40`). The api-server first **wakes a hibernated Agent** via the shared `ensureReady` primitive, since the install touches the pod (`docs/architecture/skills.md:106 @c307f40`).

agent-runtime fetches the source at the requested `version` (`docs/architecture/skills.md:118 @c307f40`):

- **GitHub URLs** — the REST tarball endpoint, anonymous first, retried authenticated on a 404 to distinguish "not found" from "private".
- **Everything else** — a shallow `git` clone.

It resolves the skill directory inside the fetch (`skills/<name>/` then top-level `<name>/`), copies it into every configured Skill Path, and returns the deterministic `contentHash`. Public-source installs are identical except the gateway has no credential to inject — an anonymous archive download (`docs/architecture/skills.md:168 @c307f40`).

## Publish: REST-only, back as a PR

Publish is **REST-only — there is no `git push` on this path** (`docs/architecture/skills.md:120,238 @c307f40`). `skills.publish(agentId, sourceId, name)` first has the api-server validate that the source is a GitHub URL (the only host that supports publish) and wake the Agent (`docs/architecture/skills.md:107,182-184 @c307f40`).

agent-runtime then reads the local skill from disk (size-capped per file and per skill) and drives the GitHub REST API end to end — no git subprocess, no working copy (`packages/agent-runtime/src/modules/skills/services/publish.ts:22-25 @c307f40`):

1. one **blob** per file (`publish.ts:55-66 @c307f40`),
2. a **tree** parented on the default-branch HEAD tree (`publish.ts:68-78 @c307f40`),
3. a **commit** authored as `Platform <platform-publish@users.noreply.github.com>` (`publish.ts:85-93 @c307f40`),
4. a **branch** ref named `platform/publish-<name>-<timestamp>` (`publish.ts:96-101 @c307f40`),
5. a **pull request** against the default branch (`publish.ts:104-111 @c307f40`).

The author and branch prefix are deliberately neutral ("platform"), not brand-driven — agent pods are brand-blind (`publish.ts:80-84 @c307f40`). On success the api-server writes the `agent_skill_publishes` row and invalidates the scan cache for that source so a freshly-merged PR appears on the next list (`docs/architecture/skills.md:107,191-192 @c307f40`). GitHub errors (missing scope, repo not found) surface as the upstream HTTP response, forwarded verbatim into the tRPC error so the UI renders the right CTA (`docs/architecture/skills.md:196 @c307f40`).

## Envoy sidecar credential injection for GitHub

agent-runtime **never holds a real GitHub token** — it makes every authenticated GitHub call *unauthenticated*. The paired gateway pod performs the swap on the wire (`docs/architecture/skills.md:126-133,237 @c307f40`). See [security and credentials](../concepts/security-and-credentials.md) for the full model.

1. The agent pod's `HTTPS_PROXY` points at its per-Agent gateway Service, and the pod's NetworkPolicy admits **no other route** to TCP 80/443 — so injection is enforced by the cluster, not by the Agent honoring an env var. `SSL_CERT_FILE` points at the cluster-issued MITM CA so the gateway's TLS termination succeeds (`docs/architecture/skills.md:128 @c307f40`).
2. Envoy renders **three** host-specific filter chains from one GitHub OAuth Secret (`docs/architecture/skills.md:129-132 @c307f40`):
   - `api.github.com` → `Authorization: Bearer <token>` (REST/GraphQL).
   - `github.com` → `Authorization: Basic base64("x-access-token:<token>")` — the HTTP Basic shape `git`-over-HTTPS expects, so private `git clone`/`fetch`/`push` work with no credential helper.
   - `raw.githubusercontent.com` → `Authorization: Bearer <token>` (private raw-file fetches).
3. agent-runtime calls without authenticating; Envoy supplies the credential (`docs/architecture/skills.md:133 @c307f40`).

When env credentials arrive over the runtime channel, agent-runtime runs `gh auth setup-git`, so a private-repo `git clone` invoked from inside the pod also routes through `gh` (and thus the gateway's injector) instead of stalling on a username prompt — it deliberately does **not** run at boot, where credentials aren't yet available (`docs/architecture/skills.md:122 @c307f40`). The runtime exposes `PLATFORM_GH_TOKEN_AVAILABLE=true|false` so wrappers can short-circuit; because it is derived in-pod from reconciled env, it reads `false` during the cold-boot window and flips on the next respawn — treat `false` as "not yet known," not "permanently absent" (`docs/architecture/skills.md:135-137 @c307f40`). This is the same mechanism connections rely on; see [connections](../concepts/connections.md).

## Reconciled state: every read is the reconciler

`skills.state(agentId)` is the single read the UI uses to render the Skills panel. It composes `listLocal` from agent-runtime (what's actually on disk, deduped across paths), the `agent_skills` rows, and the `agent_skill_publishes` rows; then it **drops ghost rows** — `agent_skills` rows whose directory was deleted out-of-band, e.g. via the Files panel — and persists the cleanup (`docs/architecture/skills.md:211-217 @c307f40`). The Postgres rows stop drifting from the filesystem without a separate reconciler, because every read *is* the reconciler.

Listing splits the same way: `skills.list` serves **public GitHub** from the api-server's `public-archive-scanner` (per-`gitUrl`, 5-minute TTL cache, no Agent required) and falls through to agent-runtime `skills.scan` for **anything else**, which requires an `agentId` and wakes a hibernated Agent first (`docs/architecture/skills.md:104-105,202-207 @c307f40`).

## Trust boundaries and invariants

- **Filesystem is authoritative for installed state.** `agent_skills` self-heals on every `state` read; a skill removed via the Files panel disappears from the UI with no explicit uninstall (`docs/architecture/skills.md:235 @c307f40`).
- **api-server never touches the pod filesystem.** Every disk-touching op goes through agent-runtime's tRPC port; the agent pod's NetworkPolicy admits ingress only from the api-server pod, so no in-process auth is needed on that hop — no Bearer token is sent (`docs/architecture/skills.md:106,236 @c307f40`).
- **agent-runtime never holds a GitHub credential.** The token is mounted only into the gateway pod, never the agent pod, and the agent pod has no route to GitHub except through that gateway — so a compromised agent pod cannot exfiltrate it (`docs/architecture/skills.md:237 @c307f40`).
- **Publish is REST-only.** No `git push`; `git` is used only for cloning non-GitHub sources, and that path too routes through the gateway via `gh auth setup-git` (`docs/architecture/skills.md:238 @c307f40`).
- **MCP `agentId` is server-bound.** Five MCP tools (`list_skill_sources`, `list_skills_in_source`, `install_skill`, `uninstall_skill`, `publish_skill`) are registered on the per-Agent MCP endpoint; `agentId` is pinned from the verified MCP session token, not tool input — agents cannot spoof which Agent they act on (`docs/architecture/skills.md:109,239 @c307f40`).

## Persistence

Skills are entirely an Application State subsystem — see [persistence](../concepts/persistence.md). Three Postgres tables (`docs/architecture/skills.md:221-227 @c307f40`):

| Table | Key | Owner |
|---|---|---|
| `skill_sources` | `(id)`, unique on `(owner, gitUrl)` | per-user (`packages/db/src/schema.ts:153-171 @c307f40`) |
| `agent_skills` | `(agentId, source, name)` | per-Agent (`packages/db/src/schema.ts:173-189 @c307f40`) |
| `agent_skill_publishes` | `(id)`, indexed by `agentId` | per-Agent (`packages/db/src/schema.ts:276-291 @c307f40`) |

System and template sources do **not** persist — they are computed at request time from `SKILL_SOURCES_SEED` and the template's `spec.skillSources` (`docs/architecture/skills.md:229 @c307f40`). On-pod state lives on the per-Agent PVC under the Skill Paths; PVC reclamation handles file-side cleanup on Agent deletion, while a cleanup saga deletes the `agent_skills` and `agent_skill_publishes` rows. User-owned `skill_sources` survive Agent deletion — they are catalog connections, not Agent state (`docs/architecture/skills.md:110,231 @c307f40`).
