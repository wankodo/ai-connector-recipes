# Google Calendar connector recipe

Service: Google Calendar — Google Calendar API v3
Purpose: list calendars, read and search events, check free/busy time, and create, update or delete events without opening the Calendar UI.
Verified against Google's official API documentation on 2026-11-23.

Status: **Official Muse connector available**（公式コネクタあり）— Google Calendar is already an official Muse connector, connected and in daily use in the maintainer's environment. Prefer it over a custom build; the API facts below are for platforms without that connector, and for understanding what the official connector sits on.

## API facts

- API host / base URI: `https://www.googleapis.com/calendar/v3` (all method URIs below are relative to it)
- Accounts: personal Google accounts work — Google's official quickstart prerequisites are only a Google Cloud project and "a Google account with Google Calendar enabled"; no Google Workspace account is required for the user OAuth flow. Domain-wide delegation and domain resources are Workspace-only extras.
- Auth: OAuth 2.0 authorization code flow.
  - Authorize: `https://accounts.google.com/o/oauth2/v2/auth`
  - Token: `https://oauth2.googleapis.com/token`
  - Scopes (least privilege first, per Google's official scope list):
    - `https://www.googleapis.com/auth/calendar.readonly` — see and download any calendar you can access
    - `https://www.googleapis.com/auth/calendar.events.readonly` — view events only
    - `https://www.googleapis.com/auth/calendar.events` — view and edit events on all your calendars
    - `https://www.googleapis.com/auth/calendar.freebusy` — view availability only (enough for free/busy checks)
    - `https://www.googleapis.com/auth/calendar` — full access: see, edit, share and permanently delete all calendars you can access; request only when calendar management itself is needed
  - **Service accounts are supported, with a hard boundary**: a service account can only reach calendars that have been shared with it (share the calendar with the service account's email address, with the permission level the work needs), or, inside Google Workspace, calendars it reaches through domain-wide delegation granted by a Workspace admin. It cannot see a user's other calendars, and an unshared calendar is simply invisible to it.
- Key endpoints (agent use first):
  - List calendars: `GET /users/me/calendarList` — returns each calendar's `id`, `summary` and the caller's `accessRole`. Call this first and copy IDs from here.
  - One calendar list entry: `GET /users/me/calendarList/{calendarId}`
  - Calendar metadata: `GET /calendars/{calendarId}`
  - List events: `GET /calendars/{calendarId}/events` — query with `timeMin` / `timeMax` for a window; use the literal `primary` as the `calendarId` for the signed-in user's primary calendar
  - One event: `GET /calendars/{calendarId}/events/{eventId}` — to find an event by its iCalendar ID instead, call events list with the `iCalUID` parameter
  - Create an event: `POST /calendars/{calendarId}/events` — body is an Event resource (`summary`, `start`, `end`, plus optional `description`, `location`, `attendees`)
  - Replace an event: `PUT /calendars/{calendarId}/events/{eventId}` (full update, no patch semantics)
  - Partial update: `PATCH /calendars/{calendarId}/events/{eventId}` — note the quota cost under Pitfalls
  - Delete an event: `DELETE /calendars/{calendarId}/events/{eventId}`
  - Recurring instances: `GET /calendars/{calendarId}/events/{eventId}/instances`
  - Free/busy: `POST /freeBusy` — returns busy blocks for a set of calendars without event details
  - Sharing rules (ACL): `GET /calendars/{calendarId}/acl`, `POST /calendars/{calendarId}/acl`
- Quotas (Google's official usage-limits page, checked 2026-11-23):
  - Per minute per project: 10,000 requests
  - Per minute per user per project: 600 requests
  - Quotas are calculated per minute using a sliding window; exceeding either returns `403 usageLimits` or `429 usageLimits` — retry with truncated exponential backoff
  - Daily billing threshold: 1,000,000 requests per day per project (usage under it incurs no charge, and this threshold cannot be increased)
  - All standard use of the Calendar API is available at no additional cost; Google states exceeding the quota limits is planned to incur charges later in 2026

## Human setup (once)

1. In Google Cloud Console, create (or pick) a project and enable the **Google Calendar API**.
2. Configure the OAuth consent screen (Google Auth Platform) and **publish the app to "In production"** — while it stays in "Testing", refresh tokens expire after 7 days and the connector silently breaks weekly.
3. Create an OAuth client (Web application) and register the redirect URI your platform gives you.
4. Connect with the Google account that owns or can access the calendars, granting only the narrowest scope above that covers the work (readonly for agenda reads, `calendar.events` for event writes).
5. Service-account route instead: create a service account, then share each target calendar with its email address at the permission level needed — or, in Google Workspace, have an admin grant it domain-wide delegation. Nothing else is reachable by that account.
6. On Muse specifically, skip all of this: connect the official Google Calendar connector (see Status above).

## Known blocker (Muse custom connectors, 2026-11-23)

This blocker applies only to building a *custom* connector — the official Muse connector above is unaffected and works. Muse's secure connector flow requires every OAuth endpoint to sit on a single issuer domain or its subdomains. Google's authorize endpoint is on `accounts.google.com` while the token endpoint is on `oauth2.googleapis.com`, so a custom registration is declined with "token_url is on a host outside issuer_domain" — the same wall as the other google-* recipes in this collection. On platforms that accept Google's endpoints, this recipe works as-is.

## Pitfalls

- Get calendar IDs from `calendarList.list` and copy them exactly; guessing an ID from a calendar's display name fails. `primary` works only for the authenticated user's own primary calendar — a service account must be addressed at the real ID of a calendar shared with it.
- `PATCH` is expensive: each patch request (events, calendars, calendarList, ACL) consumes three quota units per Google's reference; Google recommends a `get` followed by an `update` where that fits.
- A service account that "sees nothing" is almost always just not shared in: it has no view into a user's calendar list until the calendar is shared with it (or Workspace domain-wide delegation is granted).
- Quota errors arrive as `403 usageLimits` or `429 usageLimits` and are rate limits, not permission errors — back off exponentially instead of re-asking for consent.
- Whether the official connector exposes every v3 method above (e.g. ACL management, watch channels) is unverified（未確認）; this recipe verifies the API surface, not the connector's tool list.
