# Instagram connector recipe

Service: Instagram — Instagram Platform (Instagram API with Instagram Login / Instagram API with Facebook Login)
Purpose: read an Instagram professional account's profile and media, manage and reply to comments, read insights, publish media, and handle mentions and messages from an agent, without opening the Instagram app or web UI.
Verified against Meta's official developer documentation on 2026-11-23.

Status: **公式コネクタあり (official Muse connector available).** Instagram is already available as an official Muse connector and is connected and in real use in the maintainer's environment, so most users do not need to build a custom connector for it at all. This recipe documents the underlying Meta API for agents on other platforms, and for understanding what the official connector sits on top of.

## API facts

Meta's current offering is the **Instagram Platform**: one Graph API surface in two setups. They differ in how the user logs in, whether a Facebook Page is required, which host is called, which token type is issued, and which permission names are used. Pick one setup per app; if you need both, Meta's guidance is to create a separate app for each.

|  | Instagram API with Instagram Login | Instagram API with Facebook Login |
| --- | --- | --- |
| Who logs in | The Instagram account owner, with Instagram credentials | A Facebook user who manages the linked Page, with Facebook credentials |
| Facebook Page | Not required | Required — the Instagram account must be linked to a Facebook Page |
| API host | `graph.instagram.com` | `graph.facebook.com` |
| Access token | Instagram User access token | Facebook User or Page access token |
| Permission names | `instagram_business_*` | `instagram_*` plus `pages_*` |
| Publishing, comments, insights, mentions | Yes | Yes |
| Messaging | Yes, in the same API | Via Messenger Platform support for Instagram |
| Hashtag search, business discovery, product tagging, partnership ads | No | Yes |

- Account types: **both setups require an Instagram professional account — Business or Creator.** Personal (consumer) accounts are not supported by either setup. See Pitfalls for what happened to the old personal-account API.
- Auth: OAuth 2.0 authorization code flow in both setups.
  - Instagram Login setup:
    - Authorize: `https://www.instagram.com/oauth/authorize` — returns a code valid for 1 hour and usable once
    - Exchange code: `POST https://api.instagram.com/oauth/access_token` — returns a short-lived token (about 1 hour)
    - Long-lived token: `GET https://graph.instagram.com/access_token` with `grant_type=ig_exchange_token` — 60 days; refresh via `GET https://graph.instagram.com/refresh_access_token` once the token is at least 24 hours old
  - Facebook Login setup:
    - Authorize: `https://www.facebook.com/dialog/oauth` (Facebook Login dialog)
    - Token: `https://graph.facebook.com/oauth/access_token` — short-lived tokens are exchanged for long-lived tokens (60 days) through the Graph API
- Permissions (request only what the feature needs — every extra permission is one more item to justify in review):
  - Instagram Login: `instagram_business_basic` (profile and media; the base permission the others depend on), `instagram_business_content_publish`, `instagram_business_manage_comments`, `instagram_business_manage_messages`, and `instagram_business_manage_insights` for insights. Note: Meta's own pages are not fully consistent about the insights permission for this setup — the insights guide names `instagram_business_manage_insights`, while some permissions / review lists omit it. Follow the insights guide for the endpoint you actually call. The old short names (`business_basic`, `business_content_publish`, etc.) were retired on 2025-01-27 in favour of the `instagram_business_*` names — do not use the old names in new code.
  - Facebook Login: `instagram_basic`, `instagram_content_publish`, `instagram_manage_comments`, `instagram_manage_insights`, `instagram_manage_messages`, plus Page permissions such as `pages_show_list` and `pages_read_engagement` (needed to find the Page and its linked Instagram account).
- App review: a new app starts with **Standard Access**, which only works for accounts owned by, or held by people with a role on, the app itself. Serving professional accounts you do not own or manage requires **Advanced Access**, granted per permission through **Meta App Review**, and **Business Verification** is mandatory for that. Plan for a screencast and a use-case explanation per permission.
- Key endpoints (agent use; `{ig-user-id}` is the Instagram professional account ID, `{ig-media-id}` a media object ID):
  - Profile: `GET /me` (Instagram Login) or `GET /{ig-user-id}` — `id`, `username`, `name`, `biography`, `website`, `profile_picture_url`, `followers_count`, `follows_count`, `media_count`
  - Media list: `GET /{ig-user-id}/media` — published media with fields such as `id`, `media_type`, `media_url`, `permalink`, `caption`, `timestamp`, `like_count`, `comments_count`
  - One media object / stories: `GET /{ig-media-id}`, `GET /{ig-user-id}/stories`
  - Comments: `GET /{ig-media-id}/comments`, `POST /{ig-media-id}/comments`, `GET /{ig-comment-id}/replies`, `POST /{ig-comment-id}/replies`, plus hide / unhide, enable / disable comments, and `DELETE /{ig-comment-id}` for moderation
  - Mentions: media where the account has been @mentioned is reachable through the mentions / tags surface (`GET /{ig-user-id}/tags`) — exact field set unverified (unverified)
  - Insights: `GET /{ig-user-id}/insights` (account level, e.g. impressions / reach / profile views by period) and `GET /{ig-media-id}/insights` (media level)
  - Publishing (two-step container flow, both setups): `POST /{ig-user-id}/media` creates a media container from a publicly reachable media URL, check `GET /{ig-container-id}?fields=status_code`, then `POST /{ig-user-id}/media_publish` publishes it. Supports single images, videos, Reels, Stories and carousels; a carousel counts as one post.
  - Publishing quota check: `GET /{ig-user-id}/content_publishing_limit` — read this before bulk posting
  - Facebook Login account discovery: `GET /me/accounts` lists the user's Pages, then `GET /{page-id}?fields=instagram_business_account` gives the linked Instagram account ID. With Instagram Login the token already belongs to the Instagram account, so this step does not exist.
  - Facebook Login only: hashtag search (`GET /ig_hashtag_search`, then `GET /{ig-hashtag-id}/top_media` / `recent_media`) and Business Discovery (`GET /{ig-user-id}?fields=business_discovery.username(...)`) for basic metadata and metrics about other Business / Creator accounts
  - Messaging: conversations and messages endpoints exist in both setups (via the Messenger Platform on the Facebook Login side); detailed messaging endpoint shapes are unverified in this pass (unverified)
  - Webhooks: Instagram Webhooks can push real-time notifications for supported activity on connected professional accounts — prefer them over polling comments and mentions.
- Rate limits (only figures confirmed in Meta's documentation are given as fact):
  - Business Use Case call volume: **calls within 24 hours = 4800 × the account's impressions in the trailing 24 hours**, reported as percentages in the `X-Business-Use-Case-Usage` response header. A small account therefore has a small budget — this is the limit that surprises new integrations.
  - Content publishing: **100 API-published posts per account per rolling 24 hours** (a carousel counts as 1); check live usage with `GET /{ig-user-id}/content_publishing_limit`. An unpublished media container expires after 24 hours.
  - Messaging: Conversations API 2 calls/sec per Instagram account; Private Replies 750 calls/hour for posts / Reels (100 calls/sec for Live); Send API 100 calls/sec for text and 10 calls/sec for audio / video.
  - A flat "200 calls per hour" figure is widely quoted in third-party guides for this API, but it could not be confirmed as a general limit in Meta's current documentation during this pass, so treat it as unverified (unverified) and design against the Business Use Case formula and the header above instead.

## Human setup (once)

1. Register as a Meta developer at developers.facebook.com (tied to a Facebook account) and create an app in the Meta App Dashboard — a Business-type app with the Instagram use case ("Manage messaging and content on Instagram"). One API setup (Instagram Login **or** Facebook Login) per app.
2. Make sure the target Instagram account is a **professional account (Business or Creator)**. For the Facebook Login setup, also link it to a Facebook Page, and make sure the person logging in has a task / role on that Page.
3. Add only the permissions the integration actually uses (see API facts), configure the OAuth redirect URI, and run the login flow for the chosen setup to get a token; exchange it for the 60-day long-lived token and store the refresh step — an unrefreshed long-lived token expires.
4. If the app will serve accounts you do not own, connect a business portfolio, complete Business Verification, and submit each permission for App Review with a screencast. Until then you are on Standard Access and can only use your own / app-role accounts.
5. On Muse specifically: skip all of the above and use the official Instagram connector (see Status) — connect the account through Meta Accounts Center instead of registering a Meta app.

## Known blocker

For a **custom** connector on Muse, the general Muse OAuth rule applies: Muse's secure connector flow requires the OAuth endpoints to sit on a single issuer domain, and it declines a registration when the token endpoint is on a different host (`token_url is on a host outside issuer_domain`). Both Instagram setups span hosts by design:

- Instagram Login: authorize on `www.instagram.com`, code exchange on `api.instagram.com`, API and token refresh on `graph.instagram.com`.
- Facebook Login: authorize on `www.facebook.com`, token and API on `graph.facebook.com`.

So a self-built OAuth connector for Instagram hits the same cross-domain wall as the Google recipes in this collection. This does **not** block Instagram on Muse in practice, because the official Muse connector already exists and is connected and working in the maintainer's environment — use it. The cross-domain wall only matters if you try to register your own Meta app as a Muse custom connector. On platforms that accept Meta's endpoints as-is, or anywhere an API-key-style / pre-issued long-lived token can be registered, this recipe works as documented; API-key-style registration is buildable in Muse's flow in general, OAuth with split domains is not.

A second, harder wall is Meta-side and applies on every platform: **there is no API for personal Instagram accounts any more** (see Pitfalls). No connector — official, custom, or third-party — can read or publish for a personal account through Meta's official APIs.

## Pitfalls

- **Personal accounts are a dead end.** The Instagram Basic Display API, the old read-only API that worked with personal accounts, was shut down on 2024-12-04; Meta stated that after that date there is no set of Instagram APIs for consumer developer apps. If a user's feed or integration broke in December 2024, this is why. The only fix is switching the account to Business or Creator (free, in Instagram's account settings) and reconnecting through one of the two setups above. Do not follow pre-2024 guides that start from Basic Display.
- **Permission names are setup-specific and were renamed.** Instagram Login uses `instagram_business_*`; Facebook Login uses `instagram_*` plus `pages_*`. Mixing the two sets, or using the retired pre-2025 short names, fails authorization. Meta's documentation also disagrees with itself about `instagram_business_manage_insights` (see API facts) — test the insights call with the scope from the insights guide.
- **The Facebook Login setup fails silently without the Page link.** The Instagram account must be linked to a Facebook Page and the caller needs a Page task; resolve the Instagram account ID via `GET /me/accounts` → `GET /{page-id}?fields=instagram_business_account` instead of guessing it or reusing the Instagram username / numeric profile ID from the app.
- **Tokens expire on a schedule, not on use.** Short-lived tokens last about 1 hour; long-lived tokens last 60 days and can only be refreshed once they are at least 24 hours old. Build the refresh in from the start — an integration that "worked last month" and then stops is usually an unrefreshed token, not a revoked permission.
- **Publishing is a two-step, URL-based flow.** Media must be reachable at a public URL for Meta to fetch when the container is created; local files cannot be uploaded directly through these endpoints. Poll the container status before calling `media_publish`, respect the 100 posts / 24 hours quota, and remember containers expire after 24 hours.
- **Rate budget scales with the account's audience.** Because the Business Use Case budget is 4800 × impressions, a brand-new or tiny account can be throttled almost immediately. Cache profile / media reads, request only the fields you need, and use Webhooks for comments and mentions instead of polling.
- **Newer setup names are appearing in Meta's pages.** Current documentation pages also mention an "Instagram API with Unified Login" as a recommended setup for new integrations. Its hosts, permission names and review path were not fully verified against the official documentation in this pass, so they are unverified (unverified) here — this recipe documents the two established setups above.
- Unofficial libraries that automate Instagram by reverse-engineering the private / consumer app protocols violate Meta's terms, break without notice, and risk the user's account. Do not use them, and do not recommend them.
