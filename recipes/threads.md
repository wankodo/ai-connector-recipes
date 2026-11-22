# Threads connector recipe

Service: Threads — Threads API
Purpose: read a Threads profile, its posts, replies and insights, and publish posts and replies on the account's behalf, without opening the Threads UI.
Verified against Meta's official developer documentation on 2026-11-23.

Status: **公式コネクタあり** — Muse ships an official Threads connector, and it is connected and in real use in the author's environment. In Muse you do not need to build a custom connector for Threads; the API facts below are for building elsewhere, or for understanding what the official connector sits on top of.

## API facts

- API name: **Threads API** (Meta for Developers, `developers.facebook.com/documentation/threads`). Open to all developers since 2024-06-18. It is its own API surface, separate from the Facebook Graph API host.
- API hosts: `graph.threads.net` (the documented host since 2024-06-07; all calls use version `v1.0`). Since 2025-06-06 the API can also be accessed via `graph.threads.com`.
- Auth: OAuth 2.0 authorization code flow with **Threads user access tokens** (app-scoped: unique to the app + user pair).
  - Authorization Window: `https://threads.net/oauth/authorize` — the user grants permissions here; the redirect back carries an authorization code (valid 1 hour).
  - Token: `POST https://graph.threads.net/oauth/access_token` — exchanges the code for a short-lived token (valid 1 hour).
  - Long-lived token: `GET https://graph.threads.net/access_token?grant_type=th_exchange_token` — valid 60 days. Refresh before expiry via `GET /refresh_access_token`.
  - Scopes (permissions):
    - `threads_basic` — required for **all** Threads endpoints
    - `threads_content_publish` — required for publishing endpoints
    - `threads_read_replies` — required for `GET` calls to reply endpoints
    - `threads_manage_replies` — required for `POST` calls to reply endpoints
    - `threads_manage_insights` — required for `GET` calls to insights endpoints
    - Additional permissions exist for deleting posts (`threads_delete`), mentions and keyword search; request only what you use.
- Key endpoints (all under `https://graph.threads.net/v1.0`):
  - Profile: `GET /me` (or `GET /{threads-user-id}`) — fields include `id`, `username`, `name`, `threads_profile_picture_url`, `threads_biography`
  - A user's posts: `GET /{threads-user-id}/threads`
  - Publish (two steps, always): `POST /{threads-user-id}/threads` creates a media container, then `POST /{threads-user-id}/threads_publish` publishes it. Media must be hosted on a publicly accessible server — Meta downloads it at publish time. A reply is a container created with `reply_to_id`.
  - Replies: `GET /{threads-media-id}/replies` and `GET /{threads-media-id}/conversation` to read; `POST /{threads-reply-id}/manage_reply` to hide / unhide a reply
  - Insights: `GET /{threads-media-id}/insights` (per post: `views`, `likes`, `replies`, `reposts`, `quotes`, `shares`) and `GET /{threads-user-id}/threads_insights` (per account, incl. `followers_count` and `follower_demographics`)
  - Quota check: `GET /{threads-user-id}/threads_publishing_limit` — returns current usage against the limits below (needs `threads_basic` and `threads_content_publish`)
  - Delete: `DELETE /{threads-media-id}`
- Rate limits (per Threads profile, rolling / moving 24-hour windows):
  - **250 API-published posts** per 24 hours (a carousel counts as one post)
  - **1,000 replies** per 24 hours
  - **100 deletions** per 24 hours
  - **500 location searches** per 24 hours
  - General call volume is counted per app + user pair: calls within 24 hours = 4,800 × number of impressions (plus total CPU / total time formulas on the same basis)
  - Text posts are limited to 500 characters
- Personal accounts: yes, usable. The API acts on a person's own Threads profile once that person grants your app the permissions. Since 2025-09-23, Threads profiles **without a linked Instagram account** can also use the API (initially except the `followers_count` / `follower_demographics` metrics; those were opened to them on 2026-01-30). A linked Instagram account is therefore no longer a prerequisite.

## Human setup (once)

1. At developers.facebook.com, create a Meta app and pick the **Threads Use Case**. Note: the app gets **2 app IDs and 2 app secrets** — for the Threads API, use the Threads app ID and its corresponding secret.
2. Add the permissions you need in the Threads use case (start with `threads_basic`; add `threads_content_publish` to post).
3. For your own account, no App Review is needed: add yourself as a **Threads Tester** (App Dashboard > App roles > Roles > Add People), then accept the invitation in Threads under Account Settings > Website permissions. Threads testers can grant your app its permissions at any time.
4. Run the Authorization Window flow, exchange the code for a short-lived token, then exchange that for a long-lived token and store it server-side. Plan to refresh it before the 60 days run out.
5. Only when people **without a role on your app** must connect: each permission must first be approved through **App Review**, and the app must be published. Until then the app works for app-role users (admins, developers, Threads testers) only.

## Known blocker

In Muse there is no blocker in practice: an official Threads connector already exists (公式コネクタあり) and is connected and working in the author's environment, so use it instead of building one.

If you nevertheless try to register Threads as a **custom** connector in Muse, expect the general Muse custom-connector rule to bite: Muse's secure connector flow requires the OAuth authorize and token endpoints to sit on one issuer domain ("token_url is on a host outside issuer_domain"). Threads splits them — authorize on `threads.net`, token on `graph.threads.net` — so the custom registration is declined, the same wall as the Google recipes. An API-key-style route is buildable in Muse custom connectors in general, but the Threads API has no API-key auth: it is user-token OAuth only, so there is no key-based fallback for Threads itself.

## Pitfalls

- Publishing is always two calls — container, then publish. Do not expect a single "post" endpoint, and check the container is finished before publishing it.
- Long-lived tokens die after 60 days if not refreshed, and permission grants are a separate clock: grants by users with public profiles last 90 days and refreshing the token extends them, but a grant by a user with a **private** profile cannot be extended — that user must grant again after expiry.
- Use the Threads app ID / secret, not the app's main Facebook credentials; mixing the two pairs is the most common first-run failure.
- Posts with more than 5 links fail with `THREADS_API__LINK_LIMIT_EXCEEDED` (enforced since 2025-12-22).
- `follower_demographics` needs a profile with at least 100 followers and a `breakdown` parameter (`country`, `city`, `age`, or `gender`); insights data does not go back before 2024-04-13.
- Whether `threads.com/oauth/authorize` has replaced `threads.net/oauth/authorize` as the canonical Authorization Window host after Threads' domain migration is **unverified (未確認)** as of 2026-11-23 — both hosts appear in current material; confirm in the official Get Access Tokens page before hard-coding one.
