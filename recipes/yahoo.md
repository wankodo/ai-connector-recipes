# Yahoo! JAPAN connector recipe

Service: Yahoo! JAPAN — Yahoo! Developer Network (Yahoo!デベロッパーネットワーク) Web APIs: Shopping, YOLP (maps / local), Text Analysis (テキスト解析), News, and Yahoo! ID連携 (YConnect v2)
Purpose: search Yahoo! Shopping products, geocode and local-search Japanese places, run Japanese text analysis (morphology, kana-kanji conversion, keyphrases, NER and more), and optionally sign users in with a Yahoo! JAPAN ID — all from an agent, without opening the Yahoo! JAPAN web UI.
Verified against Yahoo! JAPAN's official developer documentation on 2026-11-23.

Status: **構築可 (buildable)** for the Application ID (Client ID) APIs — Shopping information search, YOLP and Text Analysis need only a Client ID sent as a request parameter, no OAuth. **条件付き (conditional)** for anything behind Yahoo! ID連携: store-operation Shopping APIs need a separate usage application (利用申請) and a linked store account, and YConnect sign-in needs a registered server-side / native application. There is **no official Muse connector** for Yahoo! JAPAN.

## API facts

- Developer portal: `developer.yahoo.co.jp` (Yahoo!デベロッパーネットワーク, run by LINEヤフー株式会社). The official API index (`/sitemap/`) currently lists exactly these families:
  - **Shopping (ショッピング)** — Yahoo! Shopping product data and EC / store-operation functions, plus affiliate support
  - **YOLP (地図)** — Yahoo! Open Local Platform: maps and local / regional information
  - **Text Analysis (テキスト解析)** — Japanese text analysis: morphological analysis, kana-kanji conversion, furigana (ルビ振り), proofreading support, dependency parsing, keyphrase extraction, natural-language understanding, named-entity extraction, name matching (名寄せ), text clustering
  - **News (ニュース)** — a "constructive comment ranking AI" (建設的コメント順位付けAI) from Yahoo! News comment moderation; this is the only News item in the index, not a general news-search API
  - **Yahoo! ID連携** — YConnect v2 sign-in / attribute access (below)
  - **Mail (メール)** — a reference guide for ISPs and mail-client developers, not a mailbox API
- **No auction API**: ヤフオク (Yahoo! Auctions) does not appear in the current official API index, and there is no application route for one. Do not build against the old `auctions.yahooapis.jp` endpoints. (Third-party reports date the old Auction Web API's end to January 2018; that date itself is unverified against a current official page.)
- Auth, two distinct shapes:
  - **Application ID (Client ID)** — most public APIs take the Client ID as a plain request parameter (`appid=...`) on every call. No token, no user sign-in.
  - **Yahoo! ID連携 v2 (YConnect)** — OAuth 2.0 authorization code flow + OpenID Connect, used for user attributes and for the store-operation Shopping APIs. It issues a Client ID **and** a Client Secret at application registration.
    - Issuer: `https://auth.login.yahoo.co.jp/yconnect/v2`
    - Authorize: `https://auth.login.yahoo.co.jp/yconnect/v2/authorization` (`response_type=code`, PKCE supported: `plain` and `S256`)
    - Token: `https://auth.login.yahoo.co.jp/yconnect/v2/token` (POST, Basic auth with Client ID : Client Secret)
    - UserInfo / attributes: `https://userinfo.yahooapis.jp/yconnect/v2/attribute` (Bearer access token)
    - Scopes: `openid` (required), `profile`, `email`, `address`
- Key endpoints (agent-relevant):
  - Shopping product search (no ID連携 needed): `GET https://shopping.yahooapis.jp/ShoppingWebService/V3/itemSearch?appid=<Client ID>&query=...` — JSON only; search by keyword, JAN code, genre category ID, brand ID or store ID. The same no-login family includes category ID lookup, product-code (detail) lookup, review search, high-rating trend ranking, keyword ranking v2 and recommendation modules.
  - Shopping store-operation APIs (ID連携 + access token required): orders (注文), recurring purchase (定期購入), products (商品), inventory (在庫), images (画像), product / SHP category / brand, store category, listing management (出品管理), design, inquiry management (問い合わせ管理), store inventory (店舗在庫). Orders, recurring purchase and inquiry management additionally require a separate usage application (利用申請が必要です).
  - YOLP geocoder: `GET https://map.yahooapis.jp/geocode/V1/geoCoder?appid=<Client ID>&query=<address>&output=json`
  - YOLP local search: `GET https://map.yahooapis.jp/search/local/V1/localSearch` — stores, facilities, events and review information by area / coordinates
  - YOLP also offers: reverse geocoder, weather (降水強度), zip-code search, place information, address directory, in-facility search, contents geocoder, distance between two points, datum conversion, elevation. Every YOLP request must carry the Client ID as the `appid` parameter.
  - Text Analysis: JSON-RPC style POST to `https://jlp.yahooapis.jp/jsonrpc` with the method and parameters (including the Client ID) in the request body; some functions also keep dedicated service URLs under `jlp.yahooapis.jp` (e.g. natural-language understanding v2 at `/NLUService/V2/analyze`). Per-request size caps apply and vary by function (e.g. 100 KB for NLU, 4 KB for named-entity extraction, per the official per-function pages).
  - **Official MCP server**: Yahoo! itself publishes a remote "Text Analysis MCP Server" (テキスト解析 MCP サーバ) that exposes the Text Analysis Web API to MCP-compatible clients, under the same Client ID handling, terms and rate limits as the Web API. Its connection URL / transport details were not captured in this check — unverified; read the linked page under `developer.yahoo.co.jp/webapi/jlp/` before wiring it up.
- Quotas and limits (official `利用制限について` page and per-family pages):
  - At most **10 applications** (Client IDs) per Yahoo! JAPAN ID.
  - An application unused for over a year may be deleted by Yahoo!; a deleted application cannot be revived, register a new one instead.
  - YOLP: **50,000 requests per day per application**; beyond that, the paid YOLP Premier plan is required.
  - Shopping Web API: flooding the same URL in a short time can get access suspended for a while — the official guidance figure is **1 query / second**.
  - Text Analysis: per-function request-count limits are published on its "利用回数の制限について" page; the exact current numbers were not captured in this check — unverified, read that page before bulk work.
  - APIs that read a user's personal data or act for a user require that user's authorization first; YConnect is the sanctioned flow for obtaining it.
- **Credit / attribution is mandatory**: every site or application built on these APIs must display Yahoo!'s prescribed credit — e.g. the text link `Webサービス by Yahoo! JAPAN` linking to `https://developer.yahoo.co.jp/sitemap/` — using the unmodified HTML snippet and placement rules from the official クレジット表示 page. Modifying the snippet or breaking the placement rules can get API use suspended.
- Commercial use: there is no single blanket answer — it is per family. Shopping explicitly supports affiliate monetisation through Value Commerce (バリューコマース). YOLP is the restrictive one: its licence can be terminated for, among other things, a service that is only open to a limited set of users / computers (unless free to end users), intranet use, and **a service from which the operator is judged to profit from users of the YOLP data** — those cases are directed to the paid YOLP Premier. A general commercial-use statement for the Text Analysis API beyond its terms pages is unverified.

## Human setup (once)

1. Get a **Yahoo! JAPAN ID** (free personal ID is enough to register as a developer; the official guide's first step is simply obtaining one — no corporate account is required for the Client ID APIs).
2. Sign in to Yahoo!デベロッパーネットワーク with that ID and open "新しいアプリケーションを開発" (develop a new application). Fill in the application name, site URL and description, review the Developer Network guideline (ガイドライン), agree, and register — the **Client ID (アプリケーションID)** is issued on completion.
3. For YConnect / ID連携, register the application with ID連携 enabled, choose the application type (server-side "Yahoo! ID連携 v2", or a native-app type with its SDK), set the callback / redirect URI exactly, and keep the issued Client Secret server-side. Attribute retrieval (属性取得API) and which attributes you may fetch are chosen at registration.
4. For Shopping store-operation APIs, register a Shopping-specific application (the portal warns that a generic Client ID without the store-operation scope note will not do), link the Yahoo! JAPAN ID to a store account registered in ストアクリエイターPro, and file the separate usage application for the order / recurring-purchase / inquiry APIs.
5. Before publishing anything, add the mandatory Yahoo! JAPAN credit (see API facts) in the prescribed form and position.

## Known blocker (Muse custom connectors, 2026-11-23)

There is no official Muse connector for Yahoo! JAPAN, so the route is a custom connector / direct API client.

- The Client ID APIs are the easy half: the credential is a query / body parameter against `shopping.yahooapis.jp`, `map.yahooapis.jp` and `jlp.yahooapis.jp`, with no OAuth dance at all — the same "buildable" shape as other API-key connectors.
- YConnect does **not** trip the usual Muse cross-domain rule on its core flow: unlike Google, both the authorize endpoint and the token endpoint live on the issuer host `auth.login.yahoo.co.jp`, so the known `token_url is on a host outside issuer_domain` failure should not occur. Two caveats, both unverified against an actual Muse registration attempt: (1) the UserInfo endpoint sits on a different host, `userinfo.yahooapis.jp` — whether Muse's flow tolerates that is unconfirmed; (2) no registration was actually attempted for this recipe, so treat "YConnect is registrable in Muse" as unverified until someone tries it.

## Pitfalls

- **YOLP data must not be fed to generative AI**: the YOLP terms (revised 2026-11-16) expressly forbid inputting YOLP-provided information into a generative AI, using it to develop or train AI, or using it for information analysis including ML training data. An agent may call YOLP to answer a user, but must not pipe its output into model training / analysis pipelines. Read the current 利用方法 clauses before designing any caching or analysis around YOLP output — the same section also restricts intentionally saving / caching YOLP data and requires always fetching fresh data.
- Client ID is not a secret for the public APIs — it is designed to travel in request URLs — but the YConnect Client Secret is one; never put the secret in a client-side app or a URL.
- The redirect URI for YConnect must match a pre-registered value exactly; a near-match fails the authorization step.
- Shopping has two worlds that are easy to confuse: information APIs (product search etc., Client ID only, anyone can call) and store-operation APIs (your own store's orders / inventory, OAuth token + linked store account + per-API application). A Client ID alone will never return order data.
- Do not resurrect dead surfaces: the auction API and the old OpenID 2.0 endpoint (`open.login.yahoo.co.jp`) are gone; YConnect v2 is the only current sign-in system. OAuth 1.0 documentation still exists on the portal for legacy APIs — unverified whether any current API still requires it; prefer YConnect v2 where a token is needed.
- Exceeding a rate limit returns an error rather than a slow-down, and Shopping may lock you out for a period after burst access to one URL — pace calls (1 query/second for Shopping) instead of retrying immediately.
- Usage is aggregated per application and visible in the portal by day; if an agent shares one Client ID across tools, their quotas are shared too — register separate applications (up to 10) per tool instead.
