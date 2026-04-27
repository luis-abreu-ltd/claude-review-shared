# claude-review-shared

Reusable GitHub Actions workflows for label-gated Claude Code reviews.
Consumed by `luis-abreu-ltd/*` repos.

## Why

Running the Claude Code review on every PR push burned credits on
diminishing-return findings. This repo splits the work in two:

1. A **scorer** that runs cheaply on every push, uses diff heuristics to
   decide whether the PR is worth a review, and applies a `review-please`
   label when it crosses a threshold.
2. A **review** job that runs only when that label is present.

Both are reusable workflows — consumer repos call them with thin stubs.

## Consumer setup

Pick a release tag (`@v1`) and add two stubs to `.github/workflows/`:

### `.github/workflows/claude-review.yml`

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [labeled]
jobs:
  review:
    if: github.event.label.name == 'review-please'
    uses: luis-abreu-ltd/claude-review-shared/.github/workflows/claude-review.yml@v1
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
    uses: luis-abreu-ltd/claude-review-shared/.github/workflows/claude-auto-label.yml@v1
    with:
      # Repo-specific tuning — see Inputs below
      engine_files_regex: '^diff --git a/src/lib/(scenario-engine|self-consumption)'
      invariant_regex: '\b(totalConsumption|gridImport|selfConsumed|toBeCloseTo)\b'
```

Create the `review-please` label once per repo:

```sh
gh label create review-please --color BFD4F2 --description "Trigger the Claude Code review workflow"
```

## Inputs

### `claude-review.yml`

| Input | Default | Notes |
|---|---|---|
| `model` | `claude-sonnet-4-6` | Override for testing newer models |
| `extra_focus` | `""` | Extra bullets appended to the review focus list |

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

- Force a review: `gh pr edit <num> --add-label review-please`
- Skip a review: remove the label before the workflow fires
  (`gh pr edit <num> --remove-label review-please`). The review job is
  triggered by the `labeled` event, so re-applying the label after a no-op
  skip works.
