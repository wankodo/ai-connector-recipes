# Discord connector recipe

Service: Discord — Discord API v10
Purpose: let an agent read and send messages, and work with channels and members, in the servers (guilds) a bot has been invited into — and document why a user's personal Discord account cannot be connected instead.
Verified against Discord's official developer documentation on 2026-11-23.

Status: **構築可 (buildable — prospect only; registration as a Muse custom connector is itself unverified)** for the bot-token lane described below. There is no official Muse connector for Discord. Connecting a personal user account is **不可 (not buildable)** — see the blocker.

## API facts

- API name / host: Discord API v10, base `https://discord.com/api/v10`. Real-time events come over the Gateway (WebSocket), reached via `GET /gateway/bot`.
- Auth — two different things, do not confuse them:
  - **Bot token (the agent lane).** Every application gets a bot user, authenticated with the bot token from the app's settings in the Developer Portal, sent as `Authorization: Bot <token>`. Unlike the normal OAuth2 flow, bot users have access to most API routes without a bearer token and can connect to the Gateway. The bot token has no OAuth scopes and no per-call consent; what it can do inside a guild is governed by the bot's role and channel permissions there.
  - **OAuth2 (user lane).** Standard authorization-code / implicit / client-credentials grants for acting on a consenting user's behalf, restricted to the scopes granted.
    - Authorize: `https://discord.com/oauth2/authorize`
    - Token: `https://discord.com/api/oauth2/token`
    - Scopes include `identify`, `email`, `guilds` (list the user's guilds via `/users/@me/guilds`) and `guilds.members.read`. These are identity/membership scopes: they do **not** grant reading or writing channel messages. `messages.read` is for the local RPC server API only, and `dm_channels.read` is only available to approved partners.
  - **Installing a bot is its own OAuth2 flow**: the bot authorization flow uses scope `bot` (add `applications.commands` for slash commands) plus a `permissions` integer, with no `response_type` or `redirect_uri`, because no user access token is returned.
- Key endpoints (agent use):
  - Bot identity check: `GET /users/@me`
  - Guilds: `GET /guilds/{guild.id}`, `GET /guilds/{guild.id}/channels`, `GET /guilds/{guild.id}/members`
  - Channels: `GET /channels/{channel.id}`
  - Messages: `GET /channels/{channel.id}/messages` (history), `GET /channels/{channel.id}/messages/{message.id}`, `POST /channels/{channel.id}/messages` (send), plus edit / delete / reactions under the same `/channels/{channel.id}/messages/{message.id}` path
  - A bot can only call these for guilds it has been added to, and only for channels its role can view.
- Privileged Gateway Intents: Discord gates sensitive data behind three privileged intents, all off by default and enabled per app on the Developer Portal **Bot** page under **Privileged Gateway Intents**, and also requested in the Gateway identify payload (a mismatch raises a `DisallowedIntents` error):
  - `MESSAGE_CONTENT` — the `content`, `embeds`, `attachments`, `components` and `poll` fields in message objects. Without it, a bot still receives message events but those fields arrive empty. **An agent that needs to read what people wrote needs this intent.**
  - `GUILD_MEMBERS` — member join/leave/update events and listing all members of a guild
  - `GUILD_PRESENCES` — online/offline status, activities and platform info
  - Review requirement (per Discord's Privileged Intent Review guide, rules changed 2026-06-10): apps reachable by fewer than 10,000 unique users across all their guilds can simply toggle privileged intents on in the Developer Portal, no application needed — but only enable what the use case actually requires. Once the app passes 10,000 unique users it must apply for access (90 days from notification to apply), and approved apps must reapply annually.
- Rate limits (official Rate Limits topic):
  - Global: all bots can make up to **50 requests per second** across the API (interaction endpoints are exempt). Without an authorization header the limit applies per IP address.
  - Per-route limits exist on top of that, keyed by top-level resources (`channel_id`, `guild_id`, `webhook_id`/`webhook_token`). Discord does not publish the per-route numbers and says rate limits should not be hard-coded: read `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `X-RateLimit-Reset-After` and `X-RateLimit-Bucket` on each response, and on HTTP 429 wait for the `Retry-After` header / `retry_after` body field.
  - Invalid-request limit: IPs making too many invalid requests are automatically and temporarily restricted — currently **10,000 per 10 minutes**. An invalid request is one returning **401, 403 or 429** (429s with `X-RateLimit-Scope: shared` do not count). A bot retry-looping on a dead token can ban its own IP.

## Human setup (once)

1. Any individual with a Discord account can create an application: in the Developer Portal (discord.com/developers/applications) choose **New Application**. Creating the app and its bot is free and does not require a company or a server of your own.
2. On the app's **Bot** page, reset/copy the bot token and store it as a secret — whoever holds it controls the bot.
3. On the same page, under **Privileged Gateway Intents**, enable only the intents the agent needs (for reading message text: **Message Content Intent**).
4. Build the install link (Developer Portal **OAuth2 → URL Generator**, or the Installation page): scopes `bot` and `applications.commands`, with the minimum bot permissions the agent needs.
5. Someone with the **Manage Server** permission in the target server opens the link and authorizes. This is the step an agent cannot do for itself: no invite, no access. Repeat per server.
6. Register the bot token with the connector / runtime that will call `https://discord.com/api/v10` with `Authorization: Bot <token>`.

## Known blocker

**The wall: there is no personal-account lane, and faking one is prohibited.** Automating a normal user account with its user token — a "self-bot" — is forbidden by Discord: its official support article "Automated User Accounts (Self-Bots)" states that automating normal user accounts outside of the OAuth2/bot API "is forbidden, and can result in an account termination if found." So an agent can never read the servers, DMs and group DMs the *user* belongs to, and must never ask for or use a user token. Everything an agent can legitimately do runs through a bot user, and a bot only exists where it has been invited: **servers the user merely belongs to are out of reach unless someone with Manage Server there adds the bot**, and inside a server the bot sees only the channels its role permits. Bots also cannot accept normal invites, cannot have friends, and cannot be added to or join group DMs.

Muse-specific notes:

- Discord has **no official Muse connector**.
- The buildable pattern is a Muse custom connector that takes the **bot token directly**. Status: **構築可 (見込み・登録自体は未検証)** — buildable in prospect; the registration itself has not been verified.
- The general Muse OAuth rule — registration is declined when the OAuth authorize host and token host differ ("token_url is on a host outside issuer_domain", see the field note `muse-oauth-single-domain-rule`) — does not apply to the token pattern, which uses no OAuth endpoints at all. (For reference, Discord's own OAuth2 authorize and token URLs are both on `discord.com`, so that particular cross-domain wall is not what blocks Discord; the personal-account prohibition above is.)

## Pitfalls

- Asking for a user token, or using a library built for self-bots, risks the user's account being terminated. Refuse that route even if the user offers the token.
- A bot with no `MESSAGE_CONTENT` intent looks like it works — events arrive, sends succeed — while every message body it receives is empty. Check the intent before debugging the code.
- Intent changes in the Developer Portal only take effect on a new Gateway connection: restart the bot after toggling.
- Do not hard-code per-route rate limits; they are unpublished and can change. Honour the `X-RateLimit-*` headers and `retry_after`, and stop calling after a 401 instead of retrying — invalid requests accumulate toward the 10,000-per-10-minutes IP restriction.
- The bot token is a standing credential with no expiry step in the normal flow: keep it in secret storage only, never in code, logs or chat, and reset it in the Developer Portal if it may have leaked.
- Unverified (未確認): the exact set of DM operations a bot can perform with users who share a guild with it, and current per-route bucket sizes (unpublished by design), were not confirmed against the official documentation for this recipe — check the live docs before relying on either.
