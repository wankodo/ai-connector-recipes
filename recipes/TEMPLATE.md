# <Service name> connector recipe

Service: <provider + API name and version>
Purpose: <what the connector lets an agent do, in one line>
Verified against <official documentation name> on <YYYY-MM-DD>.

## API facts

- API host(s): `<host>`
- Auth: <OAuth 2.0 authorization code flow / API key / other>
  - Authorize: `<url>` (OAuth only)
  - Token: `<url>` (OAuth only)
  - Revoke: `<url>` (OAuth only)
  - Scope (least privilege): `<scope>`
- Key endpoints:
  - `<METHOD> <path>` — <what it returns>
  - Request/response notes and a minimal example with placeholder IDs only.

## Human setup (once)

1. <Console steps: create project, enable the API, configure consent, create the client, register the redirect URI.>
2. <Anything account-side: roles, property access, etc.>

## Known blockers

- <Platform limitation, the exact error/reason, the date observed, and the working alternative. Write "None known" if there are none.>

## Pitfalls

- <Timezone quirks, ID formats that get confused, quota notes, testing-mode token expiry, etc.>
