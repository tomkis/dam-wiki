# Wiki log

Chronological, append-only log of what happened and when. Newest entry last.
Entry format:

`## [YYYY-MM-DD] <onboard|ingest|lint|query> | <subject>`

## [2026-06-19] onboard | initialise dam codebase wiki

Onboarded wiki for dam-agents/dam architecture. First eager ingest @c307f40: 18 pages (1 source overview, 6 package summaries, 1 entity, 10 concepts). Watermark set to c307f40. Daily maintenance scheduled (cron `23 6 * * *`, id sched-baa2b2348c98). Flagged for lint: ubiquitous-language.md describes Agent as a ConfigMap with desiredState, which lags the authoritative Agent CRD Go types / platform-topology.md (CRD with spec/status subresources, no desiredState) — surfaced on entities/agent.md.

## [2026-06-19] query | DNS-exfil protection & gateway passthrough
Resolved against source HEAD 64df37d (security-and-credentials.md byte-identical to pinned c307f40). Findings: (1) SNI-miss "passthrough" is NOT an authz bypass — ext_authz gates the catch-all chain on SNI and fails closed, so egress is allowlist-bounded and the gateway never resolves attacker-controlled names. (2) Flagged a Contradiction in the source doc: line 85 ("admits exactly DNS and the paired gateway pod's Envoy port") vs lines 371-373 ("DNS is not admitted ... fails closed") — material to DNS-exfil. Updated security-and-credentials.md: bumped commit to 64df37d, corrected the agent->gateway bullet (was silently asserting DNS-not-admitted while citing the line that says otherwise), added the ext_authz egress-allowlist clarification, added a Contradiction callout for lint.
