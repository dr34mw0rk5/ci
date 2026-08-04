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

## Runner cost, and the correctness that comes with it

These are not micro-optimisations bolted on afterwards — the cheap shape is also
the correct one. One repo's July bill made the case: 30,427 runner-minutes over
7,754 workflow runs, plus 135 GB-months of sticky disk.

**Release on `workflow_run`, never on `push`.** A push-triggered release starts
in parallel with CI and then busy-waits for CI's check runs to appear — roughly
13 billed idle minutes per merge, ~5,500 minutes/month at that repo's merge
rate. Under `workflow_run` the checks are already decided, so the gate is a
single API read. It is also *more* correct: a cancelled CI run (superseded by a
faster follow-up merge) skips the release entirely, which debounces the
deploy chains that rapid successive merges produce.

Two traps come with it, both handled by `release-docker.yml`:

- `workflow_run` runs the workflow file from the **default branch** and checks
  out the default branch HEAD. Every checkout, image tag and rollout comparison
  must use `github.event.workflow_run.head_sha`. Never `github.sha`.
- `workflow_run` has **no `paths:` filter**. The `relevant-paths` input replaces
  it by diffing against the commit production actually serves, read from
  `/version`. That is strictly better than a path filter: it self-heals, because
  after a failed rollout the next merge sees the cumulative diff. Any doubt —
  unreachable `/version`, unfetchable live commit, manual dispatch — deploys.

**A green release run no longer means a deploy happened.** Anything chained
after it (notification sync, cache warm) must check that the build job actually
ran, or it will wait for a revision that never appears:

```yaml
conclusion=$(gh api "repos/$GITHUB_REPOSITORY/actions/runs/${{ github.event.workflow_run.id }}/jobs?per_page=50" \
  --jq '.jobs[] | select(.name | startswith("build")) | .conclusion' | head -1)
[ "$conclusion" = "success" ] || { echo "No image was built — nothing to sync."; exit 0; }
```

**Prune the build cache.** On a persistent builder the buildkit cache grows
without bound — that is where the 135 GB-months came from.
`prune-build-cache` (on by default) drops layers older than 7 days after each
build, best-effort so a prune failure cannot fail a pushed image.

**Advisory scanners run on PRs only.** Re-running Semgrep and similar on the
push to main duplicates the scan the PR already passed — ~4,000 redundant runs
a month on that repo. Keep a weekly cron so newly published registry rules
still land.

**Right-size the runner.** `runner` builds images; `gate-runner` covers the
gate, deploy and verify jobs, which are API-poll and I/O bound and belong on
the smallest tier. Short checks (naming, audit, journal) belong there too —
about half the per-minute price for jobs that finish in under 30 seconds.

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

### The org ruleset

Gates that are not *required* do not gate. Each org carries the same pair of
rulesets, targeting `~DEFAULT_BRANCH` on that org's product repos:

| Ruleset | Requires | Applies to |
|---|---|---|
| `ci-standard` | `typecheck`, `test`, `lint`, `hygiene`, `toolchain`, `Semgrep scan` | every product repo |
| `ci-standard-db` | `migration-journal`, `migrate` | repos with a migration journal |

Plus, in both: a pull request (zero approvals — the point is to make the checks
run, not to add review ceremony), no force-push, no branch deletion, and an
`OrganizationAdmin` bypass so a permanently-red check cannot lock the repo with
no way out.

Two things make this workable:

**Check names are uniform.** Every repo names the same job the same way — that
is why a single ruleset can cover four organizations. A repo whose lint job was
called `Type-aware changed files` had to be renamed to `lint` to fit. Renaming a
job silently removes it from the ruleset, so treat the names as an API.

**Only require checks that always report.** A required check that a path filter
can skip blocks the PR forever with *"Expected — Waiting for status to be
reported"*. That is why `db-migrations.yml` carries no `paths:` filter, and why
path-filtered or advisory jobs (coverage floors, prose lint, `continue-on-error`
scanners) are deliberately **not** in the required set.

**Create the ruleset disabled, activate after the workflows are on the default
branch.** An active ruleset requiring checks that do not exist yet blocks the
very pull requests that introduce them.

```bash
gh api --method PUT "orgs/$ORG/rulesets/$ID" -f enforcement=active
```

Merge queues are not enabled: `pr-policy` diffs against the merge base, which
does not exist for `merge_group` events, so requiring it and enabling the queue
at the same time would deadlock. Enable the queue only for the checks that run
on `merge_group`.

Organization secrets do not cross org boundaries. Set the bootstrap credentials
(`COOLIFY_*`, `INFISICAL_*`, `TRIGGER_ACCESS_TOKEN`, registry tokens) as **organization** secrets
scoped to the relevant repos, so rotation is one edit per org.

## Traps found by the first real runs

Every one of these blocked a pull request the first time the standard ran for real.

**A required check must be able to report on every PR.** Not just "usually run".
A `branches:` or `paths:` filter on a required workflow means PRs outside that
filter sit at *"Expected — Waiting for status to be reported"* forever. `ci-*`,
`pr-policy` and `db-migrations` therefore carry no such filters, and jobs that
legitimately need them (coverage floors, prose lint) stay out of the required set.

**Do not pass a flag the package script already sets.** `bun run typecheck
--concurrency=4` where the script is already `turbo typecheck --concurrency=4`
fails with *"the argument '--concurrency' cannot be used multiple times"*.

**`eval` needs a tight pattern.** `\beval[[:space:]]*\(` matches English prose
inside template literals — *"…detection eval (per-kind precision/recall)"*.
Require no space before the paren.

**Dependabot needs a `cooldown`.** Semgrep's `dependabot-missing-cooldown` is a
real finding: without a quarantine window a compromised release can be
auto-proposed minutes after publication. See `templates/dependabot.yml`.

**`turbo run <task>` errors when the task is undefined.** A `test` job in a repo
whose `turbo.json` has no `test` task fails outright rather than no-opping.
Define the task; packages without a `test` script are then simply skipped.

**Repo-wide lint and format are red almost everywhere.** Gate changed files
instead, or the gate is red on day one and gets ignored.

**Semgrep audits your CI config, not just your code.** Two of its supply-chain
rules fire on this standard's own files: `dependabot-missing-cooldown` and
`renovate-missing-minimum-release-age`. Both are correct — an update config with
no quarantine window will happily propose a release published minutes ago. Note
that a top-level `minimumReleaseAge` does not satisfy the Renovate rule: every
`packageRule` must carry it, and a `vulnerabilityAlerts` override that shortens
it re-triggers the finding.

**Do not assume a CLI flag exists everywhere.** `oxlint
--no-error-on-unmatched-pattern` is accepted by 1.68 and rejected outright by
1.53. Pinning the toolchain per repo means the same shared snippet can fail in
one of them.

**A conflicted PR runs no workflows at all.** GitHub cannot build the merge
commit, so `pull_request` events never fire — the checks are not red, they are
absent, and a required-checks ruleset then blocks the PR forever. If a PR's
checks vanish, look at mergeability before looking at the workflows.

**Portable scripts must respect the strictest repo's lint rules.** A shared
script using non-null assertions fails in any repo that bans them.

## Versioning

- `v1` — moving tag, latest compatible release.
- `v1.2.3` — immutable. Preferred, because Dependabot then raises a visible PR per change.

Breaking changes: removing an input, changing a default in a way that changes behaviour, or renaming
a job (which renames the status check).
