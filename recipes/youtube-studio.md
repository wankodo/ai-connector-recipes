# YouTube Studio analytics connector recipe

Service: YouTube — YouTube Data API v3 + YouTube Analytics API v2 (the data behind YouTube Studio)
Purpose: read a channel's own statistics and estimated revenue directly from YouTube, instead of relying on third-party estimates.
Verified against Google's official API documentation on 2026-11-21.

## API facts

- API hosts: `youtube.googleapis.com` (Data API v3), `youtubeanalytics.googleapis.com` (Analytics API v2)
- Auth: OAuth 2.0 authorization code flow (Analytics API has no API-key mode; the channel owner must sign in)
  - Authorize: `https://accounts.google.com/o/oauth2/v2/auth`
  - Token: `https://oauth2.googleapis.com/token`
  - Scopes (least privilege):
    - `https://www.googleapis.com/auth/youtube.readonly` — channel/video metadata and statistics; also required alongside the Analytics scopes for `reports.query`
    - `https://www.googleapis.com/auth/yt-analytics.readonly` — non-monetary analytics (views, watch time, audience)
    - `https://www.googleapis.com/auth/yt-analytics-monetary.readonly` — adds estimated revenue and ad metrics. Request only when revenue is actually needed.
- Key endpoints:
  - Own channel: `GET /youtube/v3/channels?mine=true&part=snippet,statistics` — subscriber count, total views, video count
  - Video list: `GET /youtube/v3/search?forMine=true&type=video&part=snippet` or `playlistItems` on the channel's uploads playlist
  - Analytics report: `GET /v2/reports` on `youtubeanalytics.googleapis.com`
    - `ids=channel==MINE`, `startDate`, `endDate` (YYYY-MM-DD)
    - `metrics`: e.g. `views`, `estimatedMinutesWatched`, `averageViewDuration`, `subscribersGained`, and (monetary scope) `estimatedRevenue`
    - `dimensions`: e.g. `day`, `video`; `sort=-views` etc.
    - Response kind: `youtubeAnalytics#resultTable` with `columnHeaders` and `rows`

## Human setup (once, in Google Cloud Console)

1. Enable **YouTube Data API v3** and **YouTube Analytics API**.
2. Configure the OAuth consent screen and **publish to "In production"** (Testing mode expires refresh tokens after 7 days).
3. Create an OAuth client (Web application) and register your platform's redirect URI.
4. Connect with the Google account that owns the channel (or has Studio permissions for it).

## Known blocker (Muse custom connectors, 2026-11-21)

Same Google OAuth cross-domain limitation as the other Google recipes in this collection: Muse's secure connector flow declines the registration because the authorize and token endpoints span domains. Workarounds: read Studio through the signed-in browser, use a third-party analytics connector (e.g. vidIQ) as an approximation, or use this recipe on a platform whose OAuth flow accepts Google's endpoints.

## Pitfalls

- Third-party tools' headline view counts can differ from Studio's own numbers. The Analytics API is the source of truth; quote it, not the headline.
- Analytics data is not final for roughly 2–3 days; treat the most recent days as provisional.
- Data API quota defaults to 10,000 units/day; `search.list` costs 100 units per call, so prefer `playlistItems` (1 unit) for listing uploads.
- `estimatedRevenue` is an estimate and is only available for monetized channels/periods.
