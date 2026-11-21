# Google Search Console connector recipe

Service: Google Search Console — Search Console API (Webmasters v3 resources + URL Inspection v1)
Purpose: read search performance (clicks, impressions, CTR, position), check sitemap status, and inspect URL indexing without opening the Search Console UI.
Verified against Google's official API documentation on 2026-11-21.

## API facts

- API host: `searchconsole.googleapis.com`
- Auth: OAuth 2.0 authorization code flow. **Service accounts are supported**: add the service account email as a user on the property in Search Console settings, then call with the service account credentials — no user sign-in needed.
  - Authorize: `https://accounts.google.com/o/oauth2/v2/auth`
  - Token: `https://oauth2.googleapis.com/token`
  - Scopes: `https://www.googleapis.com/auth/webmasters.readonly` (read), `https://www.googleapis.com/auth/webmasters` (adds sitemap submission and site management)
- Key endpoints:
  - List properties: `GET /webmasters/v3/sites` — returns each `siteUrl` and the caller's `permissionLevel`
  - List sitemaps: `GET /webmasters/v3/sites/{siteUrl}/sitemaps`
  - One sitemap's status: `GET /webmasters/v3/sites/{siteUrl}/sitemaps/{feedpath}` — includes `isPending`, `lastDownloaded`, `warnings`, `errors` and per-type `contents` counts. This is the call that answers "is my sitemap actually being fetched?"
  - Submit a sitemap: `PUT /webmasters/v3/sites/{siteUrl}/sitemaps/{feedpath}` (needs the non-readonly scope)
  - Search performance: `POST /webmasters/v3/sites/{siteUrl}/searchAnalytics/query` — body with `startDate`, `endDate`, `dimensions` (e.g. `query`, `page`, `date`, `device`), `rowLimit`
  - URL inspection: `POST /v1/urlInspection/index:inspect` — body `{inspectionUrl, siteUrl}`; returns index status and last crawl time

## Human setup (once)

1. In Google Cloud Console, enable the **Search Console API**.
2. Either create an OAuth client (Web application) and publish the consent screen to "In production" (Testing mode expires refresh tokens after 7 days), or create a service account and add its email as a user on each property.
3. Connect with an account that has access to the properties (Owner / Full / Restricted all work for reads; sitemap submission needs more than Restricted).

## Known blocker (Muse custom connectors, 2026-11-21)

Same Google OAuth cross-domain limitation as the AdSense recipe: the authorize endpoint (`accounts.google.com`) and token endpoint (`oauth2.googleapis.com`) span domains, so Muse's secure connector flow declines the registration. On platforms that accept Google's endpoints — or anywhere a service account credential can be registered — this recipe works as-is.

## Pitfalls

- `siteUrl` must match the property string exactly: URL-prefix properties include the protocol (`https://example.com/`), domain properties look like `sc-domain:example.com`. URL-encode it in the path. When unsure, call `GET /webmasters/v3/sites` and copy the string.
- Search Analytics data lags about 2–3 days; recent dates return partial rows.
- A sitemap can be listed and still never fetched: check `isPending` and `lastDownloaded`, not just presence.
