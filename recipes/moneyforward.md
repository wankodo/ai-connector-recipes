# Money Forward connector recipe

Service: Money Forward — マネーフォワード ME (personal household finance) and マネーフォワード クラウド (Money Forward Cloud, business back-office SaaS, incl. クラウド会計 / Cloud Accounting)
Purpose: read and write Money Forward Cloud accounting data (journals, masters, reports) from an external system or AI agent — and document why the personal ME side cannot be connected the same way.
Verified against Money Forward's official documentation on 2026-11-23.

**Status: buildable (構築可) for Money Forward Cloud; NOT buildable for Money Forward ME.** The two product lines have completely different API postures, and confusing them is the main trap with this service — see the blocker section.

## API facts

### Money Forward Cloud (business) — public API exists

- Developer site: `developers.biz.moneyforward.com` (マネーフォワード クラウド 開発者サイト). It publishes REST API references, tutorials, and an API change policy for the Cloud product line.
- Services with a public API listed on the developer site (2026-11-23): クラウド会計 (Cloud Accounting — masters such as accounts, journals, reports; get and update), クラウド経費 (Cloud Expense), クラウド請求書 (Cloud Invoice), クラウド債務支払 (Cloud Debt Payment). A remote MCP server is also offered (Cloud Accounting, and MCP listings for some other services such as クラウド連結会計).
- Eligibility and price (per Money Forward's official Cloud Accounting FAQ): the API is public; anyone using Cloud Accounting — sole proprietors, companies, and tax/accounting professionals alike — can use it, at **no extra charge** on top of the Cloud subscription. Money Forward's own press release (2026-03-26) says the Accounting API and remote MCP server, previously limited to certified-member accounting firms, were opened to **all plans** on that date.
- Auth server (shared by the Cloud APIs): `api.biz.moneyforward.com`
  - Authorize: `https://api.biz.moneyforward.com/authorize`
  - Token: `https://api.biz.moneyforward.com/token`
  - Both endpoints sit on the **same host** — this matters for Muse, below.
- Auth method 1 — **OAuth 2.0 authorization code flow** (RFC 6749). Register an app in the App Portal (below), have the user sign in and approve, then send the access token as `Authorization: Bearer …`. Client authentication style (Basic vs. POST body) is set per app in the App Portal — unverified against the official reference in this pass; check the app detail screen before coding.
- Auth method 2 — **API key → JWT exchange** (the route Money Forward itself recommends for AI agents and automation):
  - Issue an API key in the App Portal (format `mf_api_pro_…`); select the usable services at issue time.
  - Exchange it: `POST https://api.biz.moneyforward.com/auth/exchange` with the key as `Authorization: Bearer <api key>`, no body. Returns a JWT (`access_token`, `token_type: Bearer`, `expires_in: 3600` — always 1 hour, ES256-signed). Cache the JWT and re-exchange before expiry; the exchange endpoint itself is rate-limited to 100 requests/minute per API key.
  - API-key support for the Cloud Accounting API across all endpoints was announced on 2026-09-23; one key can operate multiple businesses (事業者).
- Cloud Accounting API is currently **v3**. Key endpoints (paths per the v3 reference; the production host is `api-accounting.moneyforward.com` — confirm the `servers` entry in the reference before coding, see Pitfalls):
  - Journals: `GET /api/v3/journals`, `POST /api/v3/journals`, `GET /api/v3/journals/{id}`, `PUT /api/v3/journals/{id}`
  - Masters: `GET /api/v3/accounts` (chart of accounts — journals reference accounts by ID, not by name)
  - Linked-account transactions: `GET /api/v3/transactions`, `POST /api/v3/transactions/journalize`
- Scopes are per product and per endpoint, in the `mfc/…` namespace. Cloud Accounting examples from the official developer-site MCP tutorial: `mfc/accounting/journal.read`, `mfc/accounting/journal.write`, `mfc/accounting/accounts.read`, `mfc/accounting/report.read`, `mfc/accounting/taxes.read`, `mfc/accounting/transaction.read`, `mfc/accounting/transaction.write`. The authorization-server API itself uses `mfc/admin/office.read` for business (office) information. Always read the scope off the endpoint's own reference page.
- Official remote MCP server for Cloud Accounting: `https://beta.mcp.developers.biz.moneyforward.com/mcp/ca/v3` (despite the `beta` host name, Money Forward's release notes describe it as generally available to all plans since 2026-03-25/26). It accepts both OAuth 2.0 and the API key above.

### Money Forward ME (personal) — no general developer API

- ME does **not** appear anywhere in the Cloud developer site's API/MCP lineup, and in the official ME help/support pages checked on 2026-11-23 there is **no self-service developer registration, no app portal, and no published way for a general developer to obtain ME API credentials**. Within the scope of the official documentation checked, a general-purpose public API for ME could not be confirmed.
- A documentation repository under Money Forward's GitHub org (`moneyforward/api-doc`) describes an ME API (OAuth 2.0, `moneyforward.com/api/v1/…`); secondary summaries of that repository describe it as a **closed API for specific partner companies**. That partner-only status is **unverified** against a current official Money Forward announcement in this pass — but either way there is no sign-up path an individual developer or an AI agent can use, so do not plan an ME connector around it.

## Human setup (once)

1. Hold a Money Forward Cloud subscription for the service you want (e.g. クラウド会計・確定申告). The API itself has no separate fee.
2. Start using the **App Portal (アプリポータル)** — this is mandatory for both auth methods: no access token or API key can be obtained without it. Registration and use of the portal are free (per the official getting-started doc). Starting the portal is an administrator-side step; check the official App Portal guide for the exact admin permission required.
3. OAuth route: register your app in the App Portal, and grant the connecting user the app-integration permissions (アプリ連携権限) for the target service in the portal's user settings.
   API-key route: issue an API key in the App Portal with the needed services (e.g. business information + Cloud Accounting) selected, and store the key as a secret — it is shown in `mf_api_pro_…` form and acts as the issuer's own authority.
4. There is **no official Muse connector** for Money Forward, so on Muse this is a custom-connector build (next section). Note for orientation only: Money Forward announced an official connector for Anthropic's **Claude** on 2026-08-02 — that is a different platform and does not give Muse anything.

## Known blocker

Two different walls, depending on which Money Forward you mean:

- **ME (personal household data): this is the hard wall.** No general developer API, no credential issuance path — an agent cannot read a user's ME accounts, transactions, or assets through any official route available to the public. Like the LINE recipe, the honest answer for "connect my Money Forward ME" is that it cannot be built; work from exports or screenshots the user provides. (Whether a closed partner API still operates is unverified — it changes nothing for a general developer.)
- **Cloud (business): no platform wall on Muse, unusually.** Muse's custom-connector OAuth rule declines providers whose token URL sits outside the issuer domain (the Google recipes all fail on this). Money Forward Cloud puts **both** the authorize and token endpoints on `api.biz.moneyforward.com`, so that specific blocker does **not** apply, and the API-key → JWT route is buildable as well. The practical gate is organisational, not technical: an administrator must start the App Portal and grant the permissions/keys before any agent gets in, and each product's endpoints must be checked for which of the two auth methods they accept.

## Pitfalls

- **Do not mix up the two product lines.** Old blog posts about "the Money Forward API" usually mean the closed ME API; current developer documentation means the Cloud API. They share a brand and nothing else — different hosts, different credentials, different availability.
- **Accounting API v1 and v2 are dead.** The official Cloud Accounting API v3 release notes record that v1/v2 service and support ended on 2025-10-01. Build on v3 only; old integration guides referencing v1/v2 endpoints will fail.
- **The generic URL pattern in the common docs does not fit Accounting v3.** The shared getting-started material shows an `api.biz.moneyforward.com/{service}/api/v1/...` shape (that is the shape used with the JWT examples), but the Accounting API has its own host and `/api/v3/` paths. Read each product reference's base URL/`servers` entry instead of composing URLs from the common pattern.
- **An API key is not the token.** Sending the `mf_api_pro_…` key directly to a data endpoint fails; exchange it at `POST /auth/exchange` first and send the 1-hour JWT. Conversely, do not call the exchange endpoint per request — cache the JWT (the exchange endpoint is limited to 100 requests/minute per key, and each data endpoint has its own rate limit on top).
- **Permissions live in two places.** The OAuth scope (`mfc/accounting/…`) and the App Portal's per-user app-integration permissions (plus, for some data such as linked-account transactions, a separate service permission) must both be in place; a missing portal-side permission surfaces as a 403 on one endpoint family while everything else works. With API-key auth, the key follows the issuer's member permissions per business instead.
- **API-key calls need the business identified explicitly** (`office_code` query parameter on Accounting calls, per implementation guides based on the v3 reference) — unverified against the official reference in this pass; confirm in the reference before relying on it. Under OAuth the business is fixed at authorization time.
