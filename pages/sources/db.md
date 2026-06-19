---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [packages/db/README.md, packages/db/src/schema.ts, packages/db/src/migrate.ts, packages/db/src/index.ts, packages/db/src/client.ts, packages/db/drizzle.config.ts, packages/db/drizzle/0000_squashed_baseline.sql, packages/db/drizzle/0001_usage_views.sql, packages/db/drizzle/0002_marvelous_dragon_man.sql, packages/db/drizzle/meta/_journal.json, packages/api-server/src/index.ts]
updated: 2026-06-19
---

# db (package)

`packages/db/` is the api-server's Postgres state: the schema the application
compiles against, the ordered SQL migrations that build the database, and the
hand-written reporting views. It is a small, dependency-light package consumed by
[api-server](./api-server.md); see [persistence](../concepts/persistence.md) and
[usage tracking](../concepts/usage-tracking.md) for the broader picture.

## Two artifacts, one split

The package separates *what the app type-checks against* from *what builds the DB*
(`packages/db/README.md:1-13 @c307f40`):

- **`src/schema.ts`** — Drizzle ORM table/column/index/enum definitions. This is
  the schema the app's queries are type-checked against. Tables, columns, indexes
  and enums live here and are **generated** into SQL with `drizzle-kit generate`;
  hand-writing them is forbidden and `db:check:generated` fails the build if
  `schema.ts` drifts from the snapshot (`packages/db/README.md:12,37-39 @c307f40`).
- **`drizzle/`** — the ordered SQL migrations, plus `meta/_journal.json` (their
  order) and `meta/NNNN_snapshot.json` (Drizzle's model of the schema after each
  generated migration) (`packages/db/README.md:6 @c307f40`).

The key wrinkle: the `usage_*` reporting views are **not** in `schema.ts`.
Expressing 23 aggregate views in Drizzle's DSL is lossy and `generate` can't order
interdependent views correctly, so the views are hand-written raw SQL, scaffolded
with `db:new` (`packages/db/README.md:8-13 @c307f40`).

## Migration workflow (runs on api-server startup)

There is no manual migrate step in production. `runMigrations` opens a
single-connection pool, wraps it in Drizzle, and runs the Postgres-JS migrator
against the `drizzle/` folder (`packages/db/src/migrate.ts:6-16 @c307f40`). The
api-server calls it during boot, before opening the main DB handle
(`packages/api-server/src/index.ts:116-117 @c307f40`).

The migrator runs a migration only if its journal `when` is **strictly greater**
than the newest `created_at` already recorded in `drizzle.__drizzle_migrations`
(`packages/db/README.md:45 @c307f40`).

`drizzle.config.ts` points generation at `./src/schema.ts` → `./drizzle`, dialect
`postgresql` (`packages/db/drizzle.config.ts:3-11 @c307f40`).

### The squash

The original history (migrations 0000–0022) was collapsed into two from-scratch
migrations (`packages/db/README.md:41-45 @c307f40`,
`packages/db/drizzle/0000_squashed_baseline.sql:1-3 @c307f40`):

- **`0000_squashed_baseline.sql`** — the table/index/enum DDL,
  `drizzle-kit generate` output (18 `CREATE TABLE`).
- **`0001_usage_views.sql`** — the 23 reporting views, hand-written in dependency
  order. Split out so the baseline stays purely generated.

Both squash migrations carry `when` values (`1776084226452` and `1776084226453`)
that predate every migration an existing deployment has recorded, so existing
deployments treat them as already-applied and skip them; only fresh installs run
them, tables then views (`packages/db/drizzle/meta/_journal.json:8,15 @c307f40`,
`packages/db/drizzle/0000_squashed_baseline.sql:5-8 @c307f40`). **Do not change
these `when` values** (`packages/db/README.md:45 @c307f40`). The post-squash tail
adds normal generated migrations, e.g. `0002_marvelous_dragon_man.sql` creating
the `api_keys` table (`packages/db/drizzle/0002_marvelous_dragon_man.sql:1-15
@c307f40`, `packages/db/drizzle/meta/_journal.json:19-24 @c307f40`).

> Caveat from the README: the skip relies on existing deployments being fully
> migrated to the pre-squash head before the squash ships; a deployment stuck
> mid-history would skip the baseline without the tail migrations
> (`packages/db/README.md:47 @c307f40`).

## The schema, mapped

`schema.ts` defines one enum and ~19 tables (re-exported from the package entry,
`packages/db/src/index.ts:3-23 @c307f40`). The load-bearing ones:

**`agents`** — Postgres mirror of K8s agent ConfigMaps, kept here so SQL views and
cross-table joins can resolve agent ownership without a CM round-trip. Keyed on
`id`; `owner_sub` is HMACed with the same key as `activity_events.actor_sub`.
Populated by the persist-agents saga (on `AgentCreated`/`AgentDeleted`) plus a
startup bootstrap backfilling pre-saga agents. Also holds runtime hello/capability
fields (`packages/db/src/schema.ts:239-261 @c307f40`). See the
[agent entity](../entities/agent.md).

**`schedules`** — per-agent scheduled runs: `spec` (jsonb), `enabled`, `next_run`,
`last_fired_at`/`last_fired_result`. Indexed by `(agent_id, owner)` and a partial
index over enabled rows (`packages/db/src/schema.ts:394-419 @c307f40`).

**`identity_links`** — maps an external identity `(provider, external_user_id)` to
a `keycloak_sub`; composite PK on `(provider, external_user_id)`
(`packages/db/src/schema.ts:40-51 @c307f40`).

**`telegram_threads`** — authorized Telegram threads per agent, PK
`(agent_id, thread_id)`, recording `authorized_by`
(`packages/db/src/schema.ts:63-74 @c307f40`).

**`activity_events`** — append-only log of semantically-meaningful platform
activity (auth, channel turns, schedule fires, connection grants, file imports).
`actor_sub` is HMAC-SHA256(keycloak_sub, …) — pseudonymized at the storage
boundary, the same key that joins to `actor_roles` and `agents.owner_sub`.
`outcome` is constrained by the `activity_outcome` enum (`success` / `failure`)
so a typo surfaces as a constraint violation rather than a miscounting row. A
unique `auth_dedup` index collapses repeated auth events to one per actor/surface
per UTC day (`packages/db/src/schema.ts:16-22,191-226 @c307f40`). This table is
the substrate for most usage views.

**`actor_roles`** — role flags keyed by pseudonymized Keycloak sub; `is_core`
marks core-team members. Upserted by the persist-activity saga on every
`UserAuthenticated` event, and read by `usage_core_actor_subs` to drive core-team
exclusion in the usage views (`packages/db/src/schema.ts:228-237 @c307f40`).

**Skills catalog tables**:
- `skill_sources` — owner-scoped git sources of skills, unique on `(owner, git_url)`
  (`packages/db/src/schema.ts:153-171 @c307f40`).
- `agent_skills` — which skills are installed on which agent; PK
  `(agent_id, source, name)` with `version`/`content_hash`
  (`packages/db/src/schema.ts:173-189 @c307f40`).
- `agent_skill_publishes` — record of skills an agent published back to a source as
  a PR (`packages/db/src/schema.ts:276-291 @c307f40`).

**`api_keys`** — programmatic-access keys: `owner_sub`, hashed secret (`hash`,
unique-indexed), `scopes[]`, optional `agent_ids[]` scoping, plus
`expires_at`/`last_used_at`/`revoked_at`. The owner index is partial over
non-revoked keys (`packages/db/src/schema.ts:421-443 @c307f40`).

Other tables in the schema (not detailed here): `channels`, `allowed_users`,
`egress_rules`, `pending_approvals` (durable HITL approvals),
`connections`/`connection_grants`, `runtime_state_outbox`/`runtime_events`
(runtime state delivery), and `terms_acceptances`
(`packages/db/src/schema.ts:24-38,53-61,87-148,293-392 @c307f40`). Sessions are
deliberately **not** stored here — the agent's on-disk store is the source of
truth, surfaced over ACP `_meta` (`packages/db/src/schema.ts:150-151 @c307f40`).

## The `usage_*` reporting views

`0001_usage_views.sql` defines 23 views in dependency order (ADR-048 / issue #739)
(`packages/db/drizzle/0001_usage_views.sql:1-8 @c307f40`). Structure:

- **Helper views first.** `usage_core_actor_subs` selects subs flagged
  `is_core` from `actor_roles`; `usage_core_agents` joins `agents` to that to find
  core-team-owned agents (`packages/db/drizzle/0001_usage_views.sql:28-39
  @c307f40`). Every pilot view excludes core-team activity by filtering against
  these two helpers (`packages/db/drizzle/0001_usage_views.sql:10-21 @c307f40`).
- **Auth views** (from `activity_events` `type='auth'`): active users 7d, surface
  breakdowns, multi-surface users, distinct-users-per-day
  (`packages/db/drizzle/0001_usage_views.sql:46-119 @c307f40`).
- **Channel views** (`type='channel_turn'`): turns by agent/day, top agents 30d,
  with success/failure splits via `FILTER`
  (`packages/db/drizzle/0001_usage_views.sql:126-167 @c307f40`).
- **Schedule fires** (`type='schedule_fire'`, not `sessions` — continuous-mode
  schedules reuse one sessions row across fires, so counting sessions
  under-reports) (`packages/db/drizzle/0001_usage_views.sql:170-201 @c307f40`).
- **Approvals** (from `pending_approvals`), **skills** (from `agent_skills` +
  `skill_sources`), **egress** (from `egress_rules`), **connections**
  (`type='connection_added'`/`'connection_removed'`), and **file imports**
  (`type='files_imported'`) (`packages/db/drizzle/0001_usage_views.sql:208-367
  @c307f40`).

Statements are separated by `--> statement-breakpoint` so the migrator applies
each `CREATE VIEW` in order — views must be created after the views they depend on
(`packages/db/README.md:30-32 @c307f40`).

## Connection / TLS

`createDb` builds a Postgres-JS connection and wraps it in Drizzle with the schema
bound for typed queries; `buildDbSsl` injects a custom CA only when one is
provided (an external managed DB with a private CA), otherwise the DSN's `sslmode`
governs (`packages/db/src/client.ts:17-25 @c307f40`). The same CA path feeds both
`runMigrations` and `createDb` at api-server boot
(`packages/api-server/src/index.ts:111-117 @c307f40`).

## See also

- [persistence](../concepts/persistence.md) — how api-server uses this state.
- [usage tracking](../concepts/usage-tracking.md) — the reporting story the views serve.
- [agent entity](../entities/agent.md) — the `agents` mirror table's subject.
- [api-server](./api-server.md) — the sole consumer; runs migrations on startup.
- [dam](./dam.md) — platform overview.
