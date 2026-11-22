# Slack connector recipe

Service: Slack — Slack Web API
Purpose: read channel history and user lists, and post messages as a bot in a workspace the user controls, without opening the Slack UI.
Verified against Slack's official API documentation on 2026-11-23.

**Status: 条件付き (conditional)** — the Web API itself is fully buildable with an issued bot token (see below). Registering Slack through Muse's custom-connector OAuth flow is **未確認 (unverified)**: the endpoints sit on one host, which removes the known Google-style blocker, but the registration has not actually been attempted.

## API facts

- API host: `slack.com` — every Web API method is an HTTP RPC-style call at `https://slack.com/api/METHOD_FAMILY.method` (200+ methods). Responses are JSON with a top-level `ok` boolean. Authenticate with the token in an `Authorization: Bearer <token>` header.
- Auth: OAuth 2.0, via a Slack app created at `api.slack.com/apps`. Slack apps use OAuth to manage API access; installing the app is what issues the token.
  - Authorize: `https://slack.com/oauth/v2/authorize` — `scope` requests bot-token scopes, `user_scope` requests user-token scopes. The returned authorization `code` is single-use and expires after 10 minutes.
  - Token: `https://slack.com/api/oauth.v2.access` — exchange the code (with `client_id` and `client_secret`) for the access token. Note this token endpoint is itself a Web API method on the same host, not a separate OAuth domain.
  - Token types:
    - **Bot token (`xoxb-…`)** — tied to the app's bot user, not to any person. Granted once per workspace installation; the same bot token results no matter which user installed the app. This is the token most apps use, and the right default for an agent connector.
    - **User token (`xoxp-…`)** — acts on behalf of the authorizing user; there can be several per workspace. Slack's own guidance: assign scopes to the bot token, and use user-token scopes only when user impersonation is genuinely needed.
    - App-level token (`xapp-…`) — for Socket Mode WebSocket connections (`connections:write`); not needed for plain Web API calls.
  - Bot scopes an agent typically needs:
    - `channels:history` — read public-channel history (`conversations.history`); private channels need `groups:history`, DMs need `im:history` / `mpim:history`
    - `channels:read` (and `groups:read` for private) — list and inspect conversations (`conversations.list`, `conversations.info`)
    - `chat:write` — post messages as the bot, in channels the bot is a member of; add `chat:write.public` to post to public channels without joining
    - `users:read` — `users.list` / `users.info`; add `users:read.email` only if email addresses are required
- Key endpoints (all under `https://slack.com/api/`):
  - List conversations: `conversations.list` — returns channel IDs and names visible to the token
  - Read history: `conversations.history` — cursor-paginated (`limit` default 100, max 999 for Marketplace/internal apps); `conversations.replies` fetches one thread
  - Post a message: `POST chat.postMessage` — `channel` (ID preferred) plus `text` or Block Kit `blocks`; `thread_ts` replies in a thread
  - List users: `users.list` — cursor-paginated; resolve display names from message `user` IDs with `users.info`
  - Sanity check: `auth.test` — confirms which workspace/user a token belongs to
- Rate limits (per method, per workspace, per app; windows are per minute; all Slack plans get the same tier per method):
  - Tier 1: 1+ per minute · Tier 2: 20+ per minute · Tier 3: 50+ per minute · Tier 4: 100+ per minute. Each method's reference page states its tier (`conversations.list` is Tier 2; paginating collection methods such as history/user lists are typically Tier 3). Slack does not publish exact burst limits.
  - `chat.postMessage` is a special tier: generally one message per second per channel, plus a workspace-wide limit.
  - Exceeding a limit returns HTTP 429 with a `Retry-After` header (seconds) — honour it; only that method/workspace combination is throttled.
  - **History tightening (2025-05-29):** for `conversations.history` / `conversations.replies`, newly created *unlisted, commercially distributed* (non-Marketplace) apps — and new installations of existing ones — drop to 1 request per minute and at most 15 objects per request. Internal customer-built apps keep their existing (Tier 3) limits, per Slack's 2025-06-03 clarification. An app built for your own workspace is internal; do not distribute it if you want to keep the higher limits.

## Human setup (once)

1. At `api.slack.com/apps`, create a new app (from scratch) and pick the workspace. Any Slack user can create an app and install it to a workspace they have install permission for — a personal or small workspace where you are owner/admin is the straightforward case.
2. In **OAuth & Permissions**, add the Bot Token Scopes listed above. Keep them minimal; every scope is shown on the install consent screen.
3. Click **Install to Workspace** and authorize. Copy the **Bot User OAuth Token** (`xoxb-…`) from the same page. Changing scopes later requires a reinstall before they take effect.
4. Invite the bot into every channel it must read or post in (e.g. `/invite @your-app` in the channel). A bot token sees nothing in channels the bot has not joined.
5. If the workspace has the **Admin-Approved Apps** feature enabled, a Workspace Admin must approve the app before it can be installed; some restricted scopes can only be granted by a workspace admin or owner. On a workspace you administer yourself this is a non-issue; on someone else's workspace, expect an approval request step.

## Known blocker (Muse custom connectors, 2026-11-23)

Muse has no official Slack connector, so this is a custom-connector candidate. The known Muse registration wall (see `field-notes/muse-oauth-single-domain-rule.md`) is that the OAuth authorize and token endpoints must sit on one issuer domain — Google fails this because `accounts.google.com` and `oauth2.googleapis.com` differ. Slack does **not** split hosts: authorize is `slack.com/oauth/v2/authorize` and the token exchange is `slack.com/api/oauth.v2.access`, both on `slack.com`. So the cross-domain failure does not apply on host grounds.

What remains **未確認 (unverified)**: whether Muse's flow accepts Slack's token step in practice. `oauth.v2.access` is a Web API method rather than a conventional token endpoint — it takes `client_id`, `client_secret` and `code` as form parameters and returns the bot token at the top level of a JSON body (user tokens nested under `authed_user`), and Slack tokens do not expire or refresh unless the app explicitly enables token rotation. None of this has been tried against Muse's registration, so do not promise the OAuth route works there.

The route that follows the established buildable pattern today: have the human complete steps 1–3 above in Slack's own UI and hand over the **already-issued bot token (`xoxb-…`)**, then call `https://slack.com/api/…` with it as a bearer token. No OAuth flow runs at call time.

A related official option, for completeness: Slack publishes its own MCP server, but per Slack's documentation only Marketplace-published apps and internal apps may use it — unlisted apps are prohibited. Whether it can be attached to Muse is 未確認 (unverified).

## Pitfalls

- **Channel membership is the real permission gate.** Scopes alone are not enough: with a bot token, `conversations.history` returns only channels the bot has joined, and `chat:write` posts only where the bot is a member (unless `chat:write.public` was granted). A `not_in_channel` / empty-history result usually means "invite the bot", not "add a scope".
- **Scope changes need a reinstall.** Adding a scope in the app settings changes nothing for already-issued tokens; reinstall the app to the workspace, then copy the token again. `missing_scope` errors name the scope that is absent.
- **Do not distribute the app.** The moment an app for your own workspace becomes an unlisted app installed elsewhere, the 2025 history limits (1 request/minute, 15 objects) apply to new installs. Keep it internal.
- **Free-plan workspaces hide old history.** On free teams, messages beyond the free message limit are not returned (`is_limited` in the `conversations.history` response); an incomplete archive is a plan limit, not an API failure.
- **Prefer the bot token over a user token.** A user token (`xoxp-…`) reads and posts *as that person*, including channels and DMs the bot was never invited to. That breadth is exactly why Slack steers apps to bot tokens; only take a user token when the task genuinely requires acting as the user.
- **Paginate properly.** `conversations.history`, `conversations.list` and `users.list` are cursor-paginated; calling them without pagination both returns partial data and attracts stricter rate limiting. Follow `response_metadata.next_cursor` until it is empty.
