# Rakuten connector recipe

Service: Rakuten — Rakuten Web Service (楽天ウェブサービス, public APIs) — not RMS Web Service (merchant-only, see below)
Purpose: search Rakuten Ichiba (楽天市場) items, books, travel hotels and related public catalogue data from an agent, without scraping Rakuten's storefront pages.
Verified against Rakuten's official developer documentation on 2026-11-23.

Status: **Buildable (構築可), with terms conditions** — there is no official Muse connector for Rakuten, but the public APIs authenticate with an Application ID + access key (API-key style), not OAuth, so the Muse OAuth single-domain blocker does not apply. Whether a given *use* is allowed is a separate question governed by Rakuten's terms (see Known blocker / Pitfalls): revenue and access-environment restrictions make some agent uses conditional, not free.

## API facts

- API host: `openapi.rakuten.co.jp` (documentation and app registration: `webservice.rakuten.co.jp`; official help: `webservice.faq.rakuten.net`)
- Auth: API key, no OAuth for the public catalogue APIs.
  - `applicationId` — the Application ID (App ID) issued per registered application; sent as a query parameter. Official per-API input tables mark it "Required along with access key".
  - `accessKey` — issued with the app registration; official input tables mark it required along with the App ID, and say it can be provided either in a header or as a query parameter.
  - `affiliateId` — optional shared parameter on affiliate-supporting APIs. One Affiliate ID per Rakuten member (official FAQ); passing it makes returned product URLs affiliate URLs, so rewards accrue under the Rakuten Affiliate Partner Agreement.
  - There are no OAuth scopes. Which APIs an app may call is set at app registration time (exact registration-form fields: unverified from public documentation alone).
- Two different API families — do not mix them up:
  - **Rakuten Web Service (楽天ウェブサービス)** — public, read-only catalogue APIs. Any Rakuten member can register an app and use them (Terms of Service, Article 2: the Developer must be a Rakuten Member). This is the family this recipe covers.
  - **RMS Web Service (RMS WEB SERVICE)** — merchant-only APIs for Rakuten Ichiba shop operators (orders, inventory, item management for their own shop), administered inside RMS (the shop management console), with different credentials from the public APIs. A public-API Application ID does not call RMS, and vice versa. The RMS specification is not published on the public developer site (it sits behind the merchant RMS login), so RMS endpoints, credential format and expiry are **unverified** here — do not build an RMS connector from third-party blog posts without the shop's own RMS documentation.
- API groups (from the official API list at `webservice.rakuten.co.jp/documentation`; every listed API is shown there with auth type "App ID approval"):
  - Rakuten Ichiba: Item Search, Genre Search, Attribute Search, Tag Search, Item Ranking, Item Price Navi (product search)
  - Rakuten Books: Books Total Search, Book / CD / DVD-Blu-ray / Foreign Book / Magazine / Game / Software Search, Books Genre Search
  - Rakuten Travel: Simple Hotel Search, Hotel Detail Search, Vacant Hotel Search, Get Area Class, Keyword Hotel Search, Get Hotel Chain List, Hotel Ranking
  - Rakuten Recipe: Category List, Category Ranking
  - Rakuten Kobo: eBook Search, Genre Search
  - Rakuten GORA: Golf Course Search, Golf Course Detail, Golf Plan Search
- Key endpoints for agent use (REST / JSON; version suffix is part of the path — copy the current version from each API's official documentation page, versions differ per API):
  - Ichiba Item Search: `GET https://openapi.rakuten.co.jp/ichibams/api/IchibaItem/Search/20170706?applicationId=[APPLICATION ID]&keyword=...` — keyword, item code, shop code or genre ID filter the search; excludes auction, flea-market, C2C and joint-purchase items. (A newer dated version may be listed on the official documentation page at build time — the newest version number itself is unverified in this recipe.)
  - Ichiba Genre Search: `GET https://openapi.rakuten.co.jp/ichibagt/api/IchibaGenre/Search/20170711?applicationId=[APPLICATION ID]&genreId=0`
  - Ichiba Item Ranking: `GET https://openapi.rakuten.co.jp/ichibaranking/api/IchibaItem/Ranking/20220601?applicationId=[APPLICATION ID]&genreId=...`
  - Books Total Search: `GET https://openapi.rakuten.co.jp/services/api/BooksTotal/Search/20170404?applicationId=[APPLICATION ID]&keyword=...`
- Quota (official FAQ, `webservice.faq.rakuten.net`): **each Application ID may make one request to an API per second.** Exceeding it once does not stop the ID immediately, but sustained over-limit requesting may get the Application ID suspended (official FAQ). Terms Article 7(3) also reserves Rakuten's right to set access counts, frequency and similar limits separately. Any higher limit or relaxation for high-performing affiliate sites is **unverified** — the official FAQ text checked for this recipe states only the 1 request/second rule.
- Registration limits (official FAQ): each user can register **up to five Applications** (five Application IDs); one App ID may be reused across multiple apps.

## Human setup (once)

1. Have a Rakuten member account (楽天会員). Terms Article 2 makes Rakuten membership the developer qualification; a personal member account is enough for the public APIs — no shop / merchant contract is needed (that is RMS, above).
2. On Rakuten Web Service (`webservice.rakuten.co.jp`), open the app ID issuance / new application registration flow, sign in with the Rakuten member ID, agree to the Rakuten Web Service Terms of Service, and register the required application information (Terms Article 4; Rakuten may refuse a registration).
3. Save the issued credentials: **Application ID** and **access key**. Terms Article 5: the developer is responsible for keeping the App ID from becoming known to others, and use under a matching ID is treated as the developer's own use.
4. Optional: confirm your **Affiliate ID** on the Web Service site after logging in (official FAQ gives the account affiliate ID page) and pass it as `affiliateId` only if the app is operated under the Rakuten Affiliate Partner Agreement.
5. Register the Application ID + access key as an API-key style custom connector / secret in your agent platform. There is no OAuth consent screen to configure.

## Known blocker

No platform blocker for the public APIs: authentication is an Application ID + access key, so Muse's custom-connector OAuth rule ("token_url is on a host outside issuer_domain") is not triggered — that rule bites OAuth services, and Rakuten Web Service is not one. (That Muse's connector registration accepts this exact key-in-header/query arrangement in every case is unverified; no live registration was attempted for this recipe.)

The real wall is **Rakuten's terms, not the technology**:

- Article 10(4): gaining income by using the Web Services in ways other than Rakuten Affiliate is forbidden, unless Rakuten explicitly permits it. A paid product, ad-funded service or other monetisation built on this API is therefore conditional on Rakuten's explicit permission — affiliate revenue through Rakuten Affiliate is the sanctioned route (Article 3).
- Article 10(10): using the Web Services in an environment where only specific people can access it is forbidden, unless Rakuten explicitly permits it. Read literally, a private, single-user agent setup sits in the zone this clause names; whether Rakuten applies it to personal agent use is **unverified** — do not assume a private connector is automatically in the clear, and ask Rakuten before relying on it.
- Article 10(6) / 10(7) / 10(9): no services competing with the Rakuten Group, no using returned information outside Rakuten's designated purposes or methods, and no storing API-obtained information in a place where unspecified/many people can share it.

## Pitfalls

- Pace everything: 1 request per second **per Application ID**, across your whole agent — parallel tool calls against Rakuten will breach it. Serialise calls and back off; sustained excess can get the App ID suspended.
- Both credentials are required together: current official input tables list `accessKey` alongside `applicationId` ("Required along with access key"). Older blog posts and code that send only `applicationId` — including posts using the legacy `app.rakuten.co.jp` host — are out of date; use `openapi.rakuten.co.jp` and the version shown on the current official documentation page for each API.
- `affiliateId` is optional and changes what you get: without it you still get catalogue data; with it, product URLs become affiliate URLs and the Rakuten Affiliate Partner Agreement applies. One member has exactly one Affiliate ID — it is not per-app.
- Displaying API data carries duties: Terms Article 8(4) requires applications built on the Web Services to link to Rakuten Ichiba / the Rakuten sites Rakuten specifies, and not to link elsewhere from the parts of the application that use the Web Services. The official FAQ also sets data-handling rules (including cache periods for price / availability data) — check the current FAQ before caching responses (exact current cache periods: unverified in this recipe).
- Five apps is the ceiling: plan for one shared Application ID for the agent rather than one per task, and never share or publish an App ID / access key — Rakuten treats any use under it as yours.
- RMS is a different door: if the user wants their *own shop's* orders or inventory, this recipe does not cover it. That needs the shop's RMS Web Service enrolment and RMS-issued credentials, whose official specification is not publicly readable (unverified here).
