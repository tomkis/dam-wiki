# Wiki log

Chronological, append-only log of what happened and when. Newest entry last.
Entry format:

`## [YYYY-MM-DD] <onboard|ingest|lint|query> | <subject>`

## [2026-06-19] onboard | initialise dam codebase wiki

Onboarded wiki for dam-agents/dam architecture. First eager ingest @c307f40: 18 pages (1 source overview, 6 package summaries, 1 entity, 10 concepts). Watermark set to c307f40. Daily maintenance scheduled (cron `23 6 * * *`, id sched-baa2b2348c98). Flagged for lint: ubiquitous-language.md describes Agent as a ConfigMap with desiredState, which lags the authoritative Agent CRD Go types / platform-topology.md (CRD with spec/status subresources, no desiredState) — surfaced on entities/agent.md.

## [2026-06-19] query | DNS-exfil protection & gateway passthrough
Resolved against source HEAD 64df37d (security-and-credentials.md byte-identical to pinned c307f40). Findings: (1) SNI-miss "passthrough" is NOT an authz bypass — ext_authz gates the catch-all chain on SNI and fails closed, so egress is allowlist-bounded and the gateway never resolves attacker-controlled names. (2) Flagged a Contradiction in the source doc: line 85 ("admits exactly DNS and the paired gateway pod's Envoy port") vs lines 371-373 ("DNS is not admitted ... fails closed") — material to DNS-exfil. Updated security-and-credentials.md: bumped commit to 64df37d, corrected the agent->gateway bullet (was silently asserting DNS-not-admitted while citing the line that says otherwise), added the ext_authz egress-allowlist clarification, added a Contradiction callout for lint.

## [2026-06-22] ingest | dam-agents/dam @9d1bc99

Delta from c307f40. Changed files: Helm chart templates (api-server, controller, keycloak, postgres, redis, UI, NFS provisioner, migrate-stale-netpols); docs/architecture/logging.md (added controller logging section); packages/api-server-api/src/modules/connections/providers.ts (new — single source of truth for provider presets); packages/api-server-api/src/index.ts (re-exports providers.ts); packages/api-server-api/src/modules/secrets/types.ts; packages/cli/src/modules/agent/commands/create-interactive.ts; packages/cli/src/modules/connection/domain/provider-templates.ts (deleted); packages/controller/main.go; packages/ui/src/modules/connections/components/templates-section.tsx; packages/ui/src/modules/connections/lib/provider-templates.ts (deleted); packages/ui/src/modules/providers/components/anthropic/modes.ts; packages/ui/src/modules/providers/components/provider-connect-dialog.tsx; packages/ui/src/modules/providers/components/provider-section.tsx; packages/ui/src/modules/sandboxes/components/connections-section.tsx; packages/ui/src/modules/sandboxes/components/steps/connections-step.tsx; packages/ui/src/modules/sandboxes/hooks/use-sandbox-settings-form.ts; packages/ui/default.conf; pnpm-lock.yaml; package.json.

Pages updated: concepts/logging.md (added controller logging section — Go log/slog to stderr, LOG_LEVEL, no audit trail, zero-code OTel via eBPF); sources/api-server.md (added provider preset catalog to api-server-api section); sources/ui.md (noted PROVIDER_TEMPLATE_IDS import from api-server-api, removal of local provider-templates.ts); sources/cli.md (noted create-interactive.ts imports from api-server-api, removal of local provider-templates.ts). Watermark advanced to 9d1bc99.

## [2026-06-22] lint | staleness sweep @9d1bc99

Links: clean (0 orphans, 0 broken). Staleness: sources/controller.md had packages/controller/main.go in its files list and main.go was modified (added setupLogger log/slog). Refreshed: added logging section to controller.md (log/slog to stderr, LOG_LEVEL, no audit trail, no in-process OTel). Bumped commit to 9d1bc99. All other pages at c307f40 whose files list did not overlap the delta were left untouched. No contradictions found. Coverage gap noted: Helm chart template changes (deploy/helm/platform/**) are not reflected in any wiki page — deploy topology is undocumented; candidate for a future infrastructure/deployment concept page.
