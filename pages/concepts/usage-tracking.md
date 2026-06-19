---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files:
  - docs/architecture/usage-tracking.md
  - packages/db/src/schema.ts
  - packages/db/drizzle/0001_usage_views.sql
  - packages/api-server/src/core/sub-pseudonymizer.ts
  - packages/api-server/src/modules/usage/routes.ts
  - packages/api-server/src/modules/usage/compose.ts
  - packages/api-server/src/modules/usage/sagas/persist-activity.ts
  - packages/api-server/src/modules/usage/infrastructure/activity-events-repository.ts
  - packages/api-server/src/modules/usage/infrastructure/activity-retention.ts
  - packages/api-server/src/modules/usage/sagas/activity-retention-job.ts
  - packages/api-server/src/modules/usage/services/report-service.ts
  - packages/api-server/src/modules/usage/services/bootstrap-agents.ts
  - packages/ui/src/modules/usage/devtools.ts
updated: 2026-06-19
---

# Usage Tracking

Usage tracking is an **operator-facing** subsystem that captures
semantically-meaningful user activity in Postgres, shapes it into SQL views, and
exposes those views to a dedicated inspector role through an HTML report and a
JSON endpoint (`docs/architecture/usage-tracking.md:7 @c307f40`). It answers
operational questions — daily-active users by surface, channel turns by agent,
schedule fires, OAuth connection lifecycle, file-import volumes,
contribution-delivery health — rather than product analytics
(`docs/architecture/usage-tracking.md:7 @c307f40`).

The whole subsystem is the **api-server's** responsibility end-to-end: the
controller and agent-runtime do not participate. Writes happen in-process on the
existing event bus; reads happen on a Keycloak-authenticated HTTP route mounted
under the same Hono app (`docs/architecture/usage-tracking.md:15 @c307f40`). See
[../sources/api-server.md](../sources/api-server.md) and
[../sources/db.md](../sources/db.md).

## Three design choices

The operator framing drives three decisions
(`docs/architecture/usage-tracking.md:9-13 @c307f40`):

1. **The read interface is SQL views.** Adding a metric is a new view; consumers
   never see the raw event table. The HTML report renders all "pilot" views; the
   JSON endpoint returns any one of them by name.
2. **Storage is pseudonymized.** Every Keycloak `sub` written to Postgres is
   HMAC-SHA256 hashed with a per-install secret at the repository write boundary.
   Same input maps to same output, so cross-table joins and `GROUP BY sub` still
   work; reverse lookup requires the secret, which lives on the api-server pod.
3. **Access is a separate role.** The `platform-inspector` realm role gates
   `/api/usage/*`, independent of platform-access — "can read aggregates" does not
   imply "can use the platform."

## Bounded context

The subsystem owns three tables plus the `usage_*` views
(`docs/architecture/usage-tracking.md:67-72 @c307f40`):

- **`activity_events`** — the append-only event log, one row per recorded
  interaction. Columns: `id`, `type`, `actor_sub` (HMACed, nullable), `agent_id`,
  `surface`, `outcome` (a `success | failure` enum), `payload` (JSONB), and
  `occurred_at` (`packages/db/src/schema.ts:195-226 @c307f40`). The `outcome` enum
  is constrained at the DB so a typo surfaces as a constraint violation rather
  than a silently miscounting row (`packages/db/src/schema.ts:16-22 @c307f40`).
- **`actor_roles`** — a role flag per pseudonymized sub, recording whether the
  user carried the configured "core" realm role at auth time; read by
  `usage_core_actor_subs` to power the core-team exclusion filter
  (`packages/db/src/schema.ts:228-237 @c307f40`).
- **`agents`** — a Postgres mirror of agent ConfigMaps (`id`, `owner_sub` HMACed,
  `created_at`, `deleted_at`) so SQL views can resolve agent ownership without a
  K8s API round-trip (`packages/db/src/schema.ts:239-261 @c307f40`).

It **reads from but does not own** other tables — `pending_approvals`,
`agent_skills`, `egress_rules` — projecting read-only summaries over them. Schema
changes there can force view rewrites, but view rewrites never touch the source
tables (`docs/architecture/usage-tracking.md:74-78 @c307f40`). Session-derived
views were retired when sessions became agent-owned
(`docs/architecture/usage-tracking.md:76 @c307f40`,
`packages/db/src/schema.ts:150-151 @c307f40`).

## Write path

The api-server already emits domain events for its own purposes on every
meaningful interaction; usage only adds subscribers
(`docs/architecture/usage-tracking.md:82 @c307f40`). Two sagas subscribe to the
bus:

- **persist-activity** writes one `activity_events` row per `UserAuthenticated`,
  `ChannelTurnRelayed`, `ScheduleFired`, `ConnectionCreated`, `ConnectionRemoved`,
  `FilesImported`, `ContributionApplyFailed`, `ContributionRecovered`, or
  `ContributionApplyGaveUp` event; the auth subscriber also upserts `actor_roles`
  with the user's core-role flag
  (`packages/api-server/src/modules/usage/sagas/persist-activity.ts:34-257 @c307f40`).
- **persist-agents** writes one `agents` row per `AgentCreated` and marks deleted
  on `AgentDeleted`; a startup bootstrap separately backfills the table from the
  K8s API for agents that pre-dated the saga
  (`packages/api-server/src/modules/usage/compose.ts:66-77 @c307f40`,
  `packages/api-server/src/modules/usage/services/bootstrap-agents.ts:13-21 @c307f40`).

Each subscriber bounds concurrency with an RxJS `mergeMap` capped at 8 in-flight
writes, so an auth burst (restart, silent-renew storm) cannot saturate the pg
pool (`packages/api-server/src/modules/usage/sagas/persist-activity.ts:24-27 @c307f40`).
The auth path additionally exploits a partial unique index — one row per
`(actor_sub, surface, UTC-day)` for `type='auth'` — so heavy auth traffic does
not bloat the table (`packages/db/src/schema.ts:218-224 @c307f40`), and the
`actor_roles` upsert is a no-op after the first call per UTC day via a `setWhere`
guard (`packages/api-server/src/modules/usage/infrastructure/activity-events-repository.ts:23-34 @c307f40`).

persist-activity runs only when activity tracking is enabled at install time (a
chart-level toggle, on by default); persist-agents and the bootstrap run
unconditionally because the `agents` table is useful outside usage too
(`packages/api-server/src/modules/usage/compose.ts:65-92 @c307f40`).

## Pseudonymization

Every Keycloak `sub` written to Postgres is replaced with
`HMAC-SHA256(key, sub)` rendered as a 64-char hex string
(`packages/api-server/src/core/sub-pseudonymizer.ts:22-25 @c307f40`). The key —
`ACTIVITY_HMAC_KEY` — is a per-install secret auto-generated by the Helm chart on
first install and persisted across upgrades
(`docs/architecture/usage-tracking.md:97 @c307f40`). The constructor refuses to
run with an empty key, so the subsystem never silently writes raw subs
(`packages/api-server/src/core/sub-pseudonymizer.ts:15-19 @c307f40`).

The hashing is applied at a **single chokepoint**: the repository layer hashes
the sub immediately before INSERT — `actor_sub`, `actor_roles.actor_sub`, and
`agents.owner_sub` all go through the same pseudonymizer — while emit sites and
sagas keep dealing in raw subs in-memory
(`packages/api-server/src/modules/usage/infrastructure/activity-events-repository.ts:13,27 @c307f40`,
`packages/api-server/src/modules/usage/compose.ts:49-51 @c307f40`).

Determinism is load-bearing: the same key applied across all three columns is
what makes the views joinable
(`docs/architecture/usage-tracking.md:109 @c307f40`). What it protects against
and what it does not:

- A **database-only leak** (backup exfiltration, replica compromise) yields opaque
  pseudonyms; re-deriving identifiers requires the api-server pod or its mounted
  Secret. An inspector running views sees pseudonyms, not Keycloak subs
  (`docs/architecture/usage-tracking.md:99-102 @c307f40`).
- It does **not** stop an attacker holding both the database and the pod (or its
  Secret), and it does not cover other surfaces that still hold raw subs (CM
  `owner` labels, OAuth Secret keys, `pending_approvals.owner_sub`,
  `identity_links.keycloak_sub`). This is GDPR Recital 26 risk reduction, not
  anonymization — the stored value remains personal data
  (`docs/architecture/usage-tracking.md:104-107 @c307f40`,
  `packages/api-server/src/core/sub-pseudonymizer.ts:3-8 @c307f40`). See
  [../concepts/security-and-credentials.md](../concepts/security-and-credentials.md).

Rotating the key orphans every existing row, so it is treated as permanent for
the install (`docs/architecture/usage-tracking.md:109 @c307f40`).

## Read interface

Three Keycloak-gated endpoints sit behind the `platform-inspector` realm role
(`docs/architecture/usage-tracking.md:113 @c307f40`):

| Endpoint | Returns |
|---|---|
| `GET /api/usage/views` | list of queryable view names (`packages/api-server/src/modules/usage/routes.ts:60-62 @c307f40`) |
| `GET /api/usage?view=<name>` | one view's rows as JSON (`packages/api-server/src/modules/usage/routes.ts:87-95 @c307f40`) |
| `GET /api/usage/report` | full HTML page rendering the pilot view set (`packages/api-server/src/modules/usage/routes.ts:64-85 @c307f40`) |

The inspector gate is a middleware mounted on `/api/usage` and `/api/usage/*`
(deliberately not a bare wildcard, which would 403 the whole parent app). A
caller missing the role is denied — HTML for `/report`, JSON otherwise — and the
denial is recorded to the security log; a successful privileged read is also
logged, distinct from the pseudonymized rows it reads
(`packages/api-server/src/modules/usage/routes.ts:25-58 @c307f40`). See
[../concepts/logging.md](../concepts/logging.md).

`VIEW_NAMES` is the static source of truth for what is queryable: the names are
the SQL identifiers, and each query is prebuilt once as `SELECT * FROM "<name>"`
from that frozen tuple, so no runtime input ever flows into `sql.raw`
(`packages/api-server/src/modules/usage/services/report-service.ts:7-66 @c307f40`).
The two helper views are queryable via JSON for inspection but excluded from the
HTML report as plumbing
(`packages/api-server/src/modules/usage/services/report-service.ts:47-56 @c307f40`).

When no inspector role is configured at install time, the routes are replaced by
an empty no-op router while the persistence sagas still run — reads are gated on
inspector config, writes on the activity-tracking toggle
(`packages/api-server/src/modules/usage/compose.ts:54-59 @c307f40`,
`docs/architecture/usage-tracking.md:123 @c307f40`).

### Opening the report

There is no visible UI affordance. The UI registers
`window.platformUsage.openReport()` unconditionally — its existence is
meaningless without the server-side role check — and inspectors invoke it from
the browser devtools console
(`packages/ui/src/modules/usage/devtools.ts:3-13 @c307f40`). The function fetches
the report with the Bearer token, wraps the response in a Blob URL, and opens it
in a new tab (a plain `<a href>` cannot carry the Bearer token); the Blob is
revoked about a minute after open
(`docs/architecture/usage-tracking.md:121 @c307f40`). A `403` means the
signed-in user lacks the inspector role
(`docs/architecture/usage-tracking.md:134 @c307f40`).

## Retention

A weekly in-process timer runs a bulk DELETE of `activity_events` rows older than
the retention horizon (180 days)
(`docs/architecture/usage-tracking.md:138 @c307f40`,
`packages/api-server/src/modules/usage/infrastructure/activity-retention.ts:6-18 @c307f40`).
Multi-replica installs race a stable `pg_try_advisory_lock` key, so only one
replica runs the DELETE per week and losers no-op; the timer starts 5 minutes
after pod start so a rolling restart does not align every replica's tick
(`packages/api-server/src/modules/usage/sagas/activity-retention-job.ts:3-51 @c307f40`).
Retention is independent of the activity-tracking toggle — if writes are disabled
mid-deployment, existing rows still age out
(`docs/architecture/usage-tracking.md:140 @c307f40`). See
[../concepts/persistence.md](../concepts/persistence.md).

## Core-team exclusion

Pilot metrics target external users, so the platform team's own traffic would
distort them. Two helper views capture the exclusion
(`docs/architecture/usage-tracking.md:144-149 @c307f40`):

- `usage_core_actor_subs` — pseudonymized subs flagged with the core role
  (`actor_roles.is_core = true`) (`packages/db/drizzle/0001_usage_views.sql:28-31 @c307f40`).
- `usage_core_agents` — agent IDs whose owner is in the core set, computed by
  joining the `agents` mirror
  (`packages/db/drizzle/0001_usage_views.sql:36-39 @c307f40`).

Every pilot view applies the appropriate exclusion —
`actor_sub NOT IN (SELECT … FROM usage_core_actor_subs)`, or its `agent_id` /
`owner_sub` analogue (`egress_rules` is the lone exception, lacking agent-owner
capture) (`packages/db/drizzle/0001_usage_views.sql:10-15 @c307f40`). The
`is_core` flag is populated at auth time from the JWT's `realm_access.roles`, so a
user added to the core role only takes effect after their next login
(`docs/architecture/usage-tracking.md:149 @c307f40`,
`packages/db/drizzle/0001_usage_views.sql:17-19 @c307f40`).

## Trust boundaries

- **Inspector role gates the read API.** Writes are unauthenticated to the
  subsystem itself — they originate inside the api-server from already-authenticated
  user requests on other routes, so the log inherits whatever trust boundary the
  originating route enforced (`docs/architecture/usage-tracking.md:153 @c307f40`).
- **The HMAC key gates re-identification.** Holding the in-cluster Secret is what
  lets a reader correlate a pseudonym back to a Keycloak `sub`; database-only
  access does not (`docs/architecture/usage-tracking.md:154 @c307f40`).
- **Ad-hoc SQL is intentionally not exposed.** An earlier `POST /api/usage/query`
  taking raw SQL was removed: an inspector with it could read other Postgres
  tables holding credential material (refresh tokens, HITL payloads). Inspectors
  get views; operators wanting `psql` go through `kubectl exec`
  (`docs/architecture/usage-tracking.md:155 @c307f40`).

## Implementation notes

- The `usage_*` views are **hand-written raw SQL**, not generated from
  `schema.ts`, because expressing the aggregate views in the ORM is lossy and
  drizzle-kit emits interdependent views in the wrong order; they live in their
  own migration (`packages/db/drizzle/0001_usage_views.sql:1-8 @c307f40`). See
  [../sources/db.md](../sources/db.md).
- Existing deployments skip that migration (its journal `when` predates every
  migration they have recorded); only fresh installs run it after the baseline
  tables (`packages/db/drizzle/0001_usage_views.sql:6-8 @c307f40`).
