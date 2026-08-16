# adamkingdotnet-.github

Organization-level shared workflows and templates for the `adamkingdotnet` GitHub org. Primary repo for CI/CD reusable workflows consumed by all fleet repos.

## Structure

- **`.github/workflows/`** — Reusable workflows (called via `workflow_call`) and self-linting CI.
- **`README.md`** — Workflow reference table with calling patterns.

## Architecture

- Workflows are `workflow_call`-only (no direct `on:` triggers except `lint.yml` which validates the workflows themselves).
- Consumers call via: `uses: adamkingdotnet/.github/.github/workflows/<name>.yml@<sha>` (SHA-pinned with a `# main` comment; dependabot updates the pins) or `@main`.

## Key Files

- `README.md` — Workflow reference table with calling patterns. **Read this first.**
- `.github/workflows/` — Reusable GitHub Actions workflows (called via `workflow_call`).

## Skip These

## Notes

<!-- Quick-add scratchpad below -->