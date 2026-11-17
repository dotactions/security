# dotactions/security

Reusable security-scanning workflows. Findings are reported by [`dotactions/sarif-reporter`](https://github.com/dotactions/sarif-reporter).

## Versioning

Pin the **full commit SHA** of a release tag, with the version in a comment, or a floating tag. While on `0.x`, minor releases may break you:

```yaml
uses: dotactions/security/.github/workflows/trivy-audit.yml@<40-char-sha> # v0.1.0
uses: dotactions/security/.github/workflows/trivy-audit.yml@v0.1
```

Resolve a SHA with `git ls-remote https://github.com/dotactions/security 'refs/tags/v0.1.0^{}'`. Only SHAs of **release tags** pin the whole tree; a SHA from `main` still contains `@main` self-references.

| Ref | Moves on | Use |
| :-- | :-- | :-- |
| `@<sha>` | never | **Recommended**: immutable, full reproducibility |
| `@v0.1` | every `0.1.x` release | Fixes, no breaking changes |
| `@v0.1.0` | never (tag can be moved) | Readable, but tags are mutable |

Release notes: [GitHub Releases](https://github.com/dotactions/security/releases).

Reusable workflows can only narrow the caller's token permissions, never widen them. Grant the permissions listed under each workflow in the calling job.

## Trivy Security Audit (`trivy-audit.yml`)

Runs Trivy vulnerability, secret and misconfiguration scanning on the caller repository, parses findings, publishes sticky PR comments / job summaries via [`dotactions/sarif-reporter`](https://github.com/dotactions/sarif-reporter), and optionally uploads SARIF artifacts.

### Permissions Required

Both `write` permissions are required even with `manage-issue: false`: the called job requests them, and GitHub rejects the run if the caller grants less. Call this workflow once per calling workflow: the artifact name (`trivy-sarif`) and comment tag are fixed.

```yaml
permissions:
    contents: read
    pull-requests: write # sticky PR comment
    issues: write # tracking issue; required even if manage-issue is false
```

### Inputs

| Name               | Description                                | Required | Default                  |
| :----------------- | :----------------------------------------- | :------- | :----------------------- |
| `scan-type`        | Trivy scan type: `fs` or `config`         | No       | `"fs"`                   |
| `severity`         | Comma-separated severity levels to include | No       | `"CRITICAL,HIGH,MEDIUM"` |
| `fail-on-findings` | Fail the workflow if findings are detected | No       | `false`                  |
| `upload-artifact`  | Upload SARIF scan report as a job artifact | No       | `true`                   |
| `manage-issue`     | Create/update/close tracked GitHub issue   | No       | `false`                  |

### Secrets

| Name    | Description                                                  | Required |
| :------ | :----------------------------------------------------------- | :------- |
| `token` | GitHub token with pull-requests and issues write permissions | No       |

### Usage

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
    push:
        branches: [main]
        paths-ignore:
            - "**.md"
    pull_request:
        branches: [main]
        paths-ignore:
            - "**.md"
    schedule:
        - cron: "0 4 * * 4" # weekly Thursday at 04:00 UTC
    workflow_dispatch:

concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
    audit:
        uses: dotactions/security/.github/workflows/trivy-audit.yml@v0.1
        permissions:
            contents: read
            pull-requests: write
            issues: write
        with:
            severity: "CRITICAL,HIGH"
            fail-on-findings: true
            manage-issue: ${{ github.event_name == 'schedule' || github.event_name == 'workflow_dispatch' }}
```

## Releasing (maintainers)

1. Merge to `main` with conventional-commit PR titles; they become the release notes.
2. Go to **Actions → Release → Run workflow** on `main` and enter a version: `0.1.1`, or `0.2.0-rc.1` for a pre-release that moves no floating tags.

Don't use **Releases → Draft a new release**: it doesn't move the floating tags, and doesn't rewrite `@main` self-references if this repo ever gains them. The workflow (`internal-release.yml`, which calls [dotactions/tooling](https://github.com/dotactions/tooling)) does both, then creates the GitHub release with generated notes.

While on `0.x`, bump the **minor** version for breaking changes and the **patch** version for everything else. From `1.0.0`: major for breaking changes, minor for features, patch for fixes.

The workflow authenticates as a GitHub App and needs the `RELEASE_APP_PRIVATE_KEY` secret, the `RELEASE_APP_CLIENT_ID` org variable, and the app installed on this repo with **Contents** and **Workflows** read & write.
