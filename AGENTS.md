# Agents Rules for dotactions/security

Reusable security-scanning workflows, consumed by other repositories. Reporting is done by `dotactions/sarif-reporter`; this repo only wires scanners to it.

## CRITICAL INSTRUCTIONS

1. **The Ponytail Workflow (Default Execution Mode)**:
    - Climb the senior dev ladder: YAGNI -> Reuse -> Stdlib -> Platform -> Deps -> Simplicity -> Minimal Diff.
    - Deletion over addition. Boring over clever.
    - Mark intentional simplifications with `ponytail:` comments (`# ponytail:` in YAML/shell/PowerShell, `// ponytail:` in TypeScript) naming the known ceiling and upgrade path.
    - Non-trivial logic must leave behind at least one runnable check (see [Testing](#testing)).

2. **Commit Policy**:
    - AI assistants are strictly prohibited from generating, suggesting, or running git commits, pushes, tags, or releases under any circumstance. Commits and releases are human-only.

3. **Plan Execution Policy**:
    - **Never** proceed to code changes or implementation commands directly from a plan unless the human developer explicitly replies with direct authorization (e.g., "go ahead", "approved", "execute").
    - A plan is strictly for review. Keep your hands off the files until permission is explicitly given.

4. **Comments & Standards**:
    - Strictly NO inline comments explaining syntax or obvious actions. Comments explain _why_ (constraints, GitHub quirks, security reasons), never _what_.
    - Ceiling of 300 lines per file (excluding `README.md`, lockfiles, fixtures and `dist/`). Always ask before performing major file splits or autonomous refactoring.

## Repository Layout

| Path | Purpose |
| :--- | :--- |
| `.github/workflows/trivy-audit.yml` | Public reusable workflow (`workflow_call`): Trivy → SARIF → `dotactions/sarif-reporter` |
| `.github/workflows/semgrep.yml` | Public reusable workflow: Semgrep OSS (pinned image) → SARIF → `dotactions/sarif-reporter` |
| `.github/workflows/internal-security-audit.yml` | Runs `trivy-audit.yml` and `semgrep.yml` (`p/github-actions`) on this repo |
| `.github/workflows/internal-release.yml`, `internal-check-refs.yml`, `internal-actionlint.yml` | Thin callers of `dotactions/tooling` (see [Shared CI](#shared-ci-dotactionstooling)) |

## Pinning & References

- Every third-party `uses:` is pinned to a full commit SHA with a version comment: `uses: owner/repo@<40-char-sha> # vX.Y.Z`. Resolve SHAs with `git ls-remote --tags`; never guess or invent one. If you cannot resolve it, stop and ask.
- Another `dotactions` repo is pinned the same way, to the commit of its **release tag** (`git ls-remote https://github.com/dotactions/<repo> 'refs/tags/vX.Y.Z^{}'`), never to a `main` SHA: only release commits have their own `@main` self-references rewritten. `internal-check-refs.yml` fails on anything else.
- This repo has no actions of its own, so it has no `@main` self-references.

## Shared CI (`dotactions/tooling`)

- `internal-release.yml`, `internal-check-refs.yml` and `internal-actionlint.yml` are thin callers of `dotactions/tooling/.github/workflows/*.yml`. They only hold `on:`/`paths:`, `permissions:` and `with:`. Logic changes go to `tooling`, not here.
- Callers never declare `concurrency:`; the called workflow does, and an identical group in caller and callee deadlocks.

## Inputs, Scripts & Security

- Never interpolate `${{ }}` directly into `run:`. Pass values through `env:` and reference `"$VAR"`.
- Validate inputs early and fail with `::error::` messages naming the allowed values.
- Workflows declare `permissions: contents: read` at the top level and widen per job only where required.
- Input defaults must not duplicate data that lives elsewhere (severity and report options belong to the scanner and `sarif-reporter`).

## Public API & Versioning

- Reusable-workflow `inputs:`/`secrets:`, their defaults and artifact names are public API. Renaming, removing or changing the default of any of them is a **breaking change**: flag it explicitly in your response so the human can bump the major version.
- Every public input/output change must be reflected in `README.md` (tables + usage examples).
- Releases are cut only by the human via the `Release` workflow (`internal-release.yml`). Never create or move tags.

## Testing

- `internal-security-audit.yml` exercises `trivy-audit.yml` end to end; changes to it are verified in CI.
- Local checks before finishing (run whichever apply):
    - `docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12` for workflow changes
- Report honestly which checks ran and which could not (e.g. multi-OS builds only run in CI).
