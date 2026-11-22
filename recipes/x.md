# X connector recipe

Service: X (formerly Twitter) — X API v2
Purpose: post, search and read posts, and look up users on X from an agent, without opening the X web UI.
Verified against X's official developer documentation on 2026-11-23.

Status: **条件付き (conditional)** — there is no official Muse connector for X. A custom connector built on the app-only Bearer Token (public reads) is buildable, but acting as a user (posting included) needs OAuth 2.0 user context, which Muse's custom-connector flow cannot register (see Known blocker), and every call is billed under X's pay-per-use pricing (see Pitfalls). Do not plan on a free, read-everything connector.

## API facts

- API name: **X API v2** (the current version; v1.1 is legacy). Documentation has moved from `developer.x.com/en/docs` to `docs.x.com`, the developer console is at `console.x.com`, and the API itself has not moved.
- API host: `api.x.com` — base URL `https://api.x.com/2`
- Auth — three methods, and which one an endpoint accepts is fixed per endpoint:
  - **OAuth 2.0 Authorization Code Flow with PKCE (user context)** — the route for anything done as a user. Authorize: `https://x.com/i/oauth2/authorize`; Token: `https://api.x.com/2/oauth2/token`. Access tokens last 2 hours; add the `offline.access` scope or you get no refresh token and the user must re-authorize every 2 hours.
  - **App-only Bearer Token (OAuth 2.0 client credentials)** — identifies the app, not a person. Read-only access to public data; it cannot post, DM, or read private data. Issued in the console, or generated from the API Key and Secret via `POST https://api.x.com/oauth2/token` with `grant_type=client_credentials`.
  - **API Key and Secret** — identify the app; used to generate tokens and to sign legacy OAuth 1.0a user-context requests (Access Token and Secret). OAuth 1.0a still works on many v2 endpoints but is the legacy path; build new work on OAuth 2.0.
  - Main OAuth 2.0 scopes: reads — `tweet.read`, `users.read`, `users.email`, `follows.read`, `like.read`, `list.read`, `bookmark.read`, `dm.read`, `space.read`; writes — `tweet.write`, `tweet.moderate.write`, `follows.write`, `like.write`, `list.write`, `bookmark.write`, `dm.write`, `media.write`; session — `offline.access`. Posting needs `tweet.read` + `tweet.write` + `users.read` (plus `media.write` for media).
- Key endpoints (agent use first):
  - Create a post: `POST /2/tweets` — user context only; the Bearer Token is rejected
  - Delete a post: `DELETE /2/tweets/{id}` — user context only
  - Look up posts: `GET /2/tweets/{id}`, `GET /2/tweets` (by IDs)
  - Search: `GET /2/tweets/search/recent` (recent search); `GET /2/tweets/search/all` is full-archive search and is not available on every access level — which levels may use it is **unverified** for the current pay-per-use model
  - Filtered stream: `GET /2/tweets/search/stream` — app-only Bearer Token only, user tokens are refused
  - Users: `GET /2/users/me` (user context), `GET /2/users/{id}`, `GET /2/users/by/username/{username}`, `GET /2/users/by`
  - A user's posts / mentions: `GET /2/users/{id}/tweets`, `GET /2/users/{id}/mentions`
  - Home timeline: `GET /2/users/{id}/timelines/reverse_chronological` — user context
  - Follows: `GET /2/users/{id}/followers`, `GET /2/users/{id}/following`; follow with `POST /2/users/{id}/following` (user context)
  - Likes / reposts: `POST /2/users/{id}/likes`, `POST /2/users/{id}/retweets` (user context)
  - Direct Messages: `POST /2/dm_conversations/with/{participant_id}/messages` — user context with `dm.write`; the Bearer Token cannot touch DMs
  - Media upload: `POST /2/media/upload` — user context with `media.write`
- Pricing — **read this before quoting any number**: X's pricing changes often and X itself says prices are subject to change, with the Developer Console as the live source. As checked on 2026-11-23 against X's official pricing page (`https://docs.x.com/x-api/getting-started/pricing`):
  - The old subscription tiers are **no longer what a new developer buys**. X moved self-serve access to **pay-per-use** on 2026-02-06: you buy prepaid credits in the Developer Console, there is no subscription, no contract and no minimum spend, and requests are deducted per call — writes per request, reads per resource returned.
  - Rates on that page: create a post **$0.015** per request, **$0.20** if the post contains a URL; read a post **$0.005** per post returned; read a user **$0.010** per user returned; reads of your own data ("owned reads") **$0.001** per resource. Resources are deduplicated within a 24-hour UTC day.
  - Pay-per-use is capped at **3,000,000 post reads per monthly billing cycle**; beyond that X directs you to **Enterprise**, which still exists as a custom contract — X publishes no Enterprise price, so do not quote one.
  - Historical tiers, for reading older posts and recipes: Free ($0, write-only — see below), Basic ($200/month, $175/month on annual billing) and Pro ($5,000/month) were the tiers listed on `developer.x.com/en/docs/x-api`. Whether an existing Basic/Pro subscription can still be renewed is **unverified** — sources disagree, and the current pricing page lists no subscription plans or prices.
  - **There is no free tier for new sign-ups on the current pricing page.** One-time promotional credits for new paying accounts are incentives, not a standing free plan.

## Human setup (once)

1. Sign in to the Developer Console (`console.x.com`, formerly the developer portal at `developer.x.com`) with an X account. Individual / personal developer accounts can sign up this way; there is no separate company requirement for self-serve access.
2. Accept the **Developer Agreement and Developer Policy**, and complete the developer profile describing how you will use the API. That description matters: the policy expects the stated use to be accurate and kept up to date, and X can enforce against the use case after the fact.
3. Create an app (New App: name, description, use case) and **save the credentials immediately — they are shown only once**: API Key and Secret, Bearer Token, and (for your own account) Access Token and Secret. Losing one means regenerating it, which invalidates the old one.
4. If the agent must act as a user, turn on OAuth 2.0 in the app's authentication settings, pick the app type (e.g. "Web App, Automated App or Bot"), and register the redirect URI exactly — OAuth 2.0 requires an exact match.
5. Buy pay-per-use credits in the console before testing, and set a spending limit: with no credit balance, calls fail, and a link in a post costs over 13× a plain post.

## Known blocker

Two separate walls, and they stack:

1. **No official Muse connector for X**, and the user-context OAuth flow cannot be registered as a Muse custom connector: the authorize endpoint is on `x.com` (`https://x.com/i/oauth2/authorize`) while the token endpoint is on `api.x.com` (`https://api.x.com/2/oauth2/token`). Muse's secure connector flow requires all OAuth endpoints on a single issuer domain, so registration is declined with "token_url is on a host outside issuer_domain" — the same single-domain rule documented in `field-notes/muse-oauth-single-domain-rule.md`. That kills posting, DMs, bookmarks and home-timeline access through that flow.
2. **Money**: since 2026-02-06 there is no free read tier to fall back on; reads and writes are billed from prepaid credits from the first call.

What still works: a custom connector (or a plain script / MCP server outside Muse's OAuth registration flow) using the **Bearer Token / API Key route for public reads**, and a self-implemented OAuth 2.0 PKCE flow run outside that registration flow for user-context actions. Budget the credits first either way.

## Pitfalls

- The old Free tier shaped most tutorials still online: it was **write-only** — low-rate access to the post and media-upload endpoints (500 posts per month at the app level, as listed on the old access-levels page) with essentially no usable read access. Any recipe that assumes "free tier + search" is pre-2026 and wrong now; on the current model, search results are billed per post returned.
- The Bearer Token cannot post. `POST /2/tweets` with an app-only token fails; posting always needs a user-context token for the account being posted as.
- Without `offline.access`, an OAuth 2.0 access token dies after 2 hours and there is no refresh token. The authorization code also has to be exchanged promptly — do not park it.
- A post containing a URL costs $0.20 against $0.015 for a plain post. An agent that posts links by default burns credits 13× faster than expected.
- Reads are charged per resource **returned**, not per request: one search page of 100 posts is 100 billed reads. Page sizes and `max_results` are cost controls, not just performance knobs.
- Rate limits still apply on top of billing (per endpoint, mostly per 15-minute window; responses carry `x-rate-limit-limit` / `x-rate-limit-remaining` / `x-rate-limit-reset`). Paying does not remove them.
- Prices, caps and endpoint availability have changed repeatedly (2023, 2024, and the 2026-02-06 pay-per-use migration). Treat every number in this recipe as a 2026-11-23 snapshot and re-check the pricing page and the Developer Console before committing a budget.
