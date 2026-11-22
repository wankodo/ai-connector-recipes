# Stripe connector recipe

Service: Stripe — Stripe API (REST, v1 resources under `https://api.stripe.com/v1`)
Purpose: read customers, charges, PaymentIntents, products, invoices, balance and payouts for an agent's own Stripe account without opening the Stripe Dashboard, and — only with explicit human approval — prepare write operations such as creating products or invoices.
Verified against Stripe's official API documentation on 2026-11-23.
Status: 構築可（見込み・登録自体は未検証） — buildable in prospect as a Muse custom connector using the API-key route (direct key hand-off); the registration itself has not been verified. Separately, Muse also ships an official Stripe connector (via Stripe's official MCP service, OAuth sign-in, no API key shared in chat); that is a different route from the custom-connector recipe below.

## API facts

- API host: `api.stripe.com`
- Base URL: `https://api.stripe.com` — the API is organised around REST, has predictable resource-oriented URLs, accepts form-encoded request bodies, returns JSON-encoded responses, and uses standard HTTP response codes. All requests must be over HTTPS; unauthenticated requests fail.
- Auth: API keys, managed in the Stripe Dashboard. Send the key as the HTTP Basic Auth username with an empty password, or as a bearer token (`Authorization: Bearer <key>`).
  - Secret key: `sk_test_` / `sk_live_` — grants access to all Stripe API resources. Stripe creates a live-mode secret key, but recommends restricted keys instead to protect the business.
  - Restricted API key (RAK): `rk_test_` / `rk_live_` — a key with per-resource permissions. In the Dashboard you set each resource to **None**, **Read**, or **Write** (default is **None** for all; Write implies Read). Mapping: `GET` needs Read, `POST` / `DELETE` need Write. All Stripe APIs support restricted keys, and Stripe presents RAKs as the recommended key type for keys given to AI agents. If a RAK lacks the permission for a request, Stripe returns an invalid request error.
  - Publishable key: `pk_test_` / `pk_live_` — client-side only (Stripe.js / Elements / mobile SDKs); it cannot read account data and is not a connector credential.
  - Never embed a secret or restricted key in source code or a client-side app; keep it in a secrets vault or environment variables.
- Test mode / live mode (sandboxes): the API key you use determines whether a request runs in live mode or in a sandbox. Test-mode secret keys have unrestricted access to their sandboxes. After creating an account you can test Stripe services in a sandbox; live mode is separate and requires business verification and activation (see Human setup). Do not mix a test key with live data or vice versa — check the key prefix (`_test_` vs `_live_`) before every run.
- Scope equivalent: there is no OAuth scope string for the API-key route. The effective scope is the RAK's per-resource permission set (None / Read / Write per resource). For an agent, grant Read only on the resources it needs (e.g. Customers, Charges, PaymentIntents, Products, Invoices, Balance, Payouts) and leave everything else at None.
- Key endpoints (agent use, v1):
  - Charges: `GET /v1/charges`, `GET /v1/charges/{id}`
  - PaymentIntents: `GET /v1/payment_intents`, `GET /v1/payment_intents/{id}` (create / confirm / capture are writes — see Pitfalls)
  - Customers: `GET /v1/customers`, `GET /v1/customers/{id}` (`POST /v1/customers` is a write)
  - Products and Prices: `GET /v1/products`, `GET /v1/products/{id}`, `GET /v1/prices`
  - Invoices: `GET /v1/invoices`, `GET /v1/invoices/{id}`
  - Subscriptions: `GET /v1/subscriptions`, `GET /v1/subscriptions/{id}`
  - Refunds: `GET /v1/refunds`, `GET /v1/refunds/{id}` (creating a refund is a write)
  - Balance: `GET /v1/balance`, `GET /v1/balance_transactions`
  - Payouts: `GET /v1/payouts`, `GET /v1/payouts/{id}` (creating a payout is a write)
  - Account self-check: `GET /v1/account` — the account that owns the key
- Rate limits (official Rate limits page, per Stripe account, requests per second unless noted):
  - Global: live mode 100 requests/second; sandbox 25 requests/second
  - Individual API endpoints (unless otherwise noted): 25 requests/second
  - Files API: 20 read and 20 write requests/second; Search API: 20 read requests/second
  - Payouts API: 15 create requests/second, 30 concurrent requests per business
  - Default global concurrency limit for standard API traffic in live mode: 50 in-progress requests
  - Exceeding a limit returns `429 Too Many Requests` with a `Stripe-Rate-Limited-Reason` header (e.g. `global-rate`, `endpoint-rate`, `global-concurrency`); retry with exponential backoff and jitter
  - Read-request allocation: read (GET) requests must not exceed an average of 500 per transaction over a rolling 30-day period, with a minimum allocation of 10,000 read requests per month for every account; write requests have no allocation limit

## Human setup (once)

1. Create a Stripe account. You can test services in a sandbox right away.
2. To use a service in live mode, verify your business in the Dashboard and complete its activation requirements: provide information about your business, your product, and your relationship to the business. Stripe's "Know Your Customer" (KYC) obligations require it to collect and maintain this information for all users. Even if you do not accept payments, you must provide business information to verify your business. Public business information (business name and website URL, support email / phone / address, statement descriptor) is shown to customers on card statements and receipts.
   - Note on personal / individual use: an account for testing (sandbox) needs no activation, but live use is gated on the business verification above — there is no "personal, no-business-information" live route in the official set-up documentation checked. Whether a given individual / sole-proprietor form is accepted in a specific country is unverified (unverified) from the pages checked here; confirm in the Dashboard activation checklist for your country.
3. In the Dashboard, open the API keys page and create a **restricted key** for the connector: start from zero permissions, grant Read only on the resources the agent needs, leave the rest at None, and name the key after this integration so it can be revoked on its own. Do not hand over the unrestricted secret key (`sk_live_`).
4. Register that restricted key with the connector platform through its secure credential flow (direct key hand-off) — never paste it into chat, source code, or a file in a repository.
5. First call: `GET /v1/account` and `GET /v1/balance` with the restricted key, and confirm the response is the expected account and mode before granting any further permissions.

## Known blocker

No Stripe-side blocker for the API-key route, and the general Muse OAuth blocker does not apply here: in general, when an OAuth provider's authorize domain and token domain differ, Muse's connector flow declines registration with "token_url is on a host outside issuer_domain" — but the API-key route has no OAuth authorize / token endpoints at all, so that single-domain constraint is not triggered. The remaining wall is verification, not a known refusal: this recipe's custom-connector registration has not actually been attempted, so the status is 構築可（見込み・登録自体は未検証）, not "connected". The practical gate for live data is separate and human-side: a live-mode key only does useful work once the account's business verification / activation is complete (see Human setup).

## Pitfalls

- Payment execution is irreversible in effect: creating or confirming a PaymentIntent / charge, issuing a refund, or creating a payout moves real money. An agent connector should be read-centred by default (Read-only RAK), and any write — charge, refund, payout, invoice payment — must go through explicit human approval for that specific operation, amount and destination before it is sent. Never retry a failed or timed-out write automatically; check the record's status first, because the effect may already have happened.
- The key decides the mode: the same host serves test and live, so a wrong key in the wrong place means real money or silently test-only data. Assert on the key prefix and on the account returned by `GET /v1/account` before acting.
- A full secret key (`sk_live_`) is full account access with no scoping. If one leaks, rotate it immediately; a leaked restricted key's damage is bounded by its permissions, which is why the RAK is the only key an agent should hold.
- Request bodies are form-encoded, not JSON — hand-written clients that POST JSON fail in non-obvious ways. Responses are JSON.
- List endpoints paginate by cursor; a single list call is not the whole ledger. For bulk analytics, note the read-request allocation (500 reads per transaction, rolling 30 days) and prefer filtered lists.
- A `429` without a `Stripe-Rate-Limited-Reason` header is not a rate limit — per Stripe it might be an object lock timeout (`lock_timeout`), which has a different cause (concurrent access to the same object); serialise mutations on the same object instead of just slowing down.
- Webhook signing secrets are a different credential from API keys; a connector that only calls the API does not need one, and should not collect one.
