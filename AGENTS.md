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
| `worker-pr-check.yml` | **none — no caller in the fleet.** mls/pulse were folded into the cf-data-workers monorepo (`apps/mls-worker`, `apps/pulse-worker`), which uses `node-check.yml`. Confirm a real caller before assuming this is live. |
| `lint.yml` | **this repo's own self-gate** (not reusable) |

## Hard conventions

- **Path filters, `cron`, `concurrency` MUST live in the CALLER** — `workflow_call` triggers can't declare them.
- **No injection.** Pass shell as an input, expose via `env: CMD:`, run `bash -c "$CMD"`. Do NOT inline-interpolate `${{ inputs.* }}` into a `run:` block.
- **Action pins are Dependabot-managed** (`.github/dependabot.yml`, grouped weekly) — don't hand-bump.

## Local verification

The only self-gate is `lint.yml` → **actionlint** (`docker://rhysd/actionlint`), which also shellchecks `run:` blocks. Run it locally before pushing:

```
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12 -color
```

For any load-bearing change, also sanity-check a real consumer's caller (the exact `uses:` path + inputs it passes) before merging — actionlint can't see across the `@main` boundary.

### Applies here

- **CI-green is two-sided:** this repo's actionlint passing does NOT mean the fleet is green. A merged change only proves out when a downstream consumer runs against `@main`. Verify a real caller.
- **verify-latest, not verify-current:** action pin versions are the moving part; check they're current (Dependabot owns this).
- **ssh:** secondary — relevant only because `compose-validate.yml` serves the host repos (nas-docker, vps-docker).
- **wrangler:** N/A — no Workers deployed from this repo.
