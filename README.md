# installer-ci

Central, reusable GitHub Actions workflow that **auto-detects how a repo installs software and tests that install end-to-end** — build → install → launch → upgrade-over-previous-release → uninstall — across Linux/macOS/Windows, for **13 installer types**:

`tauri` · `electron` · `pyinstaller` · `python-pip-wheel` · `npm-bin` · `go-binary` · `cargo-binary` · `docker-image` · `shell-installer` · `deb-rpm` · `homebrew-formula` · `dotnet` · `java-jar`

It is built to **never false-pass**: every leg captures real exit codes, asserts the artifact exists, launches the real installed program (not a wrapper PID), distinguishes "no prior release" from "download failed", asserts real uninstall removal, and a final **tripwire** fails any matrix leg that skipped its whole lifecycle. Signing/notarization is checked **warn-only** so public CI without signing secrets isn't blocked.

## Use it in a repo

Add `.github/workflows/installer-test.yml`:

```yaml
name: installer-test
on:
  push: { branches: [main] }       # use your default branch
  pull_request: { branches: [main] }
  workflow_dispatch:
jobs:
  installer-test:
    uses: scottconverse/installer-ci/.github/workflows/installer-test.yml@main
    permissions: { contents: read, packages: read }
    secrets: inherit
```

Or run the onboarding helper: `./bootstrap.sh <repo>` (one repo) / `./bootstrap.sh --all` (every repo). It opens a PR and is idempotent.

Repos with no installer (prompt/skill suites) **no-op green** — detection finds nothing and the test job is skipped.

## Knobs (workflow inputs)
- `force_types` — CSV to override detection (validated against the known 13; an unknown value hard-fails).
- `enable_system_scope` — also run system-scope install paths (default on).

## Repo-specific config (Actions Variables)
Some recipes read optional `vars.*` with sane defaults — e.g. Tauri: `vars.TAURI_PRODUCT_NAME` / `vars.APP_DIR` (default `.`) / `vars.TAURI_DIR` (default `src-tauri`).

## Layout
- `.github/workflows/installer-test.yml` — the assembled reusable workflow (source of truth)
- `recipes/*.steps.yaml` — per-type recipe sources the workflow is stitched from
- `bootstrap.sh` — onboarding helper · `ROLLOUT.md` — rollout plan · `VERIFY-FINDINGS.json` — adversarial-review notes
