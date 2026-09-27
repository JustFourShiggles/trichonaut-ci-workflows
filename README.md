# trichonaut-ci-workflows

A shared [reusable GitHub Actions workflow](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
for the trichonaut project family's security scanning (`audit-ci` / Semgrep
/ gitleaks). Public on purpose: GitHub does not allow a private repository's
reusable workflow to be called from another private repository unless both
are owned by the same organization — these repos are all under one personal
account, so a public host is the only way to share this without making an
existing project repo public. The workflow itself carries no
account-specific secrets, ARNs, or credentials; every project-specific
detail is passed in as an input by the caller.

## Usage

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

## Inputs

| Input | Default | Purpose |
|---|---|---|
| `audit_dirs` | `.` | Space-separated directories to run `audit-ci` against, one per `package.json`/`package-lock.json` pair. |
| `audit_allowlist` | `""` | Space-separated GHSA IDs to allowlist in `audit-ci` — for advisories with no upstream fix available. Applied identically to every directory in `audit_dirs`. |
| `semgrep_exclude` | `""` | Space-separated additional `--exclude` patterns for Semgrep (e.g. `terraform` when that's covered by a separate tool like Prowler instead). |
| `cache_dependency_path` | `package-lock.json` | Passed to `actions/setup-node`'s `cache-dependency-path`. `setup-node` only looks for a lockfile at the repo root by default (not recursively) — override this when the repo's only lockfile lives elsewhere. |

## Current callers and their config

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

## Versioning

Callers reference `@main`. Since this repo has no independent release
process yet, a breaking change to the workflow's inputs/behavior should
bump to a tagged version (e.g. `@v1`) and update callers deliberately,
rather than changing `@main`'s behavior out from under every caller at
once. For now, with only 5 callers all maintained together, `@main` is
simplest.
