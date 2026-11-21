# GitHub App integrations cannot create repositories or write workflow files

Topic: GitHub (work via a GitHub App integration, e.g. an AI platform's GitHub connector)
Verified on 2026-11-21 by direct experiment.

## Situation

Setting up a brand-new repository and later adding a GitHub Actions workflow, entirely through a GitHub App integration instead of the web UI.

## What was tried

1. Creating a repository through the integration's create-repository call.
2. Pushing a batch of files that included `.github/workflows/auto-merge-community.yml` through the integration's file-push call.

## What happened (the wall)

1. Repository creation failed: `403 Resource not accessible by integration`.
2. The push failed at tree creation with the same `403 Resource not accessible by integration` — only when the batch contained a file under `.github/workflows/`. The same files without the workflow pushed fine.

## Workaround

Do those two steps as the user in the signed-in web UI (a human, or the agent's browser session): create the repository there, and add workflow files there. Everything else — pushing and deleting ordinary content files — works through the integration.

## Notes

- This is an installation-permission boundary, not a bug: app installations are typically not granted repository creation, and writing workflow files requires the "Workflows" permission.
- Expect the same shape of wall on other platforms: an integration's permission set is narrower than the user's own account, and the missing pieces are usually creation and automation-definition operations.
