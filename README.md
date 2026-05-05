# adamkingdotnet/.github

Org-level repo holding **reusable GitHub Actions workflows** that the rest of `adamkingdotnet/*` consume.

The point: change the canonical CI shape in one place (here) and every consumer repo picks it up automatically on its next workflow run. No more N-PR fanout to bump `actions/checkout` v6 → v7 across the fleet.

## Workflows

| Workflow | Used by | What it does |
|---|---|---|
| `tofu-apply.yml` | adamking.net, king-consulting, deervalleytexas.com | `tofu apply -auto-approve` on push to main |
| `tofu-pr-check.yml` | (same) | `tofu fmt -check`, `tofu validate`, `tofu plan` on PRs |
| `tofu-drift-check.yml` | (same) | Weekly `tofu plan -detailed-exitcode`; fails on drift |
| `worker-pr-check.yml` | mls, pulse | `npm install` → optional pre-build → `tsc --noEmit` → `npm test` |

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
