# trichonaut-ci-workflows

Shared [reusable GitHub Actions workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
for the trichonaut project family: security scanning (`audit-ci` / Semgrep
/ gitleaks) and, for the two plain-Astro sites, a shared quality job
(lint + accessibility scan). Public on purpose: GitHub does not allow a
private repository's reusable workflow to be called from another private
repository unless both are owned by the same organization — these repos
are all under one personal account, so a public host is the only way to
share this without making an existing project repo public. Neither
workflow carries account-specific secrets, ARNs, or credentials; every
project-specific detail is passed in as an input by the caller.

## Security scan

In a caller repo's `.github/workflows/security.yml` (or as a job inside an
existing workflow):

```yaml
name: Security

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  security:
    permissions:
      contents: read
      pull-requests: read
    uses: JustFourShiggles/trichonaut-ci-workflows/.github/workflows/security-scan.yml@main
    with:
      audit_dirs: "." # optional, defaults to "."
      audit_allowlist: "" # optional, space-separated GHSA IDs
      semgrep_exclude: "" # optional, space-separated --exclude patterns
      cache_dependency_path: "" # optional, only needed if there's no root-level lockfile
```

**The caller's `permissions:` block is required, not optional** — a
reusable workflow's own job-level permissions can only be *reduced* by
its caller, never elevated beyond what the caller explicitly grants.
Since most repos default to minimal implicit permissions when a caller
job doesn't declare `permissions:` at all, omitting this block makes
GitHub reject the whole call before it runs a single step
(`conclusion: startup_failure`, zero jobs, no useful log — confirmed by
testing). Match this workflow's own `contents: read` / `pull-requests:
read` exactly; granting less will fail the same way, granting more is
simply ignored (permissions can't be elevated up the chain either).

**Don't add `secrets: inherit`** — `GITHUB_TOKEN` is automatically
available to a called reusable workflow's own jobs with no explicit
passing needed (gitleaks uses it to comment on PRs). `secrets: inherit`
is only for *custom* secrets this workflow doesn't need, and Semgrep's
own security-audit ruleset flags it as an unnecessary least-privilege
violation if you add it anyway (confirmed by testing).

### Inputs

| Input | Default | Purpose |
|---|---|---|
| `audit_dirs` | `.` | Space-separated directories to run `audit-ci` against, one per `package.json`/`package-lock.json` pair. |
| `audit_allowlist` | `""` | Space-separated GHSA IDs to allowlist in `audit-ci` — for advisories with no upstream fix available. Applied identically to every directory in `audit_dirs`. |
| `semgrep_exclude` | `""` | Space-separated additional `--exclude` patterns for Semgrep (e.g. `terraform` when that's covered by a separate tool like Prowler instead). |
| `cache_dependency_path` | `package-lock.json` | Passed to `actions/setup-node`'s `cache-dependency-path`. `setup-node` only looks for a lockfile at the repo root by default (not recursively) — override this when the repo's only lockfile lives elsewhere. |

### Current callers and their config

| Repo | `audit_dirs` | `audit_allowlist` | `semgrep_exclude` | `cache_dependency_path` |
|---|---|---|---|---|
| trichonaut | `.` | `GHSA-jmr9-qjv8-65gv GHSA-7pqw-9j4j-h8q3` | — | — (default) |
| trichonaut-catalog | `.` | `GHSA-jmr9-qjv8-65gv GHSA-7pqw-9j4j-h8q3` | — | — (default) |
| trichonaut-manage | `. web lambdas/api lambdas/publish lambdas/image-processor` | `GHSA-jmr9-qjv8-65gv GHSA-7pqw-9j4j-h8q3` | — | — (default) |
| trichonaut-e2e | `.` | — | — | — (default) |
| trichonaut-infra | `scrapers/trichonaut-catalog` | — | `terraform` | `scrapers/trichonaut-catalog/package-lock.json` |

The `GHSA-jmr9-qjv8-65gv`/`GHSA-7pqw-9j4j-h8q3` allowlist covers two
high-severity `extract-zip` advisories (reached via `pa11y-ci` ->
`puppeteer`) with no patched version available upstream at all — dev/CI-only
tooling, tracked via each repo's own Dependabot alerts, not blocking on a
fix that doesn't exist.

## Astro quality

In a caller repo's `.github/workflows/ci.yml`:

```yaml
jobs:
  quality:
    permissions:
      contents: read
    uses: JustFourShiggles/trichonaut-ci-workflows/.github/workflows/astro-quality.yml@main
```

Same permission-block requirement as the security scan above. No inputs:
this only exists because trichonaut and trichonaut-catalog's `quality`
jobs were byte-for-byte identical (`npm ci` / `npm run lint` / `npm run
test:a11y`, same script names in both `package.json`s) — trichonaut-manage's
own `quality` job is genuinely different (multiple package directories,
a web SPA build, a preview-sync check) and was deliberately left as its
own inline job rather than forced into this shape.

### Current callers

`trichonaut`, `trichonaut-catalog`.

## Dependabot auto-merge

In a caller repo's `.github/workflows/dependabot-auto-merge.yml`:

```yaml
name: Dependabot auto-merge

on:
  workflow_run:
    workflows: ["CI", "Security"] # must match gate_workflow_names below, and each workflow's own `name:`
    types: [completed]

permissions:
  contents: write
  pull-requests: write
  actions: read

jobs:
  dependabot:
    uses: JustFourShiggles/trichonaut-ci-workflows/.github/workflows/dependabot-auto-merge.yml@main
    with:
      gate_workflow_names: "CI,Security"
```

Merges a Dependabot PR the moment every workflow named in
`gate_workflow_names` has its own successful run against that PR's exact
head commit -- never GitHub's native "auto-merge" queue feature, since
that needs required-status-check branch protection to guarantee it waits
for checks, and none of these repos can enable branch protection at all
(private repos on a plan tier where the branch-protection API 403s).
Instead this re-checks every gate workflow's conclusion itself via `gh run
list`, re-triggered on each one's own completion, and only merges once
they've *all* succeeded for that same commit. Only ever merges
patch/minor updates (reads Dependabot's own `update-type:` commit
trailer) -- a major-version bump, or a grouped update where any single
dependency in it is major, is left alone for manual review. Same
`gh pr merge --squash --delete-branch` this account already uses in
`trichonaut/auto-merge-listings.yml`.

**The caller's `permissions:` block is required**, same reasoning as the
security scan above -- this one needs `contents: write` (to merge/delete
the branch), `pull-requests: write` (to merge), and `actions: read` (the
gate-workflow check's own `gh run list --workflow` calls), not the
security scan's `read`-only pair. Missing `actions: read` specifically
fails *after* the author check passes, with a `403: Resource not
accessible by integration` on the first `gh run list --workflow` call --
a real incident: the author check itself had its own bug for days
(comparing against the wrong "is this Dependabot" string), which masked
this one entirely, since every real Dependabot PR returned long before
reaching the gate-check loop.

**`gate_workflow_names` must list every workflow that runs on a
Dependabot PR** for that repo, and the caller's own `on: workflow_run:
workflows: [...]` must name the same ones -- otherwise a gate that isn't
listed in `workflows: [...]` never triggers a re-check when it completes,
and one that's listed there but missing from `gate_workflow_names` is
never actually waited on.

### Current callers and their config

| Repo | `gate_workflow_names` |
|---|---|
| trichonaut | `CI,Security` (two separate workflows) |
| trichonaut-catalog | `CI` (security is an embedded job inside it) |
| trichonaut-manage | `CI` (security is an embedded job inside it) |
| trichonaut-e2e | `Security` (no separate CI workflow exists) |
| trichonaut-infra | `Security` (quality checks are local pre-commit hooks, not CI) |

## Versioning

Callers reference `@main`. Since this repo has no independent release
process yet, a breaking change to either workflow's inputs/behavior
should bump to a tagged version (e.g. `@v1`) and update callers
deliberately, rather than changing `@main`'s behavior out from under
every caller at once. For now, with only a handful of callers all
maintained together, `@main` is simplest.
