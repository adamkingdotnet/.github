# adamkingdotnet-.github

<!-- BEGIN working-agreement (vendored from adamkingdotnet/config — edit there, then re-vendor) -->
## Working agreement

These four tenets are non-negotiable:

1. Ask, don't assume. If something is unclear, ask before writing a single line. Never make silent assumptions about intent, architecture, or requirements.
2. Simplest solution first. Always implement the simplest thing that could work. Do not add abstractions or flexibility that weren't explicitly requested.
3. Don't touch unrelated code. If a file or function is not directly part of the current task, do not modify it, even if you think it could be improved.
4. Flag uncertainty explicitly. If you are not confident about an approach or technical detail, say so before proceeding. Confidence without certainty causes more damage than admitting a gap.

Operating instructions:

- Keep remote CI/CD green — after pushing **and** after merging. A change isn't done until checks pass on the merged result. (See **Applies here** below for what runs where — some repos gate on PRs only, some have no CI yet.)
- Reach infrastructure directly over SSH (`ssh nas`, `ssh vps`, …) for logs, inspection, and deploys rather than asking for output.
- Drive providers with their own tooling (e.g. `wrangler` for Cloudflare) rather than asking to click through a dashboard.
- Verify implementation specifics against the **latest** upstream docs — don't trust model-ingrained versions or APIs that may be stale; check the live docs first.
<!-- END working-agreement -->
Org `.github` repo: canonical **reusable GitHub Actions workflows** (`workflow_call`) the whole fleet consumes, plus centralized, Dependabot-pinned action versions. See [README.md](README.md) for the calling pattern. There is no build/test/app here — "deploy" = merging to `main`.

## THE GOTCHA (read first)

Every consumer references these workflows **`@main`**. Any change you merge to `.github/workflows/*.yml` takes effect **FLEET-WIDE on the next downstream run** — no pin, no staging, no rollout. Treat every edit as a fleet-wide change. Downstream repos: `adamking.net`, `king-consulting`, `deervalleytexas.com`, `cf-data-workers`, `netchaff`, `nas-docker`, `vps-docker`, `personal-data`.

## Workflow → consumer map

Verified against the actual `uses:` in each consumer's caller — trust this over the workflows' own header comments (some are stale).

| Reusable workflow | Consumers |
|---|---|
| `node-check.yml` | adamking.net, king-consulting, cf-data-workers |
| `python-check.yml` | cf-data-workers (python/mls-model, uv), netchaff (pip), personal-data (uv) |
| `ghcr-publish.yml` | netchaff |
| `compose-validate.yml` | nas-docker, vps-docker |
| `tofu-apply.yml` | adamking.net, king-consulting, deervalleytexas.com, cf-data-workers |
| `tofu-pr-check.yml` / `tofu-drift-check.yml` | adamking.net, king-consulting, deervalleytexas.com |
| `lint.yml` | **this repo's own self-gate** (not reusable) |

## Hard conventions

- **Path filters, `cron`, `concurrency` MUST live in the CALLER** — `workflow_call` triggers can't declare them.
- **No injection.** Pass shell as an input, expose via `env: CMD:`, run `bash -c "$CMD"`. Do NOT inline-interpolate `${{ inputs.* }}` into a `run:` block.
- **Action pins are Dependabot-managed** (`.github/dependabot.yml`, grouped weekly) — don't hand-bump.

## Local verification

`lint.yml` runs **two** self-gate jobs: **actionlint** (`docker://rhysd/actionlint`, which also shellchecks `run:` blocks) and **agreement** (the inlined working-agreement check — it verifies the vendored `BEGIN/END working-agreement` block in `AGENTS.md` still matches config's canonical via `check-agreement.sh`). Run actionlint locally before pushing:

```
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12 -color
```

For any load-bearing change, also sanity-check a real consumer's caller (the exact `uses:` path + inputs it passes) before merging — actionlint can't see across the `@main` boundary.

### Applies here

- **CI-green is two-sided:** this repo's own gates passing (actionlint + the working-agreement `agreement` job in `lint.yml`) do NOT mean the fleet is green. A merged change only proves out when a downstream consumer runs against `@main`. Verify a real caller.
- **verify-latest, not verify-current:** action pin versions are the moving part; check they're current (Dependabot owns this).
- **ssh:** secondary — relevant only because `compose-validate.yml` serves the host repos (nas-docker, vps-docker).
- **wrangler:** N/A — no Workers deployed from this repo.

## Shared agent layer

This repo consumes the **`king-agents`** plugin from `adamkingdotnet/config` (auto-enabled via the `extraKnownMarketplaces` + `enabledPlugins` block in the committed `.claude/settings.json`). Permissions live in that same file, **byte-gated to config's `meta` template** — don't hand-edit it; changes belong in the config template and re-vendor. Only genuine machine-local grants go in `.claude/settings.json`'s gitignored sibling `.claude/settings.local.json`. A repo-level `!.claude/settings.json` negation in `.gitignore` keeps the committed harness file tracked despite the global `.claude/*` ignore.

This is a **non-CI-bearing / advisory** repo: there is no `.claude/king.json`, so the plugin's verify-before-done `Stop` hook has no gate command to run here (nothing to build/test — "deploy" = merging to `main`). The template drift itself is CI-gated by `.github/workflows/self-settings-check.yml`, which calls the reusable `settings-check.yml@main` with `type: meta` on any PR/push touching `.claude/settings.json`. Run `/king:doctor` for a health check (plugin version, agreement/settings drift).
