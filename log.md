# Wiki log

Chronological, append-only log of what happened and when. Newest entry last.
Entry format:

`## [YYYY-MM-DD] <onboard|ingest|lint|query> | <subject>`

## [2026-06-19] onboard | initialise dam codebase wiki

Onboarded wiki for dam-agents/dam architecture. First eager ingest @c307f40: 18 pages (1 source overview, 6 package summaries, 1 entity, 10 concepts). Watermark set to c307f40. Daily maintenance scheduled (cron `23 6 * * *`, id sched-baa2b2348c98). Flagged for lint: ubiquitous-language.md describes Agent as a ConfigMap with desiredState, which lags the authoritative Agent CRD Go types / platform-topology.md (CRD with spec/status subresources, no desiredState) — surfaced on entities/agent.md.
