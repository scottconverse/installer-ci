# Rollout Plan

ROLLOUT PLAN (personal account — no org templates, no required-workflows, no org rulesets)

Phase 0 — Create the central public CI repo (one time)
  1. gh repo create scottconverse/installer-ci --public \
       --description "Reusable installer-test workflow + onboarding tooling" --add-readme
  2. Clone it, then add the reusable workflow at its canonical path:
       installer-ci/.github/workflows/installer-test.yml   (the reusableWorkflowYaml)
     Also add scripts/onboard-installer-ci.sh (the bootstrapScript), chmod +x it.
  3. Commit + push to main. Because the reusable workflow ALSO has push/PR/dispatch
     triggers, pushing it to installer-ci runs it against installer-ci itself —
     installer-ci has no installer manifests, so detect emits matrix=[] noop=true
     and the `test` job is skipped cleanly. That is the first proof the detect
     job + noop gate work. (Optional: gh workflow run installer-test.yml
     -f force_types=python-pip-wheel to force a path for a smoke test.)
  4. Pin the ref strategy: callers use @main. If you later want immutability,
     tag the reusable repo (e.g. v1) and flip callers to @v1 — the caller YAML is
     the only thing that changes, and onboard-installer-ci.sh can re-PR it.

Phase 1 — Reusability prerequisite (IMPORTANT for personal/public repos)
  Reusable workflows in a PUBLIC repo can be called by other repos with NO extra
  setting. (If you ever make installer-ci PRIVATE, you must enable
  Settings → Actions → "Access" → allow this repo to be used by other repos in
  the account.) Keep installer-ci PUBLIC and this is free + frictionless.

Phase 2 — Onboard the real installer PRODUCERS first (highest signal)
  Run, one at a time, watching each PR's checks before merging:
     ./scripts/onboard-installer-ci.sh klodock                 # tauri (deb/appimage/dmg/nsis) + deb-rpm
     ./scripts/onboard-installer-ci.sh civiccast               # tauri(nsis) + cargo + python + shell
     ./scripts/onboard-installer-ci.sh civic-transparency-toolkit  # pyinstaller + python-pip-wheel
     ./scripts/onboard-installer-ci.sh patentforge             # docker-image
  Each opens a branch `ci/add-installer-test` + PR adding the 16-line caller.
  Merge once green. On first run there are no prior releases, so every G6 upgrade
  step takes its graceful-skip path (::notice) — expected and correct. Repos with
  per-app env (env.APP_DIR / env.TAURI_DIR for tauri, PKG_NAME for deb-rpm,
  APP_BIN/IS_GUI for shell-installer) should set those repo-level Actions variables
  or add a thin caller-side env; defaults are wired so a run still executes.

Phase 3 — Onboard the ~7 python-package repos
  These detect as python-pip-wheel (Linux-only, free, fast). Onboard individually
  or in a small batch:
     for r in pkg1 pkg2 pkg3 pkg4 pkg5 pkg6 pkg7; do ./scripts/onboard-installer-ci.sh "$r"; done
  (Substitute the real repo names.) Library-only wheels skip pipx/system-scope and
  use the import smoke test; CLI wheels exercise the console_script for real (G5).

Phase 4 — Optional full sweep of everything else (39 repos total)
  ./scripts/onboard-installer-ci.sh --all --dry-run   # review what would be touched
  ./scripts/onboard-installer-ci.sh --all             # PR the caller into every repo
  Prompt/skill/docs repos with no installer manifest detect noop=true and the
  test job is skipped — the caller is harmless there and future-proofs the repo if
  it later grows an installer. The sweep is idempotent: re-running skips any repo
  that already has the caller on its default branch or an open onboarding PR.

Operating notes
  * Single source of truth: edit the reusable workflow ONLY in installer-ci. All
    callers pull @main, so a fix lands everywhere on the next run — no fan-out edit.
  * Cost: targets are public, GitHub-hosted minutes are free and OS multipliers do
    not apply, so the full 3-OS matrix (tauri/electron/pyinstaller/dotnet/go/cargo)
    is fine. concurrency cancel-in-progress keeps PR pushes from stacking runs.
  * Token: the reusable workflow needs only contents:read (+packages:read for GHCR
    pulls in docker-image). The caller passes `secrets: inherit` so the implicit
    GITHUB_TOKEN flows through for `gh release download` and GHCR login.

# Template Repo Plan

TEMPLATE-REPO PLAN — make new repos ship with the caller from day one (personal-account-friendly)

Goal: complement the scheduled sweep with a "born-onboarded" path, so repos created
from a template already contain the caller and don't even need a sweep PR.

1. Create a GitHub TEMPLATE repository: scottconverse/repo-template
     gh repo create scottconverse/repo-template --public --add-readme
     Then in Settings → check "Template repository".
   Personal accounts fully support template repos (this is the personal-account
   analog of org repo templates / org-required-workflows, which are NOT available
   here).

2. Seed the template with the caller already in place:
     repo-template/.github/workflows/installer-test.yml   = the callerWorkflowYaml
     (uses scottconverse/installer-ci/.github/workflows/installer-test.yml@main,
      secrets: inherit, contents:read + packages:read).
   Optionally seed a .github/workflows/README note pointing maintainers to
   scottconverse/installer-ci as the single source of truth, and a starter
   .gitignore / LICENSE. Do NOT copy the reusable workflow body into the template —
   only the thin caller, so updates stay centralized.

3. New-repo workflow:
     gh repo create scottconverse/<newproj> --public --template scottconverse/repo-template
   The new repo starts with the caller; the first push to main triggers detect →
   (noop skip if no installer yet, or the real matrix once a manifest is added).
   No onboarding PR needed.

4. Keep template + sweep complementary:
   * Template covers repos you CREATE from it going forward.
   * The scheduled sweep covers repos created WITHOUT the template (e.g. `gh repo
     create` without --template, forks, imports) and any pre-existing repos.
   * Both reference @main, so the reusable workflow itself is edited in exactly one
     place (installer-ci) and every consumer picks up the change on its next run.

5. Caveat to document in the template README: GitHub does NOT auto-update files
   copied from a template into already-created repos. So the template guarantees the
   caller at creation time only; ongoing drift (e.g. bumping @main → @v2 in the
   caller) is handled by re-running onboard-installer-ci.sh / the sweep, which
   re-PRs the refreshed caller. This division (template = initial seed, sweep =
   ongoing reconciliation, installer-ci = workflow logic) gives org-like coverage on
   a personal account.

# Scheduled Sweep Plan

SCHEDULED-SWEEP PLAN — auto-PR the caller into any repo missing it (future-repo coverage without an org)

Why: a personal account has no org-level "required workflows" or repo-creation
templates, so new repos won't get the caller automatically. A cron-driven sweep in
the central repo closes that gap: it re-runs the idempotent onboarding bootstrap on
a schedule, PRing the caller into any repo that lacks it (including repos created
after the initial rollout).

Where it lives: scottconverse/installer-ci/.github/workflows/sweep.yml

How it works:
  * schedule: weekly cron (Mondays 07:00 UTC). Also workflow_dispatch for on-demand.
  * Auth: the default GITHUB_TOKEN is scoped to installer-ci ONLY and cannot open
    PRs in OTHER repos. So the sweep must use a Personal Access Token (classic PAT
    with `repo` + `workflow` scopes, or a fine-grained PAT granting Contents:RW +
    Pull requests:RW + Workflows:RW across the account's repos), stored as the repo
    secret GH_SWEEP_TOKEN. The job exports it as GH_TOKEN so `gh` and the bootstrap
    act account-wide.
  * It checks out installer-ci to get scripts/onboard-installer-ci.sh, then runs it
    with --all. Idempotency in the script guarantees: repos that already have the
    caller (or an open ci/add-installer-test PR) are skipped; only genuinely-missing
    repos get a fresh branch + PR. Archived/empty repos are skipped.
  * Result each week: zero-noise on already-onboarded repos; a new PR appears only
    for repos created/un-archived since the last sweep. You review + merge those PRs.

sweep.yml (paste-ready):

  name: sweep-installer-ci-callers
  on:
    schedule:
      - cron: '0 7 * * 1'      # Mondays 07:00 UTC
    workflow_dispatch:
      inputs:
        dry_run:
          description: 'List actions without creating PRs'
          type: boolean
          default: false
  permissions:
    contents: read             # only needs to read THIS repo's script
  concurrency:
    group: sweep-installer-ci
    cancel-in-progress: false
  jobs:
    sweep:
      runs-on: ubuntu-latest
      timeout-minutes: 30
      steps:
        - uses: actions/checkout@v4
        - name: Run idempotent onboarding sweep
          shell: bash
          env:
            GH_TOKEN: ${{ secrets.GH_SWEEP_TOKEN }}   # PAT with account-wide repo+workflow scope
          run: |
            set -euo pipefail
            chmod +x scripts/onboard-installer-ci.sh
            if [ "${{ inputs.dry_run }}" = "true" ]; then
              ./scripts/onboard-installer-ci.sh --all --dry-run
            else
              ./scripts/onboard-installer-ci.sh --all
            fi

Operational guidance:
  * Start with dry_run=true via workflow_dispatch to confirm the PAT sees every repo
    and the skip logic behaves before letting the cron open PRs.
  * Rotate GH_SWEEP_TOKEN on the usual cadence; a fine-grained PAT scoped to "all
    repositories" with the three permissions above is least-privilege for this.
  * The sweep never merges — it only PRs. Branch protection / your review stays in
    control. If you adopt tagged refs (@v1) later, bump the version in the caller_yaml
    heredoc inside the script and the next sweep PRs the updated caller to repos that
    are behind.