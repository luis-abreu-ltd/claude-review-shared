# Release standard

Canonical release process for repos owned by Luis Abreu (`luis-abreu-ltd/*`)
and the Doist-org repos he maintains (`Doist/figma-changelog`,
`Doist/productist`).

This document is the source of truth. Repo-level `docs/RELEASE.md` files
should link here rather than restating the policy. A repo can override
any rule in its own `CLAUDE.md` with a written reason (e.g. "this is a
library with published npm tags" or "releases are managed by a separate
tool").

## SemVer for apps and libraries alike

- Bump **patch** for fixes.
- Bump **minor** for additive changes.
- Bump **major** for breaking changes.
- For pre-1.0 projects, treat **minor** as the "breaking" lever.

## CHANGELOG.md

- Lives at the repo root.
- Keep-a-Changelog-ish format, latest release at the top.
- Section heading: `## X.Y.Z — Short Title` (one title line, then bullets).

## A release is a commit on `main`

A release is a single commit on `main` that edits `CHANGELOG.md`. No
release branches, no release PRs.

- Commit message: `chore: release X.Y.Z`
- Tag every release: `git tag vX.Y.Z` on the release commit, then
  `git push --tags` (or push commit + tag atomically).

Tags unlock `git checkout vX.Y.Z` to reproduce a build, GitHub's
auto-generated Releases page, and `git describe` in CI for the version
string. Cost is ~zero.

## `scripts/release.sh`

Each repo ships `scripts/release.sh` that:

1. Reads the top of `CHANGELOG.md` to suggest the next version.
2. Opens `$EDITOR` on `CHANGELOG.md` with a stub `## X.Y.Z — ` inserted
   at the top.
3. On editor close, validates the new heading exists.
4. Commits as `chore: release X.Y.Z`.
5. Tags `vX.Y.Z`.
6. Pushes commit and tag (atomically).
7. Bails cleanly if nothing changed.

Canonical implementation: `Doist/figma-changelog/scripts/release.sh`.

## Twist announce — Doist-org repos only

Doist-org repos (`Doist/figma-changelog`, `Doist/productist`, etc.) ship
`.github/workflows/announce-release.yml` that triggers on push to `main`
with `paths: CHANGELOG.md`, extracts the top `## ` section, and posts
via `Doist/twist-post-action`.

Required repo secrets:
`TWIST_RELEASE_INSTALL_ID`, `TWIST_RELEASE_INSTALL_TOKEN`. They point at
the Twist thread the repo releases to (defaults: design-team or project
squad thread).

Personal repos (`luis-abreu-ltd/*`, `lmjabreu/*`) keep `CHANGELOG.md` as
a human-readable history but do **not** ship `announce-release.yml` —
there's no Twist thread to post to. Still use `chore: release X.Y.Z`
commits, tag, and push tags as above.

## Bootstrapping a new repo

**New Doist-org repo:**

1. Drop in a skeleton `CHANGELOG.md`.
2. Copy `announce-release.yml` and `scripts/release.sh` from
   `Doist/figma-changelog`.
3. Set the two Twist secrets above.

**New personal repo:**

1. Skeleton `CHANGELOG.md`.
2. `scripts/release.sh` (no Twist workflow).
