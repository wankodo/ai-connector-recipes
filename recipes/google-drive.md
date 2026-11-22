# Google Drive connector recipe

Service: Google Drive — Google Drive API v3
Purpose: find, read, download and organise the user's Drive files (documents, notes and notebooks kept as Docs / Sheets / files) without opening the Drive UI.
Status: 公式コネクタあり — Muse ships an official Google Drive connector, and it is connected and in real use in the maintainer's environment. You do not need to build this one on Muse; the recipe below is for other platforms and for understanding what the official connector talks to.
Verified against Google's official API documentation on 2026-11-23.

## API facts

- API host: `www.googleapis.com/drive/v3` (all paths below are under this base)
- Auth: OAuth 2.0 authorization code flow
  - Authorize: `https://accounts.google.com/o/oauth2/v2/auth`
  - Token: `https://oauth2.googleapis.com/token`
  - Revoke: `https://oauth2.googleapis.com/revoke`
  - Scopes — pick the narrowest that fits; Google classifies them by sensitivity:
    - `https://www.googleapis.com/auth/drive.readonly` (**restricted**) — see and download all Drive files. The right default for a read-only agent connector.
    - `https://www.googleapis.com/auth/drive.file` (**non-sensitive**, recommended by Google for most apps) — create new files, or modify existing files, only for files the app created or that the user opened / shared with the app (e.g. via the Google Picker). It cannot browse the whole Drive.
    - `https://www.googleapis.com/auth/drive` (**restricted**) — see, edit, create and delete all Drive files. Only take this if the agent genuinely writes anywhere.
    - Narrower metadata-only options also exist: `drive.metadata.readonly` (restricted) sees file metadata but not contents.
  - Restricted scopes require Google's restricted-scope verification, and a security assessment if the app stores or transmits restricted-scope data on servers.
  - Service accounts: supported by the API, **but a service account has no storage quota and cannot own files** (per Google's shared-drives guide). It must upload into a shared drive it is a member of, or act on behalf of a human user via OAuth 2.0 / domain-wide delegation. Sharing a normal My Drive folder with a service account does not give its uploads anywhere to live. A service-account key is also a signed-request credential and cannot be registered in Muse's secure connector flow.
  - Personal (consumer) Google accounts work with the OAuth 2.0 user flow — no Google Workspace organisation is required for the API itself. Shared drives, however, are a Workspace feature, so the service-account route above is effectively Workspace-only.
- Key endpoints (agent use):
  - Who am I / quota: `GET /about` — user info, `storageQuota`, and supported import / export formats. The `fields` parameter is required.
  - List / search files: `GET /files` — query with `q` (e.g. `name contains '...'`, `mimeType = '...'`, `trashed = false`), page with `pageSize` (max 1000) and `pageToken` / `nextPageToken`
  - File metadata: `GET /files/{fileId}` — add `fields=` to get more than the minimal default subset
  - Download a stored file: `GET /files/{fileId}?alt=media` — only for files actually stored in Drive
  - Download (newer method): `POST /files/{fileId}/download`
  - Export a Google Workspace file: `GET /files/{fileId}/export?mimeType=...` — Docs / Sheets / Slides have no stored bytes, so they must be exported (e.g. to PDF or plain text); exported content is limited to 10 MB
  - Create / write: `POST /files` (create), `PATCH /files/{fileId}` (update metadata) — need a non-readonly scope
- Quotas (official usage-limits page, in abstract "quota units"; per-method costs: reads 5, lists 100, edits 50, downloads / exports 200):
  - Per minute per project: 1,000,000 quota units
  - Per minute per user per project: 325,000 quota units
  - Per day per project: 1 TB egress before charges apply; daily billing threshold 400,000,000 quota units (cannot be increased)
  - Exceeding a quota returns `403: User rate limit exceeded` (backend checks can also return `429`); Google recommends truncated exponential backoff
  - Storage-side constraints: max upload 750 GB per user per day across My Drive and shared drives, max single file upload 5 TB, max file copy 750 GB
  - Standard use of the API is free at the time of writing; Google states that exceeding the quota request limits is planned to incur charges later in 2026.

## Human setup (once)

1. In Google Cloud Console, create (or pick) a project and enable the **Google Drive API**.
2. Configure the OAuth consent screen (Google Auth Platform) and **publish the app to "In production"** — while it stays in "Testing", refresh tokens expire after 7 days and the connector silently breaks weekly.
3. Create an OAuth client (Web application) and register the redirect URI your platform gives you.
4. Connect with the Google account that owns (or can see) the files, granting only the narrowest scope above — `drive.readonly` for a read-only connector.
5. On Muse specifically, skip all of this: connect the official Google Drive connector instead (see Status above).

## Known blocker (Muse custom connectors, 2026-11-23)

Only affects building a *custom* Drive connector on Muse — the official connector (Status above) is unaffected and is the route to use there. If you do try to register a custom one, you hit the same Google OAuth cross-domain limitation as the AdSense / Search Console / Analytics recipes: Muse's secure connector flow requires every OAuth endpoint to sit on a single issuer domain, but Google's authorize endpoint is on `accounts.google.com` while the token endpoint is on `oauth2.googleapis.com`, so registration is declined with "token_url is on a host outside issuer_domain". On platforms that accept Google's endpoints, this recipe works as-is.

## Pitfalls

- Google Workspace files (Docs, Sheets, Slides) have no downloadable bytes: `?alt=media` fails on them — use `files.export` with a target `mimeType`, and remember the 10 MB export limit.
- `files.list` returns trashed files by default; add `trashed = false` to the `q` parameter unless you want the trash too. Responses only include the `fields` you ask for, and the default subset omits `nextPageToken` — pass `fields=` explicitly or pagination silently stops.
- Shared-drive files are invisible unless the call opts in: `supportsAllDrives=true` on per-file calls, and `corpora=allDrives` + `includeItemsFromAllDrives=true` on list / search.
- `drive.file` scope blindness is by design, not a bug: with that scope the app only sees files it created or that the user explicitly opened / shared with it. If an agent must search the user's whole Drive, it needs `drive.readonly` (restricted) instead.
- A service account uploading into a My Drive folder fails with `403 storageQuotaExceeded` even when the folder owner has free space — the service account itself has no storage quota and cannot own files (see API facts). Move the target into a shared drive.
- Quota burn is per method, not per request count: one `files.list` page costs 100 units and a download 200, against 5 for a metadata read. Batch and cache metadata; do not poll with list calls.
