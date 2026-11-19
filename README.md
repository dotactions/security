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

Both `write` permissions are required even with `manage-issue: false`: the called job requests them, and GitHub rejects the run if the caller grants less. Call this workflow at most once per `scan-type` per calling workflow: the artifact name (`trivy-sarif-<scan-type>`) and comment tag (`trivy-audit-<scan-type>`) derive from it, so `fs` and `image` can run side by side.

```yaml
permissions:
    contents: read
    pull-requests: write # sticky PR comment
    issues: write # tracking issue; required even if manage-issue is false
```

### Inputs

| Name               | Description                                    | Required | Default                  |
| :----------------- | :--------------------------------------------- | :------- | :----------------------- |
| `scan-type`        | Trivy scan type: `fs`, `image` or `config`     | No       | `"fs"`                   |
| `image-ref`        | Image to scan; required for `scan-type: image` | No       | `""`                     |
| `severity`         | Comma-separated severity levels to include     | No       | `"CRITICAL,HIGH,MEDIUM"` |
| `fail-on-findings` | Fail the workflow if findings are detected     | No       | `false`                  |
| `upload-artifact`  | Upload SARIF scan report as a job artifact     | No       | `true`                   |
| `manage-issue`     | Create/update/close tracked GitHub issue       | No       | `false`                  |

### Outputs

| Name             | Description                  |
| :--------------- | :--------------------------- |
| `total-findings` | Number of findings reported  |

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
        paths-ignore: &paths
            - "**.md"
    pull_request:
        branches: [main]
        paths-ignore: *paths
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

## Semgrep SAST (`semgrep.yml`)

Runs Semgrep OSS (pinned image) with the given rulesets and reports the findings through [`dotactions/sarif-reporter`](https://github.com/dotactions/sarif-reporter). Metrics are off, so `--config auto` is not supported: name rulesets explicitly from the [Semgrep registry](https://semgrep.dev/explore) (`p/default`, `p/golang`, `p/typescript`, `p/github-actions`, …) or pass paths to your own rule files.

### Permissions Required

Same as `trivy-audit.yml`: `contents: read`, `pull-requests: write`, `issues: write`.

### Inputs

| Name               | Description                                                     | Required | Default       |
| :----------------- | :-------------------------------------------------------------- | :------- | :------------ |
| `config`           | Whitespace-separated rulesets or rule file/directory paths      | No       | `"p/default"` |
| `fail-on-findings` | Fail the workflow if findings are detected                      | No       | `false`       |
| `upload-artifact`  | Upload the SARIF report as `semgrep-sarif`                      | No       | `true`        |
| `manage-issue`     | Create/update/close tracked GitHub issue                        | No       | `false`       |
| `comment-tag`      | Sticky PR comment tag, distinct per call if called repeatedly   | No       | `"semgrep"`   |

Output `total-findings` and secret `token` work as in `trivy-audit.yml`.

### Usage

```yaml
# .github/workflows/semgrep.yml
name: Semgrep

on:
    push:
        branches: [main]
        paths-ignore: &paths
            - "**.md"
    pull_request:
        branches: [main]
        paths-ignore: *paths
    schedule:
        - cron: "0 4 * * 4" # weekly Thursday at 04:00 UTC
    workflow_dispatch:

concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
    semgrep:
        uses: dotactions/security/.github/workflows/semgrep.yml@v0.1
        permissions:
            contents: read
            pull-requests: write
            issues: write
        with:
            config: p/default p/golang .semgrep/ # registry rulesets plus your own rules directory
            fail-on-findings: true
            manage-issue: ${{ github.event_name == 'schedule' || github.event_name == 'workflow_dispatch' }}
```

To run it next to Trivy, add the `semgrep` job to the same workflow file as the `audit` job above.

## Releasing (maintainers)

1. Merge to `main` with conventional-commit PR titles; they become the release notes.
2. Go to **Actions → Release → Run workflow** on `main` and enter a version: `0.1.1`, or `0.2.0-rc.1` for a pre-release that moves no floating tags.

Don't use **Releases → Draft a new release**: it doesn't move the floating tags, and doesn't rewrite `@main` self-references if this repo ever gains them. The workflow (`internal-release.yml`, which calls [dotactions/tooling](https://github.com/dotactions/tooling)) does both, then creates the GitHub release with generated notes.

While on `0.x`, bump the **minor** version for breaking changes and the **patch** version for everything else. From `1.0.0`: major for breaking changes, minor for features, patch for fixes.

The workflow authenticates as a GitHub App and needs the `RELEASE_APP_PRIVATE_KEY` secret, the `RELEASE_APP_CLIENT_ID` org variable, and the app installed on this repo with **Contents** and **Workflows** read & write.
