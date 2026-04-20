# Repository Guidelines

## Project Structure
This repository hosts reusable composite GitHub Actions for the `skaphos` organization. Each action lives in its own top-level directory with a single `action.yml`.

- `release-pr/action.yml` — opens or updates a release PR that bumps `.release-please-manifest.json` based on conventional commits (driven by [`svu`](https://github.com/caarlos0/svu)).
- `release-tag/action.yml` — on merge of a release PR, creates and pushes the annotated `vX.Y.Z` tag.
- `.github/workflows/` — CI for this repo only (lint actions, shellcheck embedded scripts).

Composite YAML only — no JavaScript, no Docker images.

## Build, Test, and Development
- `actionlint` against every `action.yml` and every workflow in `.github/workflows/`.
- `shellcheck` on embedded `run:` blocks (CI runs both).
- Local smoke tests against a throwaway test branch in a scratch repo; document the procedure in each action's directory if it has one.

## Coding Style
- `action.yml` keys follow GitHub's standard order: `name`, `description`, `inputs`, `outputs`, `runs`.
- Embedded shell: `#!/usr/bin/env bash`, `set -Eeuo pipefail`, no `set +e` short-circuits. Quote everything.
- Inputs use kebab-case (`manifest-path`, not `manifestPath`).
- Pin third-party action references by SHA in CI; semver tags for the action's own consumers (`@v1`).

## Commit & Pull Request Guidelines
- **All changes via a PR. Never commit directly to `main`.**
- All commits must be **cryptographically signed AND carry a DCO sign-off** — always pass `-S -s` to `git commit`.
- Conventional Commits so `svu` (used by the release-pr action to release *this* repo) can infer the next version:
  - `feat:` -> minor bump
  - `fix:` / `perf:` / `refactor:` -> patch bump
  - `!` in the type/scope or `BREAKING CHANGE:` footer -> major bump
- Branch prefixes: `feat/`, `fix/`, `chore/`, `docs/`, `ci/`.
- PRs should include: summary, the consumer-facing diff (new inputs/outputs, breaking changes), and any backport plan.

## Versioning
This repo follows the GitHub Actions versioning convention:
- Pinned semver tags (`v1.0.0`, `v1.1.0`, …) — immutable.
- Floating major tag (`v1`) — reassigned on every release within that major.
- Consumers pin to `@v1` for convenience or `@v1.2.3` for strict. Never pin to `main` in production.
