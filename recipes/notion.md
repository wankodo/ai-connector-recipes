# Notion connector recipe

Service: Notion — Notion API
Purpose: search, read, create and update pages, databases (data sources) and blocks in a Notion workspace from an agent, without opening the Notion UI.
Verified against Notion's official developer documentation on 2026-11-23.
Status: **構築可（見込み・登録自体は未検証）** — buildable in prospect via an internal integration token registered as a Muse custom connector; the Muse registration itself has not been tested. Notion also offers OAuth integrations on the same API host (see Known blocker).

## API facts

- API host: `api.notion.com` — base URL `https://api.notion.com/v1`. All requests go over HTTPS.
- Auth header on every request: `Authorization: Bearer <token>`, plus `Content-Type: application/json` when sending a JSON body.
- **`Notion-Version` header is required** on all REST API requests. Versions are named for their release date (for example `2022-06-28`, `2025-09-03`); the `/v1` in the URL is unrelated to this versioning and is not expected to change. The exact latest version string was unverified in this pass — the versioning page renders it dynamically — so read the current value off the official versioning page when you build, rather than copying a version number from a blog post.
- Auth types (three, per the official authorization docs):
  - **Internal integration (internal connection)** — a single-workspace connection that acts as its own bot user. Creating one immediately issues a static **internal integration token**; there is no OAuth flow, and the same token is sent with every request. You must be a **workspace owner** to create a connection. Its reach is set by **capabilities** (set on the Configuration tab — e.g. read content, update content, insert content, read user information) *and* by which pages/databases have been shared with it (below).
  - **Public / OAuth integration** — for installing into other people's workspaces. OAuth 2.0 authorization code flow:
    - Authorize: `https://api.notion.com/v1/oauth/authorize` (`client_id`, `response_type=code`, `owner=user`, `redirect_uri`, optional `state`). The user picks which pages to share in a page picker; access to a parent page includes its child pages.
    - Token: `POST https://api.notion.com/v1/oauth/token` — HTTP Basic auth with `client_id:client_secret` base64-encoded, body `grant_type=authorization_code` + `code` (+ `redirect_uri` when required). The response includes `access_token`, `refresh_token`, `bot_id`, `workspace_id` and `owner`.
    - Note both OAuth endpoints live on `api.notion.com` — the same host as the API itself.
  - **Personal access token (PAT)** — a user-scoped static token that acts as the creating user and uses that user's existing page permissions, so no separate sharing step is needed. Officially positioned for scripts/CLI/trusted tools acting as one person, not for team-owned automations or multi-user products.
- Key endpoints (agent use):
  - Search everything shared with the connection: `POST /v1/search` — by title; with no `query` it returns all shared pages and data sources; `filter` can limit it to pages or data sources only
  - Validate a token / who am I: `GET /v1/users/me`; list workspace users: `GET /v1/users`
  - Get a page: `GET /v1/pages/{page_id}`; create a page: `POST /v1/pages`; update page properties: `PATCH /v1/pages/{page_id}`
  - Read page content: `GET /v1/blocks/{block_id}/children`; append content: `PATCH /v1/blocks/{block_id}/children`
  - Get a database: `GET /v1/databases/{database_id}`; query its rows: `POST /v1/databases/{database_id}/query`
  - Data sources: since the `2025-09-03` version a database is modelled as one or more **data sources** — the official search reference already speaks of data sources, and querying moves to `POST /v1/data_sources/{data_source_id}/query`. Which shape you call depends on the `Notion-Version` you pin; check the reference for that version before coding against either.
  - Comments: `GET /v1/comments`, `POST /v1/comments` (subject to the connection's capabilities)
- Rate limits (official request-limits page): two limits apply — per-connection and per-workspace (shared across all of the workspace's connections).
  - Per-connection: **180 requests per minute (an average of 3 per second)** on all plans other than Business/Enterprise; **600 per minute (an average of 10 per second)** on Business and Enterprise. The budget can be spent in bursts within the 60-second window.
  - Querying a data source has its own separate limit of about 3 requests per second per connection.
  - Over the limit: HTTP 429 with error code `rate_limited` and a `Retry-After` header (integer seconds — use the returned value, do not assume a fixed wait). HTTP 529 `service_overload` is retried the same way. Payload size limits also apply: max 1000 block elements and 500 KB per request.
- Personal / free-plan use: yes. Notion's official help states that on **Free and Plus plans, only workspace owners can create connections** (on Business/Enterprise, owners can also let members create them) — so a personal workspace owner on the Free plan can create an internal integration and use the API; the plan mainly changes who may create connections and the rate limit above.

## Human setup (once)

1. Be signed in as a **workspace owner** of the workspace you want to connect (on Free/Plus this is mandatory for creating a connection).
2. In the Notion Developer portal, create a new **internal integration / connection**, associate it with that one workspace, and set only the capabilities the agent needs (start with read content; add update/insert content only if it must write).
3. Copy the **internal integration token** from the connection's Configuration tab. Treat it like a password: it is static and works until refreshed/revoked.
4. **Share the content**: for every page or database the agent must see, open it in Notion, use the `•••` menu → **Add connections**, and add the connection (or manage access from the connection's Content access tab in the Developer portal). Sharing a parent page also grants its child pages.
5. Register the token as the credential of a Muse custom connector pointing at `https://api.notion.com/v1`, and make sure every request also sends a current `Notion-Version` header.

## Known blocker

There is no Notion-side blocker for the token route: an internal integration token is a plain bearer credential for `api.notion.com`, so it does not touch OAuth at all.

For Muse custom connectors specifically:

- The general Muse rule is that an OAuth flow whose authorize host and token host differ cannot be registered — the flow declines it with "token_url is on a host outside issuer_domain" (that is what blocks every Google API in this collection). A static-token connector is not subject to that constraint.
- Notion's OAuth route would not obviously trip that rule either, because both `…/v1/oauth/authorize` and `…/v1/oauth/token` are on `api.notion.com`. That is a reading of the official endpoint URLs, not a tested registration.
- **Unverified:** neither the token route nor the OAuth route has actually been registered and exercised as a Muse custom connector in this project. Hence the status above: 構築可（見込み・登録自体は未検証）.

The real wall in day-to-day use is not registration — it is the sharing model in Pitfalls: a perfectly valid token sees nothing until a human shares each page/database with the connection.

## Pitfalls

- **Unshared = invisible, and it looks like "empty", not "forbidden".** An internal connection is a separate bot user. It can only access pages/databases explicitly shared with it; search returns only shared items, and requests for unshared objects error out. When results come back empty, check Add connections / Content access before debugging the token.
- **Capabilities and sharing are two separate gates.** Granting "update content" capability does not grant access to any page, and sharing a page does not widen the connection's capabilities. Both must line up for a call to succeed.
- **Missing `Notion-Version` fails requests.** It is a required header, not an optional nicety — and pinning an old version changes the data model you get back (notably databases vs data sources around `2025-09-03`).
- **One workspace per internal integration.** It cannot be pointed at another workspace; create a separate connection (and token) per workspace. Access also survives independently of people: if the member who shared a page leaves, the connection keeps its access to that page.
- **Respect `Retry-After`, and mind the shared workspace budget.** Even if your connection is under its own 180/min, the per-workspace limit shared across all connections can still throttle you, and its `Retry-After` can exceed a minute. Queue requests instead of bursting through a bulk sync.
- **Do not retry blindly.** A 429 with `rate_limit_reason: public_api_request_blocked` means the connection's API access has been restricted — retrying will not help. A write that returns 503 may already have been saved; check `additional_data` / the object's current state before repeating it, or you will apply the change twice.
