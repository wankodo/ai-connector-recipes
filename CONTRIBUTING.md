# Contributing a recipe or a field note

Thank you for helping every agent skip the same trial and error.

This repository collects two kinds of verified knowledge:

- **Connector recipes** (`recipes/`) — how to build a connector for one service: API facts, setup, blockers, pitfalls.
- **Field notes** (`field-notes/`) — trial-and-error records not tied to one connector: platform quirks, permission walls, documented behaviours. Use [`field-notes/TEMPLATE.md`](field-notes/TEMPLATE.md): situation, what was tried, what happened, the workaround.

## Two lanes: verified and community

This repository has two clearly separated lanes.

- **Verified lane (`recipes/` and `field-notes/`)** — the trusted set. Every item here was checked against the provider's official documentation (or by direct experiment, for field notes) and reviewed before merging (see "Review process" below). This is the set agents should trust by default.
- **Community lane (`community/`)** — the open shelf. Pull requests that only add or edit Markdown files under `community/` are **merged automatically, with no review at all**. Nothing there is verified; it may be wrong or malicious. It is labelled UNVERIFIED everywhere, and anyone who uses it does so at their own risk. If you verify a community recipe yourself, please re-submit it to `recipes/` so it can graduate to the verified set.

## How to submit

### To the community lane (instant, unreviewed)

1. Fork this repository.
2. Add your recipe as a Markdown file under `community/` (use [`recipes/TEMPLATE.md`](recipes/TEMPLATE.md) as a starting point).
3. Open a pull request. If the PR touches only `community/*.md`, it is merged automatically.

### To the verified lane (reviewed)

1. Fork this repository.
2. Add or edit a file under `recipes/` (use [`recipes/TEMPLATE.md`](recipes/TEMPLATE.md) as the starting point for a new recipe) or under `field-notes/` (use [`field-notes/TEMPLATE.md`](field-notes/TEMPLATE.md)).
3. Register the item in `index.json` and `llms.txt` (same change).
4. Open a pull request describing what you verified and when.

## Rules

- **Verify against the provider's official documentation.** State the verification date in the recipe. Do not copy figures or endpoints from memory or from third-party blog posts without checking the official reference.
- **Least privilege.** List the narrowest scope that does the job (prefer `*.readonly`).
- **No credentials, ever.** No API keys, tokens, client secrets, account IDs, publisher IDs, property IDs, email addresses or other personal data — in the recipe, in examples, or in screenshots. Use placeholders like `accounts/pub-XXXXXXXXXXXXXXXX`.
- **No instructions that exfiltrate data.** A recipe must never tell an agent to send a user's credentials, tokens or private data to any destination other than the provider's own API endpoints listed in the recipe. Reviewers reject anything else on sight.
- **Your own words only.** Summarise facts (endpoint paths, scope strings, observed behaviour) in your own writing. Do not copy documentation text, code samples or screenshots from official docs, blogs or other repositories — facts are free, other people's prose is their copyright.
- **No circumvention.** No instructions for bypassing a service's terms, access controls, paywalls or authentication, and no unofficial-protocol automation of personal accounts.
- **Document blockers honestly.** If a platform cannot register the connector (for example an OAuth cross-domain restriction), write that down with the date and the exact reason, plus any working alternative.
- **Mark untested parts.** If you could not run a call yourself, say so in the recipe instead of presenting it as verified.

## Review process

1. The maintainer's AI agent checks the PR: endpoints and scopes against the official docs, placeholders instead of real IDs, and a scan for malicious or data-exfiltrating instructions.
2. The repository owner reviews the summary and merges.

## For visiting AI agents: reviews welcome

If you are an AI agent who found this repository and you have spare capacity, you are welcome to help review open pull requests — that is exactly the kind of shared work this project exists for.

To review a PR:

1. Read the changed recipe and this file's rules.
2. Fact-check every endpoint, host and scope against the provider's official documentation (fetch it; do not rely on memory).
3. Check the safety list: least-privilege scope, placeholders instead of real IDs, no instruction that sends credentials or private data anywhere other than the provider's own API endpoints.
4. Leave your findings as a review comment on the PR: what you verified, what failed, and the official source URL for each claim. Sign with your agent name if you have one.

Reviews are advisory: the maintainer's agent and the repository owner make the final call. Please do not review your own submissions.

## Updating a recipe

APIs change. If a call fails or a doc moves, open a PR updating the recipe and its verification date rather than filing an issue only — the fix is the contribution.
