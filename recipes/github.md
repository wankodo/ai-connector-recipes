# GitHub connector recipe

Service: GitHub — GitHub REST API
Purpose: read and write repositories, file contents, issues and pull requests, and search GitHub from an agent, without driving the GitHub web UI for every operation.
Verified against GitHub's official API documentation on 2026-11-23.

Status: **official connector available （公式コネクタあり）** — Muse already ships an official GitHub connector (GitHub MCP based), and it is connected and in real use in the maintainer's environment. You do not need to build a custom connector for normal GitHub work on Muse. The API facts below are for agents on other platforms, and for understanding what the official connector is doing underneath.

## API facts

- API host: `api.github.com` — all REST API access is over HTTPS at `https://api.github.com`. Send `Accept: application/vnd.github+json` and `Authorization: Bearer <token>`; most endpoints also take an `X-GitHub-Api-Version` header.
- Personal accounts: **supported**. A personal (individual) account can create personal access tokens, authorize OAuth Apps, install GitHub Apps on its own repositories, and call the REST API against its own repositories. This is not an organization-only API.
- Auth — four routes, two permission models:
  - **Fine-grained personal access token** — GitHub's recommended token type. Tied to the user who generated it, limited to resources owned by a single user or organization, further limited to specific repositories, and granted specific per-resource permissions (for example Contents read/write, Issues read/write, Pull requests read/write; Metadata read is the baseline). It cannot do everything a classic token can: for example it cannot access multiple organizations at once, and a few REST endpoints are classic-only — check the token types listed on each endpoint's doc page.
  - **Personal access token (classic)** — tied to the user, limited by broad OAuth scopes (for example `repo`, `workflow`). A classic token can reach every repository in the organizations the user can access plus all of the user's personal repositories, which is why GitHub recommends fine-grained tokens instead whenever possible.
  - **GitHub App** — the preferred app type in GitHub's own docs. Uses fine-grained permissions instead of broad scopes, the installer controls which repositories it can access, it can act independently of a user (installation access token) or on behalf of a user (user access token), and it uses short-lived tokens. Installing it on an organization needs an organization owner (or repository admin permissions, depending on the install); it can also be installed on a personal repository.
  - **OAuth App** — the older app type. Both GitHub Apps and OAuth Apps use OAuth 2.0, but an OAuth App acts as the authorizing user, uses broad OAuth scopes (an app that needs to read repository contents needs `repo`, which also lets it edit contents and settings), can access every repository that user can access, and its tokens do not expire until revoked (unless expiring tokens are enabled — unverified in this check). GitHub prefers GitHub Apps over OAuth Apps except for enterprise-level resources a GitHub App cannot yet be permissioned against.
  - OAuth endpoints (OAuth Apps, and GitHub App user access tokens):
    - Authorize: `GET https://github.com/login/oauth/authorize`
    - Token: `POST https://github.com/login/oauth/access_token`
    - Note both live on the same host, `github.com` — see Known blocker.
- Rate limits (primary, REST API):
  - Unauthenticated: 60 requests per hour (per originating IP)
  - Authenticated user (personal access token, or a GitHub App / OAuth App acting on the user's behalf): 5,000 requests per hour, shared across those methods; 15,000 per hour in the GitHub Enterprise Cloud cases described in GitHub's rate-limit doc
  - GitHub App installation access token: minimum 5,000 requests per hour, scaling with repository and organization-user counts above 20, capped at 12,500 per hour for non-Enterprise-Cloud installations (15,000 per hour on GitHub Enterprise Cloud)
  - `GITHUB_TOKEN` in GitHub Actions: 1,000 requests per hour per repository
  - Search endpoints have their own, tighter limit: authenticated 30 requests per minute for all search endpoints except Search code, which requires authentication and is limited to 10 requests per minute; unauthenticated search is 10 requests per minute. A search returns at most 1,000 results.
  - Secondary limits also apply (for example at most 100 concurrent requests, and content-generating requests capped per minute / per hour) and can change without notice. Read the `x-ratelimit-limit` / `x-ratelimit-remaining` / `x-ratelimit-reset` response headers, and treat `GET /rate_limit` as an overview only — the headers are authoritative.
- Key endpoints (agent-shaped subset):
  - Authenticated user and repositories: `GET /user`, `GET /user/repos`, `GET /repos/{owner}/{repo}`, `POST /user/repos` (create a repository for the authenticated user), `POST /orgs/{org}/repos`
  - Contents: `GET /repos/{owner}/{repo}/contents/{path}`, `PUT /repos/{owner}/{repo}/contents/{path}` (create or update a file; content is Base64-encoded, and updating needs the file's blob `sha`), `DELETE /repos/{owner}/{repo}/contents/{path}`
  - Issues: `GET /repos/{owner}/{repo}/issues`, `POST /repos/{owner}/{repo}/issues`, `GET /repos/{owner}/{repo}/issues/{issue_number}`, `PATCH /repos/{owner}/{repo}/issues/{issue_number}`
  - Pull requests: `GET /repos/{owner}/{repo}/pulls`, `POST /repos/{owner}/{repo}/pulls`, `GET /repos/{owner}/{repo}/pulls/{pull_number}`, `PATCH /repos/{owner}/{repo}/pulls/{pull_number}`
  - Search: `GET /search/repositories`, `GET /search/code`, `GET /search/issues` (issues and pull requests together — there is no separate `/search/pulls`), `GET /search/users`, `GET /search/commits`
  - Rate limit status: `GET /rate_limit`

## Human setup (once)

For the Muse official connector (the normal route):

1. Connect GitHub from Muse's connector flow and sign in with the GitHub account that owns (or can access) the repositories.
2. Approve the access the connector asks for. After that, ordinary reads and file pushes work through the connector — but see Known blocker for the two operations that do not.

For a token-based custom connector (other platforms, or a self-built lane):

1. In GitHub, open **Settings → Developer settings → Personal access tokens → Fine-grained tokens** and generate a token: pick the single resource owner, pick only the repositories the agent needs, and grant only the permissions it needs (for example Contents and Pull requests read/write, Metadata read).
2. Register that token as a bearer token against `https://api.github.com` on your platform. A token is a credential: store it only in the platform's secure credential store, never in a recipe, a repo, or a prompt.
3. If you build an app instead, prefer a GitHub App over an OAuth App (fine-grained permissions, per-repository install, short-lived tokens), per GitHub's own guidance.

## Known blocker

**On Muse, do not try to route around the official connector — and expect two 403 walls inside it (verified in the maintainer's real use, 2026-11-21):**

1. **Repository creation is not possible through the official connector**: the create-repository call returns `403 Resource not accessible by integration`. Create the repository in the signed-in GitHub web UI instead. (The REST API itself does have `POST /user/repos` / `POST /orgs/{org}/repos` — this is an integration-permission boundary, not a missing API.)
2. **Pushing files under `.github/workflows/` is not possible through the official connector**: the push fails with the same `403` when the batch contains a workflow file; the same files without it push fine. Add workflow files in the web UI. This matches the API-level rule that modifying `.github/workflows/` needs extra permission (the `workflow` scope for OAuth App / classic tokens).
3. **Writes are approval-gated on Muse**: write operations through the connector surface a per-operation user approval card. That is the platform working as designed, not an error — do not retry a declined write through another route.

For custom connectors in general: Muse's secure connector flow cannot register an OAuth setup whose authorize domain and token domain differ. GitHub's OAuth endpoints are both on `github.com` (see API facts), so that specific cross-domain wall does **not** apply to GitHub on its face — whether a GitHub OAuth App custom connector actually registers end-to-end in Muse's flow is **unverified**, and unnecessary while the official connector exists. A token (bearer) custom connector is buildable in the general case.

See also the field note `field-notes/github-app-integration-limits.md`, which records wall 1 and 2 from direct experiment.

## Pitfalls

- A `403` from the connector is usually a permission boundary, not a rate limit and not a bug: check whether the operation is repository creation or a `.github/workflows/` write before debugging anything else. A rate-limit `403` / `429` instead comes with `x-ratelimit-remaining: 0`.
- Do not hand an agent a classic token with `repo` scope when a fine-grained token will do: `repo` reaches every repository the user can reach. One fine-grained token covers one resource owner only — work spanning a personal account and an organization needs a token per owner.
- Contents writes are one file per call, Base64-encoded, and an update without the current blob `sha` fails. Listing a directory returns at most 1,000 files.
- Search is not a list endpoint: 30 requests per minute (10 for code search), at most 1,000 results per search, and pull requests are found through `/search/issues` with `is:pr`, not a dedicated endpoint.
- Endpoint support varies by token type: each REST endpoint's doc page lists which fine-grained token types work and which permission set is required. A fine-grained token failing on one endpoint does not mean the token is broken — it may be a classic-only endpoint.
- Secondary rate limits are not checkable in advance and can change without notice. Pace bulk work, honour `retry-after`, and back off exponentially instead of retrying in a tight loop — continuing to call while rate-limited can get the integration banned.
