# Facebook connector recipe

Service: Facebook — Graph API (including the Pages API)
Purpose: read and publish Facebook Page content, read the signed-in user's own profile basics, and moderate Page comments from an agent, without opening the Facebook UI.
Status: **Official connector available in Muse (公式コネクタあり)** — Facebook is already a Muse official connector / CLI, connected and in real use in the author's environment. A custom Graph API build (below) is only needed outside Muse.
Verified against Meta's official developer documentation on 2026-11-23.

## API facts

- API name: **Graph API**. It models Facebook as nodes (User, Page, Post, Photo, Comment), edges (a Page's `/feed`, a post's `/comments`) and fields you request explicitly with `?fields=`.
- API host: `graph.facebook.com` — almost all requests go here. The documented exception is video uploads, which use `graph-video.facebook.com`.
- Versioning: versions are released quarterly and the version goes in the path, e.g. `GET https://graph.facebook.com/v{VERSION}/me`. Always pin a version: with no version in the path, the API defaults to the **oldest** available version.
- Auth: OAuth 2.0 via Facebook Login.
  - Authorize: `https://www.facebook.com/v{VERSION}/dialog/oauth` (parameters: `client_id`, `redirect_uri`, `state`, `scope`)
  - Token: `https://graph.facebook.com/v{VERSION}/oauth/access_token`
  - Note the two different hosts — this matters for the blocker below.
- Access token types (pick the wrong one and calls fail even with the right permissions):
  - **User access token** — acts for a signed-in person. Short-lived tokens last about 1–2 hours; exchange one server-side (`grant_type=fb_exchange_token`, with the app secret) for a long-lived token of about 60 days.
  - **Page access token** — acts as one Page. Get it with `GET /me/accounts` using a user token; each Page in the response carries its own `access_token`. A Page token obtained from a **long-lived** user token has no expiration date (it can still be invalidated, e.g. if the person loses their Page role or removes the app). A Page token obtained from a short-lived user token inherits the short lifetime.
  - **App access token** — the app itself, for app-level calls; derived from the app ID and app secret, server-side only.
  - **System user token** — a business automation identity created in Business settings; the usual choice for unattended jobs on business assets.
- Key endpoints (agent use):
  - Own profile: `GET /me` — with a user token this resolves to that person; by default it returns only `id` and `name`. IDs are app-scoped: the same person has a different ID in every app. More fields (e.g. `email`, `picture`) need the matching permission granted.
  - List managed Pages + Page tokens: `GET /me/accounts` (needs `pages_show_list`)
  - Page info: `GET /{page-id}?fields=id,name,about,fan_count`
  - Publish to a Page: `POST /{page-id}/feed` with `message`, `link`, `published` (needs a Page access token and `pages_manage_posts`)
  - Read Page posts / insights: `GET /{page-id}/posts`, `GET /{page-id}/insights` (needs `pages_read_engagement`)
  - Comments: `GET /{object-id}/comments`, `POST /{object-id}/comments`, `DELETE /{comment-id}` (needs `pages_manage_engagement`)
  - User content on a Page (visitor posts, ratings, mentions): `GET /{page-id}/tagged` (needs `pages_read_user_content`)
- Permissions are `pages_*`-centred for Page work: `pages_show_list`, `pages_read_engagement`, `pages_manage_posts`, `pages_manage_engagement`, `pages_read_user_content`, `pages_manage_metadata`, `pages_manage_ads`, `pages_messaging`, plus `business_management` for business assets. Ask for the minimum set — each one is reviewed separately.
- Access levels and App Review:
  - **Standard Access** is granted automatically for all permissions, but only works for people who have a **role on the app**. An app used only by its own owner/admins (e.g. an agent acting on the owner's own Pages) needs nothing more.
  - **Advanced Access** is required before people *without* an app role can grant a permission. It is approved per permission through **App Review**, and since 2023-02-01 apps requesting Advanced Access may also have to be connected to a **verified business** (Business Verification).
- Personal profiles are *not* the centre of this API:
  - The documented publishing surface is **Pages**: the Graph API overview directs publishing to a Page's feed via the Pages API, and user-side sharing to the Sharing (dialog) guide. There is no Pages-API-style endpoint for posting to a personal timeline; treat any recipe that promises one as stale.
  - What an app can read about a person is essentially that person's own data, limited to the fields and permissions they granted to *your* app. There is no general API for reading other people's profiles, timelines or friends' data.
  - **Marketplace:** no general Graph API endpoint for searching or reading Marketplace listings was confirmed in the official documentation checked for this recipe (**unverified**). Meta's Marketplace documentation describes partner/commerce integrations for approved businesses distributing their own products, not a public listing feed. In Muse, personal Marketplace use (search, saved items, own listings, buying/selling) works through the **official Facebook connector**, not through a self-built Graph API app.
- Rate limits (**unverified** — Meta's rate-limiting guide itself could not be fetched for this recipe, see Pitfalls; figures below are the formulas that guide is consistently reported to state):
  - Platform limit, app level: 200 calls per hour × the app's number of users, reported in the `X-App-Usage` header (error code 4 when hit).
  - Platform limit, per user: a separate cap exists but Meta does not publish the number (error code 17).
  - Pages API with a Page or system user token uses Business Use Case limits: 4,800 calls per 24 hours × the Page's engaged users, reported in `X-Business-Use-Case-Usage` (error code 80001).
  - Re-check these numbers against the live guide before designing around them.

## Human setup (once)

1. **In Muse: nothing to build.** Connect the official Facebook connector with the user's account; that lane already covers profiles, timelines, Pages they manage, Groups, events and Marketplace.
2. Only for a self-built Graph API app (outside Muse): register as a Meta developer and create an app at developers.facebook.com, add the Facebook Login product, and register every callback under Valid OAuth Redirect URIs (exact match, HTTPS).
3. For the owner's own account and Pages, Standard Access is enough — generate a user token (Graph API Explorer is fine for testing), exchange it for a long-lived token on your server, then call `GET /me/accounts` and store the Page tokens.
4. If anyone *else* will ever use the app, budget for App Review per permission plus Business Verification before you build on it — and add a privacy policy URL, which review requires.

## Known blocker

For Muse users there is no blocker: the **official connector exists** and is the supported route.

For a **custom** Muse connector built on Facebook Login OAuth, expect the same single-domain wall as the Google recipes: the authorize endpoint is on `www.facebook.com` while the token endpoint is on `graph.facebook.com`, and Muse's secure connector flow declines cross-domain registrations with "token_url is on a host outside issuer_domain". The general rule for custom connectors: an OAuth flow whose authorize and token hosts differ cannot be registered there; an API-key-style auth can. Facebook offers no plain API-key auth for user/Page data (the app secret is used server-side to mint tokens, not as a per-request key), so the custom-connector route has no simple fallback — use the official connector.

## Pitfalls

- **Verification note:** on 2026-11-23, `developers.facebook.com` returned HTTP 403 to this agent's page fetcher, so the facts above were checked against Meta's official documentation as mirrored and quoted in full (the Graph API overview and the Pages API documentation, including their `developers.facebook.com` source URLs). Items that could not be confirmed that way — the Marketplace surface and the exact rate-limit formulas — are marked **unverified** in the text. Re-check them on the live site before relying on them.
- Page token lifetime is inherited silently: mint Page tokens *after* exchanging for a long-lived user token, or they die in about an hour and nothing in the response tells you which kind you got.
- Standard Access fails quietly: pointed at a Page the app-role holder doesn't own, calls return empty results or permission errors that look like bugs. It is the access level, not your code.
- `/me` and user IDs are app-scoped. Never store a user ID from one app and use it in another, and never confuse a Page ID with the Page's username/vanity URL — get IDs from `GET /me/accounts`.
- Personal account ≠ Page. Almost everything an agent wants to *publish* must target a Page with a Page access token; building the same flow against a personal profile is the most common dead end.
- Token lifetimes are Meta's to change; read `expires_in` from the exchange response instead of hard-coding 60 days.
