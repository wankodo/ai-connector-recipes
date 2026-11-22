# freee connector recipe

Service: freee — freee Accounting API (会計 freee API), with sibling Public APIs for freee HR (人事労務), freee Invoice (請求書) and other freee products
Purpose: read and write accounting data for a company (事業所) — companies, deals (取引), account items (勘定科目), partners, trial balances — and, where needed, HR data such as employees and work records, without opening the freee web UI.
Verified against freee's official developer documentation on 2026-11-23.

Status: 構築可 (buildable) for your own companies via a private app — no review needed. Public distribution is 条件付き (conditional): it needs freee's app review and is subject to company-count caps (see Human setup). There is no Muse official connector for freee.

## API facts

- API host: `api.freee.co.jp`
  - Accounting (会計): base `https://api.freee.co.jp/api/1/`
  - HR (人事労務): base `https://api.freee.co.jp/hr/api/v1/` — the HR OpenAPI spec asks for the header `FREEE-VERSION: 2022-02-01` and states a limit of 5,000 requests/hour
  - Invoice (請求書) and other products (工数管理, 販売, 業務委託管理, IT管理, 申告, サイン, …) have their own Public APIs on the same host; per freee's official help table, which plans may use each API differs by product
- Auth: OAuth 2.0 authorization code flow only (no API key, no service account). All three OAuth endpoints sit on **one host**, `accounts.secure.freee.co.jp`:
  - Authorize: `https://accounts.secure.freee.co.jp/public_api/authorize` — parameters `response_type=code`, `client_id`, `redirect_uri`, `state` (CSRF token), plus `prompt=select_company` to have the user pick one company (事業所) during consent
  - Token: `https://accounts.secure.freee.co.jp/public_api/token` — `POST`, `application/x-www-form-urlencoded`, `grant_type=authorization_code` with `client_id`, `client_secret`, `code`, `redirect_uri`; refresh with `grant_type=refresh_token`
  - Revoke: `https://accounts.secure.freee.co.jp/public_api/revoke`
  - Access token: expires in 6 hours (`expires_in: 21600`). Refresh token: expires 90 days after issue and is **single-use** — every refresh returns a new refresh token and the old one stops working, so persist the newest one every time
  - Permissions are **not** passed as an OAuth `scope` parameter. They are set per product and per resource (read / write) on the app itself, in the app store's 権限設定 tab. The token response still reports a `scope` string (e.g. `read write default_read`). Effective access is the intersection of the app's permissions and the authorizing user's role inside the company
  - When `prompt=select_company` is used, the token response also returns `company_id` (and `external_cid`). Store `company_id` with the tokens: almost every endpoint requires it. Without `select_company`, the token can reach every company the user belongs to and no `company_id` comes back
- Key endpoints (Accounting, all under `/api/1`, all take `company_id` unless noted):
  - Current user: `GET /users/me` — the authorizing user and their companies; the natural first call alongside companies
  - Companies (事業所): `GET /companies` — list accessible companies and get a `company_id`; `GET /companies/{id}` — company detail, including fiscal years
  - Deals (取引, income/expense): `GET /deals`, `POST /deals`, `GET /deals/{id}`, `PUT /deals/{id}`; payments on a deal: `POST /deals/{id}/payments`
  - Manual journals (振替伝票): `GET /manual_journals`, `POST /manual_journals` — note journals do not appear in the receivables/payables reports; use deals for 債権・債務
  - Account items (勘定科目): `GET /account_items`, `POST /account_items`, `GET /account_items/{id}`, `PUT /account_items/{id}`
  - Masters an agent usually needs next: `GET /partners` (取引先), `GET /items` (品目), `GET /sections` (部門), `GET /wallet_txns` (口座明細 — see the bank-statement note under Pitfalls)
  - Reports: `GET /reports/trial_bs` (貸借対照表), `GET /reports/trial_pl` (損益計算書)
  - HR, for reference: `GET /hr/api/v1/employees`, work records under `/hr/api/v1/employees/{employee_id}/work_records`

## Human setup (once)

1. Create the app in the freee app store developer pages (`app.secure.freee.co.jp/developers/applications`, 新規追加). You need a freee account; a free developer test company (開発用テスト事業所) can be created for development, but a public app cannot be created from a test/demo company.
2. **Choose the app type at creation — it cannot be changed later:**
   - Private app (プライベートアプリ): for specific companies only. Capped at 5 companies, cannot be listed in the app store, and **needs no review**. This is the right type for a personal/own-company connector.
   - Public app (パブリックアプリ): for distribution to many companies. Capped at 20 companies by default; the cap is lifted only by listing in the app store or by applying to freee.
3. In 基本設定, set the callback URL (redirect URI) your platform gives you. For local testing only, freee allows `urn:ietf:wg:oauth:2.0:oob`; for anything real, use an HTTPS URL you own. Note the Client ID and Client Secret.
4. In 権限設定, tick only the resources and read/write access the connector actually needs (least privilege — freee explicitly asks for this, and it is checked at review).
5. Authorize as a freee user who belongs to the target company: they sign in, select the company, check the app's permissions and approve. Then exchange the code at the token endpoint and store access token, the (rotating) refresh token and `company_id` together.
6. Only if you go public: apply for app review (アプリ審査) from the app store. Review is required to list in the store, to serve 21+ companies, or to add permissions to an already-reviewed app. After submitting the required files it takes roughly 1–2 months per freee's official guide. A help page is mandatory for store listing.

## Known blocker

No Google-style blocker here. Muse's custom-connector flow declines registrations whose token URL is on a host outside the issuer domain ("token_url is on a host outside issuer_domain"), but freee's authorize, token and revoke endpoints are **all on `accounts.secure.freee.co.jp`**, so on domain grounds freee passes the rule that blocks every Google API. Two qualifications:

- Actual registration inside Muse's secure connector flow has **not been attempted (unverified)** — the domain check is the known wall and it does not apply, but no completed Muse registration is claimed here.
- Separately, freee itself publishes an official MCP server (`freee/freee-mcp`, with a remote MCP endpoint) — that is freee's official MCP, not a Muse official connector. On a platform that accepts remote MCP servers it is the shortest route and it handles this same OAuth flow for you.

Plan and app-policy limits that shape what a connector may do:

- The Accounting API itself is available on **all plans** (全プラン), for corporate and sole-proprietor (個人事業主) accounts alike, per freee's official help table — the same table gives all plans for the HR and Invoice APIs too. What you can actually call still follows the company's contracted product features.
- A daily API call cap applies per company × app and varies by plan; the exact figures were not re-verified against the official reference in this pass (unverified) — check the reference's specification section before designing bulk jobs.
- At public-app review, freee may designate an app a "plan-limited app" (プラン限定アプリ) if it implements advanced workflow, advanced reconciliation (高度な消込), advanced corporate-card linkage, received-invoice management, or Webhooks — such apps may then be offered only to companies on 法人アドバンス / 法人エンタープライズ plans, or 個人プレミアム for sole proprietors. This designation applies only at public review, not to private apps.

## Pitfalls

- Refresh tokens rotate and are single-use: if two tools share one refresh token, the first refresh silently breaks the other. Give each connector its own authorization, and always overwrite the stored refresh token with the one from the latest token response.
- A private app with **no boxes ticked** in 権限設定 behaves as if everything were ticked (per freee's official error-reference). Tick the intended permissions explicitly instead of leaving it blank.
- `company_id` discipline: with `prompt=select_company` the token is bound to the one company the user picked. Users often belong to several companies and freee otherwise acts on the last-used company — pin and verify `company_id` before any write.
- Bank statements: a public app calling `GET /api/1/wallet_txns` needs a separate bank-statement application and security check; a private app does not. Financial-service apps likewise need a separate application (public only).
- Adding permissions to a reviewed public app invalidates the old access pattern: users must re-authorize for the new scope (calls to newly-scoped endpoints return 403 until they do).
- The old OAuth endpoints on `secure.freee.co.jp/oauth/*` and `api.freee.co.jp/oauth/*` were retired in December 2020 — any sample code using them is stale; use the `accounts.secure.freee.co.jp/public_api/*` endpoints above.
- Invoices moved products: since the October 2023 invoice-system change, invoices and quotations are created in the freee Invoice (freee請求書) API, not via the accounting API's legacy invoice endpoints.
