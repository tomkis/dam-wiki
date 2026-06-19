---
source: dam-agents/dam
commit: 64df37d1d3d6e4dd337e55733a6a83e24e86c515
files: [docs/architecture/security-and-credentials.md]
updated: 2026-06-19
---

# Security & Credentials

DAM runs untrusted agent code in the cloud and must let that code reach real
upstream services (GitHub, Anthropic, Slack) without ever handing it the
credentials to do so. The whole security model exists to resolve that tension.
Three rules carry it: agents never hold upstream credentials, identity flows
from Keycloak, and two enforcement boundaries are layered — a kernel
NetworkPolicy on the agent and a mesh AuthorizationPolicy on the gateway
(`docs/architecture/security-and-credentials.md:7-38 @c307f40`).

## The trust model: the pod is the credential boundary

The credential boundary is the pod, not the process. Real upstream tokens live
in Kubernetes Secrets and are mounted **only** into a paired *gateway* pod; the
agent pod never mounts Secret bytes
(`docs/architecture/security-and-credentials.md:9-13 @c307f40`,
`docs/architecture/security-and-credentials.md:77-79 @c307f40`). An Envoy proxy
inside the gateway pod injects the credentials into outbound traffic on the
wire, so the agent gets the *effect* of authenticated egress without ever seeing
the secret (`docs/architecture/security-and-credentials.md:9-13 @c307f40`).

Two structural facts close the obvious bypasses: the agent pod has no service
account token (`automountServiceAccountToken: false`), and there is no
co-located sidecar sharing its network or PID namespace
(`docs/architecture/security-and-credentials.md:104-106 @c307f40`). The agent
runs untrusted code and is treated as hostile; the gateway is
platform-controlled (`docs/architecture/security-and-credentials.md:409-414 @c307f40`).

Workspace contents are explicitly **outside** the trust boundary — see the
persistence security note (`docs/architecture/security-and-credentials.md:40-41 @c307f40`).

This pairing of an agent pod with a gateway pod is the unit of the platform's
topology — see [platform topology](platform-topology.md) and the
[agent](../entities/agent.md) entity.

## Identity: Keycloak is the only authority

Keycloak runs in-cluster as a Helm subchart and is the OIDC provider for every
authenticated surface (`docs/architecture/security-and-credentials.md:110-112 @c307f40`).
The browser user flow:

1. The browser authenticates against Keycloak and obtains a JWT with audience
   `platform-api` (`docs/architecture/security-and-credentials.md:114-115 @c307f40`).
2. The UI sends that JWT to the api-server on every tRPC and ACP call; the
   api-server validates it against Keycloak's JWKS
   (`docs/architecture/security-and-credentials.md:116-117 @c307f40`).
3. The JWT's `sub` claim becomes `agent-platform.ai/owner=<sub>` on every
   resource the user creates — Agent CR, credential Secret, and so on
   (`docs/architecture/security-and-credentials.md:118-120 @c307f40`).

There is no token exchange: credential storage is K8s-native and label-scoped,
so the api-server enforces ownership directly on read and write
(`docs/architecture/security-and-credentials.md:122-124 @c307f40`).

For headless / CI use, the CLI passes a long-lived **API key** in the same
`Authorization: Bearer` slot, distinguished by a `pk_` prefix. Keys carry the
owner's `sub`, a subset of permission scopes, and an optional agent allowlist;
the bearer middleware dispatches by prefix and produces the same downstream
authenticated-principal shape (`docs/architecture/security-and-credentials.md:126-132 @c307f40`).
API keys cannot mint or revoke other API keys — the management surface rejects
any request whose principal was authenticated via a key, so an exfiltrated key
cannot escalate (`docs/architecture/security-and-credentials.md:132-134 @c307f40`).

### Keycloak as audit source

Keycloak also doubles as an audit event source. Its built-in `jboss-logging`
event listener emits login and admin events to pod stdout, riding the same
cluster log pipeline out to the external log service; successes log at `info`,
errors at `warn` (`docs/architecture/security-and-credentials.md:138-145 @c307f40`).
Persistence is split by event class to keep secrets out of the database and
spare Postgres high-volume writes: **login events** are not written to the DB at
all — the external log service is the audit source of truth
(`docs/architecture/security-and-credentials.md:149-154 @c307f40`). **Admin
events** record who-acted/what/where metadata to Postgres (low volume) but
`adminEventsDetailsEnabled` is off, so the full request body — which could carry
plaintext credentials on user-create/update — is never stored
(`docs/architecture/security-and-credentials.md:156-165 @c307f40`).

## Resource ownership: soft multi-tenancy

Multi-tenancy is **soft** — a single Kubernetes namespace, with an
`agent-platform.ai/owner` label carrying the authenticated `sub` on every owned
resource. The api-server is the sole writer of resource spec, stamps the label
on create, and filters every list and get by it; there is no namespace-per-user
(`docs/architecture/security-and-credentials.md:172-176 @c307f40`).

Per-user credential isolation *is* that label. The controller picks credentials
per-Agent by listing Secrets labelled
`agent-platform.ai/owner=<sub>,agent-platform.ai/managed-by=api-server`, then
mounting only the matching set into the paired gateway pod
(`docs/architecture/security-and-credentials.md:178-182 @c307f40`). Cross-owner
leakage is structurally prevented by the selector — a Secret with a missing
owner label is treated as having no owner and is never mounted
(`docs/architecture/security-and-credentials.md:180-182 @c307f40`).

## Credential storage

Each connected service produces one K8s Secret per `(owner, connection)`
(`docs/architecture/security-and-credentials.md:186 @c307f40`). See
[connections](connections.md) for how those bindings are modelled.

- **OAuth-issued tokens** (GitHub, MCP servers, Generic OAuth apps) — the
  api-server's `/api/oauth/callback` writes the access + refresh token pair plus
  a structured host list of every wire position the token should be injected on.
  A refresh loop re-mints access tokens before expiry; the agent never sees the
  refresh token (`docs/architecture/security-and-credentials.md:188-193 @c307f40`).
- **User-supplied secrets** (Anthropic API keys, generic API tokens) — written
  by the secrets module with the same labels and annotations
  (`docs/architecture/security-and-credentials.md:194-195 @c307f40`).
- **GitHub PATs** — one PAT becomes *two* `generic` Secrets sharing a display
  name. The `api.github.com` half stores the raw PAT, injects
  `Authorization: Bearer {value}`, and projects `GH_TOKEN` into the agent env for
  the `gh` CLI; the `github.com` half stores `base64("x-access-token:" + PAT)`
  and injects `Authorization: Basic {value}` for `git clone` over HTTPS. Both are
  written atomically via `secrets.createGithubPat`, rolling back the api half if
  the git half fails (`docs/architecture/security-and-credentials.md:196-205 @c307f40`).

**Multi-host connections.** A single OAuth connection can inject the same token
on more than one host with **different auth schemes per host**, all from one
Secret. A JSON `agent-platform.ai/injection-hosts` annotation lists each
`{host, headerName?, valueFormat?, encoding?, pathPattern?}` tuple; the
controller fans the Secret into one Envoy filter chain per host and drives the
egress allowlist from the same list — there is no second source of truth
(`docs/architecture/security-and-credentials.md:207-217 @c307f40`). GitHub.com
is the motivating case: the same token must reach `api.github.com` and
`raw.githubusercontent.com` as `Bearer` and `github.com` as `Basic` for
`git clone` (`docs/architecture/security-and-credentials.md:219-224 @c307f40`).

### Image pull credentials are a separate class

Pulling the agent's container image from a private registry uses a structurally
separate credential class that does **not** ride the Envoy path. It is a
`kubernetes.io/dockerconfigjson` Secret referenced from the pod spec's
`imagePullSecrets`; the kubelet — not Envoy — reads it at pod creation
(`docs/architecture/security-and-credentials.md:233-244 @c307f40`). It is
**agent-scoped**, not owner-scoped: one Secret per Agent, created and torn down
with the Agent, with no cross-agent reuse
(`docs/architecture/security-and-credentials.md:245-249 @c307f40`). A per-agent
ref takes precedence over an install-wide default by being listed first on
`imagePullSecrets`, with the default retained as fallback — override, not
replace (`docs/architecture/security-and-credentials.md:250-255 @c307f40`).
Scope is long-lived static credentials; dynamically-minted tokens like ECR are
out of scope (`docs/architecture/security-and-credentials.md:266-267 @c307f40`).

## Envoy credential injection on the wire

The controller renders a per-Agent Envoy bootstrap ConfigMap and a cert-manager
Certificate whose Secret holds the leaf TLS material the gateway uses to
terminate the agent's egress TLS. The leaf comes from a chart-managed
`platform-mitm-ca-issuer` ClusterIssuer; the CA cert is mounted into the agent
at `/etc/platform/ca/ca.crt` (the private `tls.key` stays in the gateway) so the
agent's TLS clients trust Envoy's intercept cert
(`docs/architecture/security-and-credentials.md:271-277 @c307f40`).

On the wire (`docs/architecture/security-and-credentials.md:279-302 @c307f40`):

1. The agent sets `HTTPS_PROXY=http://<agent>-gateway:<envoyPort>`; the per-Agent
   gateway Service routes every egress to the paired gateway pod as HTTP CONNECT.
2. Envoy's outer listener (bound on `0.0.0.0`, reach gated by NetworkPolicy)
   terminates the CONNECT and routes the inner stream to an internal listener
   that reads SNI.
3. Per-host filter chains terminate TLS with the leaf cert, run the credential
   injector(s) to add header(s) (or rewrite a URL query param), then forward to a
   per-chain `STRICT_DNS` cluster pinned to the host with explicit upstream SNI +
   SAN-bound TLS validation. The agent's inner `Host` header has no influence on
   the upstream destination — **the route-confusion exfiltration path is
   structurally closed**.
4. The default chain (SNI miss) does TCP passthrough.

Hosts the api-server has issued a credential for surface as L7 chains (header
injection); hosts without a credential surface as L4 passthrough
(`docs/architecture/security-and-credentials.md:300-302 @c307f40`). A single
host can carry multiple ordered `credential_injector` filters — two different
credentials, or the same credential injected into both a header and a URL query
parameter via a follow-up Lua filter that strips the carrier header before it
reaches the upstream (`docs/architecture/security-and-credentials.md:304-313 @c307f40`).

## Human-in-the-loop: ext_authz

Each credentialed request makes an ext_authz `Check` call against the
api-server. The caller's identity is the **per-Agent ext-authz Service**
(`<release>-extauthz-<id>`) the gateway's Envoy was configured to dial; an
AuthorizationPolicy on each Service ALLOWs only the matching SA principal, so by
the time a Check arrives the calling Agent is already proven cryptographically
(`docs/architecture/security-and-credentials.md:317-323 @c307f40`). The handler
parses the Agent ID from the gRPC `:authority`, finds the matching egress rule,
and either allows, denies, or **holds the request open** while the user renders
a verdict in the inbox (`docs/architecture/security-and-credentials.md:323-325 @c307f40`).
`failure_mode_allow: false` means a blocked Check fails closed — the agent gets
403 with no inbox prompt (`docs/architecture/security-and-credentials.md:326-328 @c307f40`).
This gate is how [skills](skills.md) that touch credentialed hosts get
surfaced to a human.

**ext_authz is the destination-side egress allowlist — passthrough is not a
bypass.** The L4 "default chain (SNI miss) does TCP passthrough — the request
reaches the upstream unchanged" line describes only that TLS is *not* terminated
and *no* credential is injected on that chain; it does **not** mean the request
is unauthorized. ext_authz "gates everything the gateway forwards on behalf of
the agent … this is the destination-side egress gate"
(`docs/architecture/security-and-credentials.md:93-96 @64df37d`), and it runs on
the catch-all chain too — "the network filter on the catch-all chain sees SNI
only" (`docs/architecture/security-and-credentials.md:330-331 @64df37d`). So an
unknown SNI hits ext_authz, finds no matching egress rule, and fails closed
(403) under `failure_mode_allow: false`
(`docs/architecture/security-and-credentials.md:326 @64df37d`); the gateway
never connects to or resolves it. The allowlist itself is driven from the
per-host `injection-hosts` list (one `connection:<id>` egress rule per host),
"there is no second source of truth"
(`docs/architecture/security-and-credentials.md:216-217 @64df37d`). Net: the
gateway will not resolve an attacker-controlled name for the agent, so the
gateway-side DNS-resolution exfil channel is closed — **subject to the agent
having no DNS path of its own, which the Contradiction above leaves unresolved.**

## The two layered boundaries

The agent and the gateway sit on opposite sides of the credential boundary, so
they are gated by different mechanisms matched to different threat models
(`docs/architecture/security-and-credentials.md:352-355 @c307f40`,
`docs/architecture/security-and-credentials.md:409-414 @c307f40`):

- **Agent → paired gateway: kernel.** The per-pair `<id>-agent-egress`
  NetworkPolicy is the sole gate. The agent pod opts out of ambient mesh
  (`istio.io/dataplane-mode: none`) so the kernel sees real destination IPs
  rather than HBONE tunnelled to ztunnel; the policy admits the paired
  gateway pod's Envoy port, and HBONE 15008 is not admitted — the agent never
  speaks it. Pair pinning is structural: the pod-selector is the gateway pod
  itself, so a compromised agent has no admitted IP-and-port to reach anything
  else (`docs/architecture/security-and-credentials.md:81-86 @64df37d`,
  `docs/architecture/security-and-credentials.md:368-378 @64df37d`).

> **Contradiction:** the source doc disagrees with itself on whether the agent
> egress NetworkPolicy admits **DNS (port 53)** — which is the crux of DNS-exfil
> protection. The layered-enforcement bullet says the policy "admits exactly DNS
> and the paired gateway pod's Envoy port"
> (`docs/architecture/security-and-credentials.md:85 @64df37d`), whereas the
> per-boundary detail says "DNS is not admitted — the agent addresses its gateway
> by ClusterIP … anything in the pod that tries to resolve names directly fails
> closed" (`docs/architecture/security-and-credentials.md:371-373 @64df37d`).
> These cannot both hold: if 53 is admitted the agent can reach cluster DNS and
> tunnel data out via crafted queries (a path that never touches the Envoy/
> ext_authz gate, since DNS does not ride `HTTPS_PROXY`); if it is not, the
> gateway is the only resolver and that channel is closed. Both statements carry
> the same `Last verified: 2026-06-15` date in one file, so neither wins on
> recency — flagged for `lint` to resolve against the rendered NetworkPolicy.
- **api-server / controller → agent: kernel.** The chart-rendered
  `agent-ingress-platform-only` NetworkPolicy admits the agent port only from
  api-server pods (ACP/tRPC relay) and controller pods (idle-checker busy-probe).
  agent-runtime serves unauthenticated on the assumption that this kernel gate is
  the auth boundary; kubelet probes are node-originated and unaffected
  (`docs/architecture/security-and-credentials.md:87-92 @c307f40`,
  `docs/architecture/security-and-credentials.md:379-385 @c307f40`).
- **Gateway → api-server (harness + ext-authz): mesh.** The gateway pod stays in
  ambient; istiod stamps it with a SPIFFE workload cert whose SA name equals the
  Agent name. Two per-Agent AuthorizationPolicies enforce this cryptographically:
  the api-server's harness waypoint ALLOWs the gateway principal to
  `/api/agents/<id>/*`, and the per-Agent ext-authz Service ALLOWs only the
  matching SA. The agent has no SPIFFE identity in this model
  (`docs/architecture/security-and-credentials.md:19-38 @c307f40`,
  `docs/architecture/security-and-credentials.md:99-102 @c307f40`,
  `docs/architecture/security-and-credentials.md:386-403 @c307f40`).
- **Pod-level DENY** on the api-server pod rejects anything that is neither the
  waypoint SA (harness) nor a per-Agent SA from the agent namespace (ext-authz),
  closing the direct pod-IP bypass
  (`docs/architecture/security-and-credentials.md:404-407 @c307f40`).

Because the agent has no admitted route to Postgres, Redis, Keycloak, or the
harness/ext-authz Services, no NetworkPolicies on those are needed — the absence
of a route is the gate (`docs/architecture/security-and-credentials.md:93-98 @c307f40`).

Per-Agent ServiceAccount lives in the agent namespace with name == Agent ID;
both pods of the long-lived pair run as it, but only the gateway is a mesh
participant. `automountServiceAccountToken` stays false on both — the gateway's
SPIFFE cert is independent of SA-token mounts
(`docs/architecture/security-and-credentials.md:357-367 @c307f40`).

## Per-turn fork pods (Slack foreign replier)

When a user other than the Agent owner replies in a Slack thread, the api-server
creates a Fork CR that the controller materialises into a per-turn paired pod
set (a fork agent Job + fork gateway Pod). Who may drive a thread is decided
upstream by the per-Agent `allowedUsers` gate
(`docs/architecture/security-and-credentials.md:333-340 @c307f40`). The fork's
gateway pod mounts the **replier's** Secrets — selected by
`agent-platform.ai/owner=<replier-sub>`, never the parent owner's — so the
credential boundary is preserved: the fork runs the replier's credentials
(`docs/architecture/security-and-credentials.md:340-349 @c307f40`).

Crucially, fork pairs get their **own** per-fork SA, distinct from the parent's,
with narrow per-fork AuthorizationPolicies. A compromised fork cannot
impersonate the parent on the full `/api/agents/<parent>/*` surface — per-fork
policies admit the fork's gateway SA only to `/api/agents/<parent>/mcp` and to
the parent's ext-authz Service, so the runtime channel stays parent-only and the
parent owner's HITL rules remain the gate
(`docs/architecture/security-and-credentials.md:33-38 @c307f40`,
`docs/architecture/security-and-credentials.md:362-366 @c307f40`,
`docs/architecture/security-and-credentials.md:394-403 @c307f40`).

## Dev-cluster caveat: SVID rotation

Not an architectural property. The local k3s/lima install pins
`DEFAULT_WORKLOAD_CERT_TTL=720h` on istiod and runs a `ztunnel-cert-watchdog`
CronJob that rolls `ds/ztunnel` when it sees expired-cert signatures, absorbing
the race where a sleeping laptop's VM suspend/resume slips past the default 24h
rotation window and stalls every mesh hop. Production deployments configure mesh
PKI separately and get neither knob
(`docs/architecture/security-and-credentials.md:418-429 @c307f40`).

## See also

- [Platform topology](platform-topology.md) — the agent/gateway pod pairing and plane layout
- [Connections](connections.md) — how credential bindings are modelled and issued
- [Skills](skills.md) — capabilities gated through the HITL ext_authz path
- [api-server](../sources/api-server.md) — JWT validation, ownership stamping, ext_authz handler
- [Agent](../entities/agent.md) — the per-Agent CR, SA, and paired-pod lifecycle
