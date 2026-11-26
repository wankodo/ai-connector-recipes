# On Muse, a token-based custom GitHub connector writes without per-write approval cards

Topic: GitHub on Muse (official connector vs a self-built token connector)
Verified on 2026-11-27 by direct experiment.

## Situation

Every write through Muse's official GitHub connector surfaces a per-operation approval card. For a workflow with frequent file pushes across several site repositories, the maintainer wanted to know whether a custom connector built on a personal access token would reduce that approval frequency.

## What was tried

1. The user issued a fine-grained personal access token in GitHub (Settings → Developer settings → Personal access tokens → Fine-grained tokens), scoped to 7 selected repositories with Contents read and write and Metadata read-only.
2. The token was registered through Muse's secure API access flow as an API key credential (host `api.github.com`, placement: bearer header). The agent never saw the token value.
3. A small helper was built on the platform's surrogate credential mechanism, calling the REST API directly.
4. Verified in order: `GET /user`, `GET /user/repos`, then a Contents API write test in a low-impact repository — create a test file with `PUT`, delete it with `DELETE`, and confirm `GET` on the path returns 404.

## What happened (the result)

- The credential asked for approval once, on its first use.
- After that, reads and writes ran without approval cards. The write test succeeded end to end.
- This is a different lane from the official connector, not a bypass of a declined approval: the user chose to issue their own token for their own agent. Nothing about the official connector's approval gating changed.

## Workaround

That is the workaround: when the user is willing to issue a token, a token-based custom connector is a working lane with far fewer approval prompts. The build steps are summarised in the GitHub recipe (`recipes/github.md`).

## Notes

- The premise is the user's consent to issue a key. A user who does not want to issue tokens cannot use this lane, and the official connector remains the route.
- Issuing the token from a phone browser hit a wall: GitHub's final confirmation dialog button did not respond in either of two mobile browsers tested. A desktop browser is the reliable route for the issuance step.
- In the permission picker, the search box searches by permission name (for example "Contents"). "Read and write" is a level you choose after adding the permission, not a search term.
- A create-then-delete write test still leaves two real commits in history. On a public repository anyone can see them, so keep test file contents harmless and free of any private data.
- Multi-file pushes through the Git Data API (blobs, trees, commits) and binary files are unverified on this lane so far. The official connector and the web UI remain the routes for those.
