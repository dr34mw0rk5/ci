# dr34mw0rk5/ci — shared CI/CD standard

Reusable GitHub Actions workflows and composite actions for our product monorepos: Bun or pnpm +
Turborepo, Drizzle/Postgres, Docker images to GHCR, Coolify or Vercel as the deploy target.

> **Public on purpose.** Reusable workflows in a *private* repository can only be called by
> repositories in the same organization, and our repos are spread across several orgs. This
> repository therefore contains pipeline logic only — no secrets, no hostnames, no product code.
> Secrets stay in each caller repo or its GitHub Environment and reach these workflows via
> `secrets: inherit`. Keep it that way: nothing repo-specific belongs here.

## Contents

| Path | Kind | Purpose |
|---|---|---|
| `.github/workflows/ci-bun.yml` | reusable | typecheck / test / audit matrix for Bun repos |
| `.github/workflows/ci-pnpm.yml` | reusable | the same, for pnpm + Node repos |
| `.github/workflows/pr-policy.yml` | reusable | whitespace, tracked `.env`, obfuscated payloads, pinned toolchain |
| `.github/workflows/semgrep.yml` | reusable | SAST scan |
| `.github/workflows/db-migrations.yml` | reusable | journal invariants + replay every migration on an empty Postgres |
| `.github/workflows/release-docker.yml` | reusable | gate on CI → build/push to GHCR → Coolify deploy → verify |
| `.github/workflows/verify-release.yml` | reusable | poll public `/version` endpoints until they serve a SHA |
| `actions/setup-bun` | composite | Bun pinned from `package.json#packageManager` + caches + install |
| `actions/setup-pnpm` | composite | pnpm from `packageManager`, Node from `.nvmrc`, frozen install |
| `actions/wait-for-ci` | composite | refuse to release a commit whose required checks are not green |
| `actions/deploy-coolify` | composite | webhook → deployment-status poll → release verification → smoke |
| `templates/` | copy-paste | thin caller workflows, Dependabot and Renovate config |

## Design rules

1. **One source of truth per fact.** The runtime version lives in `package.json#packageManager` or
   `.nvmrc`. Nothing restates it — no hardcoded versions in YAML, no globally installed tools.
2. **Everything that gates a merge runs on PR; everything that gates a release runs before the image
   is pushed.**
3. **A deploy is not done when the webhook returns 200.** It is done when the SHA is observably live.
4. **Fail closed on the release path, fail open on advisory scanners.**
5. **Pin everything**: actions by SHA, CLIs by exact version, base images by digest, deps by lockfile.
6. **Never interpolate `${{ }}` into a `run:` body.** Route through `env:` — `workflow_dispatch`
   inputs are attacker-controlled.

## Using it

Pin to a tag. Dependabot's `github-actions` ecosystem updates reusable-workflow `uses:` refs, so
pinned callers stay current automatically.

```yaml
# .github/workflows/ci.yml in a product repo
name: CI

on:
  pull_request:
  merge_group:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

permissions:
  contents: read

jobs:
  ci:
    uses: dr34mw0rk5/ci/.github/workflows/ci-bun.yml@v1.0.0
    with:
      tasks: >-
        [
          {"name": "typecheck", "run": "bun run typecheck --cache-dir=.turbo --concurrency=4"},
          {"name": "test",      "run": "bun run test --cache-dir=.turbo"},
          {"name": "audit",     "run": "bun audit --audit-level=high"}
        ]
```

Every entry in `tasks` becomes its own job, and therefore its own required status check. See
`templates/` for full callers.

### Check-run names are an API

Branch protection, `wait-for-ci` and the merge queue all match on job names. Renaming a job silently
disables the gate that referenced it. Under a reusable workflow the name becomes
`<caller-job-id> / <job-name>` — e.g. `ci / typecheck`. `wait-for-ci` matches by substring, so
`typecheck` keeps matching after a migration; branch-protection entries must be updated by hand.

## Rolling it out

1. Tag a release here (`v1.0.0`). Move `v1` with each compatible change; cut `v2.0.0` for breaking
   input changes.
2. In each product repo, replace the self-contained workflow with the matching `templates/` caller.
3. Create a `.github` repository in each org with a `workflow-templates/` directory holding those
   callers, so new repos start from the standard.
4. Org ruleset on `~DEFAULT_BRANCH`: require a pull request plus the status checks `ci / typecheck`,
   `ci / test`, `lint`, `hygiene`, `toolchain`, `semgrep`, `migration-journal`, `migrate`, and
   require the merge queue. Rulesets cover private repos on Team plans; the stricter *required
   workflows* rule needs Enterprise Cloud.
5. Add `templates/dependabot.yml` (or `templates/renovate.json`) to every repo, including the
   `github-actions` ecosystem — that is what keeps both the action pins and the `@vX.Y.Z` refs above
   moving.
6. Create `production` and `staging` GitHub Environments per repo, with required reviewers on
   production, and put the deploy secrets there.

Organization secrets do not cross org boundaries. Set the bootstrap credentials
(`COOLIFY_*`, `INFISICAL_*`, `TRIGGER_ACCESS_TOKEN`, registry tokens) as **organization** secrets
scoped to the relevant repos, so rotation is one edit per org.

## Versioning

- `v1` — moving tag, latest compatible release.
- `v1.2.3` — immutable. Preferred, because Dependabot then raises a visible PR per change.

Breaking changes: removing an input, changing a default in a way that changes behaviour, or renaming
a job (which renames the status check).
