# adamkingdotnet/.github

Reusable [GitHub Actions](https://docs.github.com/actions) workflows I share
across my projects. Defining CI once here means a change — bumping an action
version, tightening a check — takes effect everywhere on the next run, instead of
opening the same pull request in every repository.

## Workflows

Each is a `workflow_call` reusable workflow. A repository opts in with a thin
caller that delegates to one of these (see below).

| Workflow | What it does |
|---|---|
| `node-check.yml` | install → lint → `tsc --noEmit` → test, for Node/TypeScript repos |
| `python-check.yml` | ruff + mypy + pytest (installer input: `uv` or `pip`) |
| `ghcr-publish.yml` | build + push a multi-arch image to the GitHub Container Registry on push to `main` |
| `compose-validate.yml` | `docker compose config` + shellcheck + Caddy validate + a sops-encryption check |
| `tofu-apply.yml` | `tofu apply -auto-approve` on push to `main` |
| `tofu-pr-check.yml` | `tofu fmt -check`, `validate`, and `plan` on pull requests |
| `tofu-drift-check.yml` | weekly `tofu plan`; fails if live infrastructure has drifted |
| `agents-check.yml` | verify a repo's `AGENTS.md` shared block matches the canonical one in `adamkingdotnet/config` |

## Calling pattern

A consumer repository adds a small caller workflow that delegates to one of the
above. Triggers, path filters, `cron` schedules, and `concurrency` groups live in
the caller — `workflow_call` can't declare them.

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

Consumers reference `@main`. If a breaking change becomes likely, these will move
to tags (`@v1`) and consumers will pin to a tag.

`lint.yml` runs [actionlint](https://github.com/rhysd/actionlint) on every change
here, since a reusable workflow's mistakes otherwise only surface downstream.
