# Gmail connector recipe

Service: Gmail — Gmail API v1
Purpose: search, read and send mail, and manage labels and threads in a Gmail mailbox from an agent, without opening the Gmail UI.
Verified against Google's official API documentation on 2026-11-23.

Status: **公式コネクタあり** — Gmail already has an official Muse connector, connected and in real use in the maintainer's environment. You do not need to build this one yourself on Muse; the recipe below is for other platforms, and for understanding what the official connector sits on top of.

## API facts

- API host: `gmail.googleapis.com` (service endpoint `https://gmail.googleapis.com`; all paths below are under `/gmail/v1/users/{userId}/…`, and `userId` can be the special value `me` for the authenticated user)
- Auth: OAuth 2.0 authorization code flow, with the mailbox owner granting consent.
  - Authorize: `https://accounts.google.com/o/oauth2/v2/auth`
  - Token: `https://oauth2.googleapis.com/token`
  - Scopes (pick the narrowest that covers the job):
    - `https://www.googleapis.com/auth/gmail.readonly` — read messages, threads, labels and settings; no writes (restricted scope)
    - `https://www.googleapis.com/auth/gmail.metadata` — headers and labels only, no message body (restricted)
    - `https://www.googleapis.com/auth/gmail.send` — send mail only, no reading (sensitive scope)
    - `https://www.googleapis.com/auth/gmail.compose` — manage drafts and send mail (restricted)
    - `https://www.googleapis.com/auth/gmail.modify` — read/write including label changes, but no immediate permanent deletion (restricted)
    - `https://www.googleapis.com/auth/gmail.labels` — see and edit labels only (non-sensitive)
    - `https://mail.google.com/` — full access including permanent deletion; request only if the app truly needs to bypass the trash
  - Restricted scopes require Google's OAuth verification, and storing restricted-scope data on a server requires a security assessment.
  - **Service accounts:** a service account has no mailbox of its own. It can reach another user's mailbox only by impersonating that user under **domain-wide delegation**, which a Google Workspace domain administrator must grant for the specific scopes. That route exists only inside a Google Workspace domain — there is no administrator to grant it for a personal account.
  - **Personal accounts:** a regular (non-Workspace) Google account works through the normal user-consent OAuth flow above; that is the only route for personal mailboxes.
- Key endpoints (agent use):
  - Profile: `GET /gmail/v1/users/{userId}/profile`
  - List / search messages: `GET /gmail/v1/users/{userId}/messages` — returns message IDs; fetch bodies separately
  - Get a message: `GET /gmail/v1/users/{userId}/messages/{id}`
  - Attachment: `GET /gmail/v1/users/{userId}/messages/{messageId}/attachments/{id}`
  - Send: `POST /gmail/v1/users/{userId}/messages/send`
  - Change labels on one message: `POST /gmail/v1/users/{userId}/messages/{id}/modify`; in bulk: `POST /gmail/v1/users/{userId}/messages/batchModify`
  - Trash / untrash: `POST /gmail/v1/users/{userId}/messages/{id}/trash` / `…/untrash`
  - Threads: `GET /gmail/v1/users/{userId}/threads`, `GET /gmail/v1/users/{userId}/threads/{id}`, `POST /gmail/v1/users/{userId}/threads/{id}/modify`
  - Labels: `GET /gmail/v1/users/{userId}/labels`, `POST /gmail/v1/users/{userId}/labels`, `GET|PATCH|PUT|DELETE /gmail/v1/users/{userId}/labels/{id}`
  - Drafts: `GET|POST /gmail/v1/users/{userId}/drafts`, `POST /gmail/v1/users/{userId}/drafts/send`
  - Incremental sync: `GET /gmail/v1/users/{userId}/history` — list mailbox changes instead of re-listing everything; push alternative: `POST /gmail/v1/users/{userId}/watch` (Pub/Sub), stopped with `POST /gmail/v1/users/{userId}/stop`
- Quotas (from Google's official Gmail API usage-limits page, checked 2026-11-23 — note it now states limits **per minute**, not per second):
  - Per minute per project: 1,200,000 quota units
  - Per minute per user per project: 6,000 quota units
  - Daily billing threshold: 80,000,000 quota units per day per project — standard use below it is free; Google says charges above it are planned later in 2026, and the threshold cannot be raised
  - Per-method cost examples: `messages.list` 5, `messages.get` 20 (60 with `format=full`/`raw`/`html`), `messages.send` 100, `messages.modify` 5, `messages.batchModify` 50, `labels.list` 1, `history.list` 2, `threads.list` 10, `threads.get` 40 (120 with `format=full`), `watch` 100
  - Older figures still circulating in third-party guides (250 units per user per second, 1 billion units per day) could **not** be confirmed on the current official quota page — unverified (未確認); trust the per-minute figures above.
  - Also from the official page: max 500 recipients per message.

## Human setup (once)

1. In Google Cloud Console, enable the **Gmail API**.
2. Configure the OAuth consent screen and **publish the app to "In production"** — while it stays in "Testing", refresh tokens expire after 7 days and the connector silently breaks weekly. Restricted scopes (readonly / modify / compose / metadata) additionally need Google's verification before real users can grant them.
3. Create an OAuth client (Web application) and register the redirect URI your platform gives you.
4. Connect with the Google account that owns the mailbox, granting only the scopes the job needs (start with `gmail.readonly`; add `gmail.send` only if the agent must send).
5. Workspace alternative: a domain admin can instead grant a service account domain-wide delegation for the chosen scopes, after which it impersonates users in that domain. Personal accounts cannot use this route.
6. On Muse specifically: skip all of the above and use the official Gmail connector (see Status).

## Known blocker (Muse custom connectors, 2026-11-23)

Building Gmail as a *custom* connector on Muse hits the same Google OAuth cross-domain wall as the other google-* recipes: the authorize endpoint is on `accounts.google.com` while the token endpoint is on `oauth2.googleapis.com`, so Muse's secure connector flow declines the registration with "token_url is on a host outside issuer_domain". This does not matter in practice on Muse, because the official Gmail connector already exists and works. On platforms that accept Google's endpoints — or anywhere a Workspace service account with domain-wide delegation can be registered — this recipe works as-is.

## Pitfalls

- Messages are immutable: once created, a message cannot be changed — only the labels applied to it can (`messages.modify`). To "edit" a sent mail you must delete and resend; to edit a draft, replace it via the drafts resource.
- `messages.list` returns IDs only, and body fetches are the expensive part: `messages.get` costs 20 units, or 60 with `format=full`/`raw`/`html`. List first, fetch only what you need, and prefer `history.list` (2 units) for incremental sync over re-listing the mailbox.
- Sending is the priciest common call at 100 units (`messages.send`, `drafts.send`), and the per-user budget is 6,000 units per minute — batch and back off (truncated exponential backoff is Google's own recommendation) instead of polling in a tight loop.
- `gmail.modify` cannot permanently delete: trash is fine, but immediate permanent deletion of messages/threads needs the full `https://mail.google.com/` scope. Don't request it just to tidy up.
- Some settings operations are Workspace-only: e.g. the `gmail.settings.sharing` scope is available only to Google Workspace customers using a service account with domain-wide delegation, and `users.settings.delegates` methods are likewise only available to domain-wide-delegated service account clients.
- Labels are the only organisation mechanism — there are no folders. System labels (`INBOX`, `SENT`, `TRASH`, `SPAM`, …) cannot be deleted or modified, though some (like `INBOX`) can be applied to or removed from messages.
