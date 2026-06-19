---
source: dam-agents/dam
commit: c307f40480aa96788cb9c4f6a06f7e5b732e5bd7
files: [docs/architecture/channels.md, packages/api-server/src/modules/channels/services/channel-manager.ts, packages/api-server/src/modules/channels/infrastructure/telegram.ts, packages/api-server/src/modules/channels/infrastructure/slack.ts]
updated: 2026-06-19
---

# Channels

A **channel** is a messenger surface — Slack or Telegram today — that lets users drive an [Agent](../entities/agent.md) from outside the web UI. Channels are pluggable adapters that live *inside* the api-server process: there is no separate Deployment and no sidecar in the agent pod `docs/architecture/channels.md:7 @c307f40`. Each adapter (the *worker*) owns its inbound socket, its outbound API, and its thread-to-session bookkeeping; a `ChannelManager` service composes the workers and reacts to lifecycle events on the in-process event bus `docs/architecture/channels.md:7 @c307f40`.

Channel bindings are **1:1 with Agent**: a Slack channel may be bound to at most one Agent globally, and Agent delete or messenger disconnect releases the binding `docs/architecture/channels.md:9 @c307f40`.

## The structural axis: platform vs. per-Agent

Channels split along an axis that has real consequences for secrets and identity `docs/architecture/channels.md:11 @c307f40`:

- **Platform channel** — one app serves the whole install. The operator configures it once via Helm values; per-Agent config is just *which conversation this Agent listens to*. Identity linking ties messenger users to Keycloak subs at the workspace level. **Slack** is the platform channel today `docs/architecture/channels.md:13 @c307f40`.
- **Per-Agent channel** — each Agent brings its own bot, created by the Agent owner. The platform learns the bot token from the UI at connect time and stores it in a per-`(Agent, channel-type)` k8s Secret. Authorization is per-thread, not per-user, because there is no workspace to anchor identity in. **Telegram** is the per-Agent channel today `docs/architecture/channels.md:14 @c307f40`.

The channel module models this difference explicitly so future per-Agent channels (WhatsApp Business, Discord, SMS) follow the Telegram pattern mechanically `docs/architecture/channels.md:213 @c307f40`.

## Inbound vs. outbound: two different paths

Inbound and outbound traffic take different paths `docs/architecture/channels.md:16 @c307f40`:

- **Inbound** is *push* from the messenger into the api-server worker, which routes the message to the agent pod over ACP.
- **Outbound** is *pull* initiated by the agent: the harness calls a tool on the api-server's per-Agent MCP endpoint, and the api-server delegates to the right worker.

## Adapters

Both workers implement the same internal contract — `start`, `stop`, `stopAll`, `listConversations`, `postMessage` — keyed by agent id `docs/architecture/channels.md:100 @c307f40`. The contract is the worker interface in code `packages/api-server/src/modules/channels/infrastructure/telegram.ts:56 @c307f40`. The differences are transport, identity model, and where the bot token comes from.

### Slack — platform channel

- **Transport.** Socket Mode: one workspace-level WebSocket from the api-server to Slack. The api-server needs no inbound network access — events arrive over the socket the api-server itself opened. Slack caps Socket Mode at ten concurrent connections per app, which is the install-level scale ceiling for Slack `docs/architecture/channels.md:104 @c307f40`.
- **Token provenance.** App-Level Token (`xapp-…`) and Bot Token come from Helm values, set at install time, living in api-server env — not stored per-Agent `docs/architecture/channels.md:105 @c307f40`.
- **Identity linking.** A `/platform login` slash command starts a Keycloak OAuth flow; on callback the api-server stores `slack_user_id ↔ keycloak_sub`. All subsequent interactions require a linked identity; unlinked users get an ephemeral prompt to log in. The link table is the source of truth for "who is this Slack user in Platform terms" `docs/architecture/channels.md:106 @c307f40`.
- **Access control — two tiers.** *Channel membership* is the coarse gate (users must be in the Slack channel to see the bot). *Per-Agent allowed users* is the fine gate — each Agent optionally declares the subs that may *trigger* work; non-listed users in the channel still see responses but cannot drive a session `docs/architecture/channels.md:107 @c307f40`. Combined with foreign-replier forking, this lets one thread have multiple authorized drivers whose actions land under their own identities.
- **Agent selection per thread.** On a post, the worker checks which Agents the user can access in that channel. One match → route directly. Multiple matches → emit an `external_select` block that lazy-loads from the api-server. The selected Agent is stored as `thread_ts → agent_id` in memory; once a thread is bound, every subsequent message goes to the same Agent `docs/architecture/channels.md:108 @c307f40`.

### Telegram — per-Agent channel

- **Transport.** Long-poll `getUpdates`. Each Agent has its own bot, so the api-server runs one Telegram client per active Telegram-connected Agent `docs/architecture/channels.md:112 @c307f40`.
- **Token provenance.** The Agent owner creates a bot via `@BotFather` and pastes the token in the UI. The api-server writes it to a k8s Secret named `platform-channel-telegram-<agentId>` labeled `agent-platform.ai/type=channel-secret`. The token never round-trips to the UI on reads and never traverses the in-process event bus — `TelegramConnected` carries only `agentId`, and the worker reads the token from the Secret store at start time `docs/architecture/channels.md:113 @c307f40`.
- **Identity model — there is none.** Telegram has no workspace to anchor a user-to-Keycloak link. Authorization shifts from user to *conversation*: a thread (DM or group) is unauthorized until someone runs `/login` and completes a Keycloak OAuth flow, recording the thread in `telegram_threads`; `/logout` revokes. In groups, only chat admins may `/login` (verified via `getChatMember`); unauthorized groups stay silent so the bot does not spam every chat it has been added to `docs/architecture/channels.md:114 @c307f40`.
- **Lifecycle.** Connect = create Secret + emit `TelegramConnected`; the `ChannelManager` subscription reads the token from the Secret store and starts the worker. Disconnect = stop worker + delete Secret. Agent deletion cascades via label selector. No bot token at rest in Postgres `docs/architecture/channels.md:115 @c307f40`.

### Shared lifecycle plumbing

`SlackConnected` / `SlackDisconnected` / `TelegramConnected` / `TelegramDisconnected` / `AgentDeleted` events on the rxjs bus drive `start` / `stop` calls on the appropriate worker `docs/architecture/channels.md:117 @c307f40`. The `ChannelManager` subscribes to each of these event types `packages/api-server/src/modules/channels/services/channel-manager.ts:89 @c307f40`. Bootstrap on api-server startup walks the per-Agent channel config and calls `start` for each enabled channel `docs/architecture/channels.md:117 @c307f40`. See [agent-lifecycle](../concepts/agent-lifecycle.md) for the pod side of these events.

## Inbound — channel message to ACP session

A messenger event is delivered to the worker (Socket Mode for Slack, long-poll for Telegram); the worker runs identity/auth checks, then resolves the thread to a session over ACP `docs/architecture/channels.md:130 @c307f40`. Key behaviors the happy path glosses over:

- **Identity gates differ per adapter.** Slack runs the linked-identity check, the per-Agent allowed-users check, and the owner-vs-foreign decision (the latter selects whether the relay targets the main pod or a fork Job). Telegram runs the per-thread `/login` check; there is no foreign fork because there is no workspace identity to fork under `docs/architecture/channels.md:151 @c307f40`.
- **Wake is implicit.** The relay step is the same `ACP relay → wake-if-hibernated → forward` path the UI uses. Channels never call lifecycle endpoints directly; routing an ACP frame is what wakes the pod `docs/architecture/channels.md:152 @c307f40` (see [agent-lifecycle](../concepts/agent-lifecycle.md), §Wake).
- **Resume vs. new is decided by the ACP session list.** The binding lives on the session itself: the worker lists sessions over ACP and resumes the one whose `_meta.platform.threadTs` matches. If `unstable_resumeSession` fails (PVC lost, session expired), the worker falls back to creating a new session with thread history injected from the messenger API — degrading gracefully, no regression `docs/architecture/channels.md:153 @c307f40`. The Telegram worker logs and starts fresh on resume failure `packages/api-server/src/modules/channels/infrastructure/telegram.ts:180 @c307f40`.
- **`threadKey` is adapter-specific.** Slack uses `thread_ts`; Telegram uses chat id. It is carried on the session as `_meta.platform.threadTs` and matched in-process; there is **no DB uniqueness guard**, so two concurrent first messages in a brand-new thread can mint two sessions for the same key `docs/architecture/channels.md:154 @c307f40`.
- **Turn relays emit `ChannelTurnRelayed`.** After the ACP turn finishes, both workers emit a `ChannelTurnRelayed` event on the bus carrying `channel`, `agentId`, `actorSub` (the relaying user's Keycloak `sub`, or `null` on Telegram), and `outcome` (`"success" | "failure"`) `docs/architecture/channels.md:155 @c307f40`. The Slack worker emits it `packages/api-server/src/modules/channels/infrastructure/slack.ts:302 @c307f40` and the Telegram worker emits it `packages/api-server/src/modules/channels/infrastructure/telegram.ts:201 @c307f40`. Usage tracking consumes it; the forks subsystem also subscribes to drive paired-pod teardown on Slack `docs/architecture/channels.md:155 @c307f40`.

### Thread-session binding

A thread maps to one resumable session, giving the agent real conversational continuity. The binding *is* the session's own `_meta.platform.threadTs`, resolved by listing sessions over ACP and matching — there is no server-side session store `docs/architecture/channels.md:21 @c307f40`.

## Outbound — agent to channel

Outbound is initiated by the agent process. The harness calls a tool on the api-server's per-Agent MCP endpoint (`POST /api/agents/{id}/mcp`); the endpoint authenticates the call and the channel manager routes the message back through the right worker `docs/architecture/channels.md:159 @c307f40`.

What the agent sees:

- **Two tools** are registered on the per-Agent MCP server. `describe_channel` returns the authorized chats (DMs / threads / groups) for a given channel type; `send_channel_message` posts text to a chat. The agent picks the channel by argument (`slack` or `telegram`); `chatId` addresses a specific chat. Omitting `chatId` resolves per channel type: Telegram posts to the worker's last-active thread (error if none); Slack posts to the single channel bound to the Agent and rejects a `chatId` that isn't that bound channel `docs/architecture/channels.md:190 @c307f40`.
- **Tools are always registered.** Calls are rejected at invocation time when no channel is connected — no dynamic tool list, no per-session toggle `docs/architecture/channels.md:191 @c307f40`.
- **Bidirectional.** If a channel is connected, *every* session on that Agent can post — interactive and scheduled alike. There is no per-session outbound flag `docs/architecture/channels.md:192 @c307f40`.

Why the dedicated MCP endpoint:

- **Network isolation.** The MCP port is the only api-server port the agent's NetworkPolicy admits. The agent cannot reach the admin API (tRPC, OAuth, agent management) — only this endpoint `docs/architecture/channels.md:196 @c307f40`.
- **Auth without an admin login.** Caller identity is derived from the source pod IP, mapped to a `platform.ai/agent` label via the api-server's `podIpResolver` cache. The agent presents no Bearer token — a compromised harness can't claim to be a different Agent because the kernel-verified source IP is the source of truth. Owner match (`caller.owner == agent.owner`) is the second check `docs/architecture/channels.md:197 @c307f40`. See [security-and-credentials](../concepts/security-and-credentials.md).
- **Direct path to channel infra.** The MCP endpoint dispatches into the same `ChannelManager.postMessage` workers use internally — no agent-runtime round-trip, no second relay hop `docs/architecture/channels.md:198 @c307f40`.

### Threading model

Outbound posts are **fire-and-forget at the thread level**. The agent posts a top-level message; the worker stores no `threadTs → sessionId` mapping for proactive posts. If a user replies, the inbound path treats it as a *new mention* — a fresh session, with no continuity from the originating session. This is a deliberate trade-off: keep outbound simple and stateless at the cost of session bridging on Slack-side replies `docs/architecture/channels.md:202 @c307f40`.

The two messengers diverge on what a top-level post means: Slack posts to the channel id (or last-active channel) with no `thread_ts`, producing a new top-level message — a reply is a new mention. Telegram has no thread primitive in DMs and only weak threading in groups, so the worker posts to the chat id; if the prompt was triggered by a previous message in the same chat, that chat is still the conversation `docs/architecture/channels.md:208 @c307f40`.

## Foreign replier fork (Slack only)

Slack's two-tier access admits multiple authorized users into one thread. **Owner** replies relay to the main pod; replies from any *other* authorized user **fork** into a per-turn paired pod set — a fork agent Job and a fork gateway Pod, each with its own NetworkPolicy — whose gateway mounts the replier's K8s credential Secrets `docs/architecture/channels.md:20 @c307f40`. The pair spec, foreign-credential selection, and shared-PVC mechanics are owned by [security-and-credentials](../concepts/security-and-credentials.md); channels just see "main pod or fork pod" at the relay step `docs/architecture/channels.md:20 @c307f40`. Telegram is single-track — the main pod handles every turn, no fork `docs/architecture/channels.md:96 @c307f40`.

## Persistence touchpoints

Channels touch two stores `docs/architecture/channels.md:217 @c307f40`:

- **Identity-link tables (Postgres).** `identity_links` is keyed on `(provider, external_user_id)` and maps to `keycloak_sub` — Slack populates it today, but the `provider` column makes it reusable for any future workspace channel. `telegram_threads` records per-conversation authorization for Telegram. Different shapes by design — Slack has a workspace, Telegram does not `docs/architecture/channels.md:219 @c307f40`.
- **Channel Secrets (k8s).** `platform-channel-telegram-<agentId>` per Telegram-connected Agent. Slack has none — its tokens live in api-server env from Helm values `docs/architecture/channels.md:220 @c307f40`.

Channels do **not** participate in the Agent ConfigMap spec/status split. An earlier design kept channel config in the Agent ConfigMap; that was superseded — channel routing metadata lives in Postgres, secrets in k8s Secrets, Agent ConfigMaps stay channel-free `docs/architecture/channels.md:222 @c307f40`.

## See also

- [api-server](../sources/api-server.md) — the process that hosts the channel workers and the MCP endpoint.
- [Agent](../entities/agent.md) — what a channel binds to (1:1).
- [agent-lifecycle](../concepts/agent-lifecycle.md) — wake-on-relay and the lifecycle events that start/stop workers.
- [security-and-credentials](../concepts/security-and-credentials.md) — IP-based MCP auth and the foreign-replier fork pair.
- [connections](../concepts/connections.md) — how external integrations attach to an Agent.
