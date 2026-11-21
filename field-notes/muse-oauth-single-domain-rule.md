# Muse custom-connector OAuth requires all endpoints on one issuer domain (blocks every Google API)

Topic: Muse (custom connector registration) × Google OAuth
Verified on 2026-11-21 by direct experiment and the platform's error message.

## Situation

Registering a custom connector for a Google API (Analytics, AdSense) in Muse's secure connection flow, using the API-access registration form.

## What was tried

Registering Google's OAuth endpoints: authorize on `https://accounts.google.com/o/oauth2/v2/auth`, token on `https://oauth2.googleapis.com/token`.

## What happened (the wall)

Registration was declined: **"token_url is on a host outside issuer_domain"**. Muse's flow requires every OAuth endpoint to sit on a single issuer domain (or its subdomains). Google splits its endpoints across `accounts.google.com` and `oauth2.googleapis.com`, so **no Google API can be registered this way** — this one rule blocks AdSense, Analytics, Search Console, YouTube and the rest at once.

## Workaround

None inside that registration flow as of 2026-11-21. Working alternatives, per service:

- Use the signed-in browser session to read the web console.
- Route through an already-connected hub (e.g. Zapier / Composio) when it offers the service.
- Use an official MCP server where one exists (Google publishes one for Analytics, using Application Default Credentials).
- Use a service account where the API supports it (Search Console does; AdSense does not).

## Notes

- The per-service recipes in `recipes/` (google-adsense, google-analytics, google-search-console, youtube-studio) each record this blocker; this note is the shared root cause.
- If you hit a different platform with the same single-domain rule, expect the same result for every Google service.
