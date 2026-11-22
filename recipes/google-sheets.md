# Google Sheets connector recipe

Service: Google Sheets — Google Sheets API v4
Purpose: read and write spreadsheet values (single ranges, batches, appends), inspect spreadsheet structure, and create spreadsheets without opening the Sheets UI.
Verified against Google's official API documentation on 2026-11-23.

Status: 公式コネクタあり — Muse already ships an official Google Sheets connector, and it is connected and in regular use in the maintainer's environment. This recipe documents the underlying API for agents on other platforms, and for anyone building a custom connector elsewhere.

## API facts

- API host: `sheets.googleapis.com` (service endpoint `https://sheets.googleapis.com`, discovery document at `https://sheets.googleapis.com/$discovery/rest?version=v4`)
- Auth: OAuth 2.0 authorization code flow.
  - Authorize: `https://accounts.google.com/o/oauth2/v2/auth`
  - Token: `https://oauth2.googleapis.com/token`
  - Scopes (pick the narrowest that fits; per Google's scopes page):
    - `https://www.googleapis.com/auth/spreadsheets.readonly` — see all the user's spreadsheets (Sensitive)
    - `https://www.googleapis.com/auth/spreadsheets` — see, edit, create and delete all the user's spreadsheets (Sensitive)
    - `https://www.googleapis.com/auth/drive.file` — only the specific Drive files the app uses; Google's recommended, non-sensitive option, but limited to files the app created or the user opened with it
    - `https://www.googleapis.com/auth/drive.readonly` / `https://www.googleapis.com/auth/drive` — full Drive read / read-write (Restricted). Sheets methods accept these Drive scopes as alternatives, and general Drive access (finding, moving, deleting files) is a Drive API job, not a Sheets API job — prefer the `spreadsheets` scopes unless the app genuinely needs Drive-wide access.
  - **Service accounts are supported**: share the spreadsheet with the service account's email address like any other user (Viewer for reads, Editor for writes) and call with the service account credentials — no user sign-in needed. Note that all calls from one service account count as a single user against the per-user quota below.
  - Personal accounts: the official quickstart's only account prerequisite is "A Google Account" — a Google Workspace organisation account is not required, and personal (consumer) Google accounts work.
- Quotas (official usage-limits page; per-minute quotas, refilled every minute, no daily cap while inside them):
  - Read requests: 300 per minute per project, 60 per minute per user per project
  - Write requests: 300 per minute per project, 60 per minute per user per project
  - Over the limit returns HTTP `429`; back off exponentially and retry after the minute resets. A single request that takes over 180 seconds to process times out. Google recommends keeping payloads to about 2 MB. Standard use is free; Google states that charges for exceeding quota limits are planned for later in 2026.
- Key endpoints (all under `/v4`):
  - Get spreadsheet metadata and structure: `GET /v4/spreadsheets/{spreadsheetId}` — sheet (tab) list, properties, named ranges; trim the response with `fields=`
  - Create a spreadsheet: `POST /v4/spreadsheets`
  - Structural / formatting changes: `POST /v4/spreadsheets/{spreadsheetId}:batchUpdate` — add or rename tabs, insert/delete dimensions, formatting; this is separate from value writes
  - Read one range: `GET /v4/spreadsheets/{spreadsheetId}/values/{range}` — range in A1 (or R1C1) notation, e.g. `Sheet1!A1:D5`; returns a `ValueRange`
  - Read several ranges at once: `GET /v4/spreadsheets/{spreadsheetId}/values:batchGet`
  - Write one range: `PUT /v4/spreadsheets/{spreadsheetId}/values/{range}?valueInputOption=RAW|USER_ENTERED` — body is a `ValueRange` with `values`; existing values in the range are overwritten
  - Write several ranges at once: `POST /v4/spreadsheets/{spreadsheetId}/values:batchUpdate`
  - Append rows after a table: `POST /v4/spreadsheets/{spreadsheetId}/values/{range}:append?valueInputOption=...` — add `insertDataOption=INSERT_ROWS` to push existing rows down instead of overwriting empty-looking rows below the table
  - Clear values: `POST /v4/spreadsheets/{spreadsheetId}/values/{range}:clear` (and `:batchClear` for several ranges)

## Human setup (once)

1. In Google Cloud Console, create (or pick) a project and enable the **Google Sheets API**.
2. Either create an OAuth client (Web application) and publish the consent screen to "In production" (Testing mode expires refresh tokens after 7 days), or create a service account and share each target spreadsheet with its email address.
3. Connect with an account that can access the spreadsheets, granting only the scope the job needs (`spreadsheets.readonly` for read-only work).
4. In Muse specifically, none of this is needed for Sheets itself: use the official Google Sheets connector, which is already connected in the maintainer's environment.

## Known blocker (Muse custom connectors, 2026-11-23)

Only relevant if you try to register Google Sheets as a *custom* connector in Muse instead of using the official one: Muse's secure connector flow requires every OAuth endpoint to sit on a single issuer domain, and registration is declined with "token_url is on a host outside issuer_domain" because Google's authorize endpoint is on `accounts.google.com` while the token endpoint is on `oauth2.googleapis.com`. Same wall as the other google-* recipes here. The official Muse connector is unaffected. On platforms that accept Google's endpoints — or anywhere a service account credential can be registered — this recipe works as-is.

## Pitfalls

- Scopes apply to the whole spreadsheet file, not to a single tab: Google's scopes page states Sheets API scopes cannot be limited to a specific sheet. To protect part of a spreadsheet, use a `ProtectedRange` in the sheet itself.
- `values.update` and `values.append` require `valueInputOption`. `USER_ENTERED` parses input like typed input (numbers, dates, formulas become live values/formulas); `RAW` stores the literal strings. Picking the wrong one is the most common cause of "my dates/formulas look wrong".
- `values.get` omits trailing empty rows and columns, so row counts from a read can be shorter than the sheet looks. Default `valueRenderOption=FORMATTED_VALUE` returns display strings, not raw numbers — request `UNFORMATTED_VALUE` (or `FORMULA`) when the numbers matter.
- A service account is one "user" for quota purposes: funnel all traffic through a single service account and the practical ceiling is 60 reads + 60 writes per minute, however many humans are behind it. Batch calls (`batchGet`, `values.batchUpdate`, `spreadsheets.batchUpdate`) each count as one request — batch aggressively.
- Deleting the spreadsheet file itself, moving it, or changing sharing is the Drive API, not the Sheets API; a Sheets-only scope will not do it.
