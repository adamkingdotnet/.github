# adamkingdotnet/.github

Org-level repo holding **reusable GitHub Actions workflows** that the rest of `adamkingdotnet/*` consume.

The point: change the canonical CI shape in one place (here) and every consumer repo picks it up automatically on its next workflow run. No more N-PR fanout to bump `actions/checkout` v6 → v7 across the fleet.

## Workflows

Consumer lists below are verified against each repo's actual `uses:` — trust them over any stale header comment.

| Workflow | Used by | What it does |
|---|---|---|
| `node-check.yml` | adamking.net, king-consulting, cf-data-workers | `npm/pnpm` install → lint → `tsc --noEmit` → test |
| `python-check.yml` | cf-data-workers (python/mls-model, uv), netchaff (pip), personal-data (uv) | ruff + mypy + pytest (installer input: uv \| pip) |
| `ghcr-publish.yml` | netchaff | Build + push a multi-arch image to GHCR on push to main |
| `compose-validate.yml` | nas-docker, vps-docker | `docker compose config` + shellcheck + caddy validate + sops check |
| `tofu-apply.yml` | adamking.net, king-consulting, deervalleytexas.com, cf-data-workers | `tofu apply -auto-approve` on push to main |
| `tofu-pr-check.yml` | adamking.net, king-consulting, deervalleytexas.com | `tofu fmt -check`, `tofu validate`, `tofu plan` on PRs |
| `tofu-drift-check.yml` | adamking.net, king-consulting, deervalleytexas.com | Weekly `tofu plan -detailed-exitcode`; fails on drift |
| `worker-pr-check.yml` | **none** (orphaned — mls/pulse folded into the cf-data-workers monorepo, which uses `node-check.yml`) | `npm ci` → optional pre-build → `tsc --noEmit` → `npm test` |
| `agents-check.yml` | fleet (via each repo's `agents.yml`) | Assert AGENTS.md's working-agreement block matches `config` canonical |

## Calling pattern

Each consumer repo has a thin caller workflow that delegates to one of these. Path filters, cron schedules, and concurrency groups stay in the caller (GitHub Actions doesn't allow them in `workflow_call` triggers).

Example caller (`adamking.net/.github/workflows/infra.yml`):

```yaml
name: Infra apply
on:
  push:
    branches: [main]
  workflow_dispatch:
concurrency:
  group: infra-${{ github.repository }}
  cancel-in-progress: false

jobs:
  apply:
    uses: adamkingdotnet/.github/.github/workflows/tofu-apply.yml@main
    secrets: inherit
```

## Versioning

Consumers reference `@main` for now. If/when a breaking change becomes likely, switch to tags (`@v1`) and pin consumers to a tag.
