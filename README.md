# claude-review-shared

Reusable GitHub Actions workflows for label-gated Claude Code reviews,
plus shared standards documents linked from consumer repos.

Consumed by `luis-abreu-ltd/*` repos and the Doist-org repos Luis
maintains.

## Shared standards

- [Release standard](docs/RELEASE-STANDARD.md) — SemVer, `CHANGELOG.md`
  format, `scripts/release.sh` contract, Twist-announce note for
  Doist-org repos.

## Why

Running the Claude Code review on every PR push burned credits on
diminishing-return findings. This repo splits the work in two:

1. A **scorer** that runs cheaply on every push, uses diff heuristics to
   decide whether the PR is worth a review, and applies a `review-please`
   label when it crosses a threshold.
2. A **review** job that runs only when that label is present.

Both are reusable workflows. Consumer repos call them with thin stubs.

## Consumer setup

Pick a release tag (`@v2`) and add two stubs to `.github/workflows/`:

### `.github/workflows/claude.yml`

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [labeled]
  workflow_dispatch:
    inputs:
      pr:
        description: PR number to review
        required: true
        type: number
jobs:
  review:
    if: github.event_name == 'workflow_dispatch' || github.event.label.name == 'review-please'
    uses: luis-abreu-ltd/claude-review-shared/.github/workflows/claude-review.yml@v2
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    with:
      pr_number: ${{ github.event.inputs.pr || github.event.pull_request.number }}
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### `.github/workflows/claude-auto-label.yml`

```yaml
name: Claude Auto-Label
on:
  pull_request:
    types: [opened, synchronize, reopened]
jobs:
  label:
    uses: luis-abreu-ltd/claude-review-shared/.github/workflows/claude-auto-label.yml@v2
    permissions:
      contents: read
      pull-requests: write
      actions: write
    with:
      dispatch_review: true
      # Repo-specific tuning, see Inputs below
      engine_files_regex: '^diff --git a/src/lib/(scenario-engine|self-consumption)'
      invariant_regex: '\b(totalConsumption|gridImport|selfConsumed|toBeCloseTo)\b'
```

Create the `review-please` label once per repo:

```sh
gh label create review-please --color BFD4F2 --description "Trigger the Claude Code review workflow"
```

## Why two triggers on the review workflow

The auto-labeller applies the `review-please` label using the default
`GITHUB_TOKEN`. GitHub deliberately does not let actions performed with
that token trigger downstream workflows (anti-recursion guard), so a
review workflow listening only on `pull_request: [labeled]` silently
no-ops when the label is bot-applied.

The fix: the auto-labeller (with `dispatch_review: true`) calls
`gh workflow run claude.yml -f pr=<num>` after applying the label.
`workflow_dispatch` is exempt from the anti-recursion guard, so the
review actually fires. The `pull_request: [labeled]` trigger is kept
for the legacy path where a human applies the label manually.

## Opt-out

Two ways, both checked at run time after a short delay (default 30s,
configurable via `skip_delay_seconds`) so you have a window to react to
the auto-labeller's PR comment:

1. **Remove the `review-please` label.** One-shot opt-out for the run
   that's currently waiting. Mirrors the v1 UX.
2. **Add `[skip-review]` (case-insensitive) anywhere in the PR body.**
   Sticky opt-out. Takes effect on every dispatched run, so it also
   covers later pushes without re-removing the label each time.

Either signal causes the job to log a single line and exit without
calling the Claude action. The review job uses a `concurrency:` group
keyed on the PR number, so if a label flap kicks off two runs, the
older one is cancelled.

## Inputs

### `claude-review.yml`

| Input | Default | Notes |
|---|---|---|
| `model` | `claude-sonnet-4-6` | Override for testing newer models |
| `extra_focus` | `""` | Extra bullets appended to the review focus list |
| `pr_number` | `""` | PR number. Required when called via `workflow_dispatch`. Optional under the legacy `pull_request: [labeled]` path; falls back to `github.event.pull_request.number`. |
| `skip_label` | `review-please` | Label whose absence at run time signals "skip review". Match the `label_name` used by the auto-labeller. Set to `""` to disable the label gate. |
| `skip_delay_seconds` | `30` | Seconds to sleep before re-checking the PR for skip signals. Set to `0` to disable. |

Required secret: `ANTHROPIC_API_KEY`.

### `claude-auto-label.yml`

| Input | Default | Notes |
|---|---|---|
| `label_name` | `review-please` | Match the `if:` in the review caller |
| `threshold` | `3` | Score required to label the PR |
| `api_prefix_regex` | `^diff --git a/(src/)?app/api/` | Override for non-Next.js repos |
| `lib_prefix_regex` | `^diff --git a/(src/)?lib/` | Override for repos with a different lib root |
| `engine_files_regex` | `""` (no nudge) | Per-repo: `^diff --git a/src/lib/(engine|core)/` |
| `invariant_regex` | `\b(toBeCloseTo\|conservation\|invariant\|balance)\b` | Per-repo: domain-specific identifiers |
| `large_added_lines` | `200` | |
| `large_changed_files` | `4` | |
| `dispatch_review` | `false` | When true, dispatch the consumer's review workflow via `gh workflow run` after labelling. Requires `actions: write` in the consumer's permissions block. |
| `dispatch_workflow_file` | `claude.yml` | Filename of the consumer workflow to dispatch when `dispatch_review` is true |

## Heuristic (in points)

| Signal | Points |
|---|---|
| Cache / TTL / invalidation surface | +3 |
| Touches both API prefix AND lib prefix (path unification) | +3 |
| Changes a balance/sum invariant | +3 |
| Removes `@deprecated` field or `fallback` line | +2 |
| Large diff (`> large_added_lines` OR `> large_changed_files`) | +2 |
| Touches `engine_files_regex` | +1 |

`.github/workflows/` and `.gitignore` are excluded from the scored diff so
the workflow that defines the regexes doesn't self-match.

## Manual override

- Force a review: add the `review-please` label to the PR. A dispatch
  (Actions UI, or `gh workflow run claude.yml -f pr=<num>`) only reviews a
  PR that already carries the label; on an unlabelled PR it skips at the
  gate. Pass `skip_label: ""` in the consumer stub to let dispatches
  bypass the gate.
- Skip a review: remove the `review-please` label within the
  `skip_delay_seconds` window after the bot's comment, or add
  `[skip-review]` to the PR body (sticky across pushes).

## Migrating from v1 to v2

1. Bump `@v1` to `@v2` in both `.github/workflows/claude.yml` and
   `.github/workflows/claude-auto-label.yml` of the consumer repo.
2. In `claude.yml`, add the `workflow_dispatch` trigger, widen the `if:`
   to also match `workflow_dispatch`, and pass `pr_number` through.
3. In `claude-auto-label.yml`, add `actions: write` to `permissions:`
   and set `dispatch_review: true` under `with:`.
4. v2 keeps "remove the label" as an opt-out gesture (handled at run
   time via the `skip_label` input + `skip_delay_seconds` window) and
   adds `[skip-review]` in the PR body as a sticky alternative. PR
   templates and contributor docs can mention either or both.

The v1 workflows remain on the `v1` tag for repos that haven't migrated.
