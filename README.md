# skaphos/actions

Reusable composite GitHub Actions for release automation. Small, opinionated, and deliberately narrow: they run a single gate (open a PR to bump the manifest; push the tag on merge) and let your existing release tooling — `goreleaser`, Docker publish, npm publish, Helm chart push, whatever — own everything downstream.

> **Status:** scaffolding. The two action.yml files are landing in follow-up PRs on this repo. This README describes the contract they will implement.

## Why not release-please?

[`googleapis/release-please-action`](https://github.com/googleapis/release-please-action) is the industry default for this pattern, but it bundles a lot more than "open a version-bump PR":

- It generates and maintains `CHANGELOG.md` in the consumer repo.
- It creates the GitHub release object (colliding with orgs that have immutable releases enabled — see [`skaphos/repokeeper` ADR-0007](https://github.com/skaphos/repokeeper/blob/main/docs/adr/0007-release-binaries-and-homebrew.md)).
- It manages multi-package manifests, plugin configs, and a `release-please-config.json`.
- Under `skip-github-release: true`, it stops pushing the git tag too — callers have to add a workaround workflow step to push it themselves.

If your release tool of choice (for us: `goreleaser`) already creates the GitHub release, generates the changelog, signs artifacts, builds SBOMs, and publishes to downstream registries, everything release-please does *except* the PR-based version-bump gate is duplicate work. These two actions do only the gate.

## Design

Three opinions, all narrow on purpose:

1. **One file changes per release PR: the version manifest.** No generated changelog, no release notes stub in the repo. Your release tool owns the changelog.
2. **Semver is inferred from Conventional Commits via [`svu`](https://github.com/caarlos0/svu).** `feat:` → minor, `fix:`/`perf:`/`refactor:` → patch, `!` or `BREAKING CHANGE:` → major. Inputs let you force any bump.
3. **Commits on the release branch go through the GitHub Contents API,** producing verified commits signed by GitHub's web-flow key. No bot SSH keys to provision or rotate. DCO sign-off is added to the commit message.

## TL;DR — minimum viable setup

Two tiny workflows in the consumer repo, one manifest file, and whatever your real release tool is listening on `v*` tag pushes.

```yaml
# .github/workflows/release-pr.yml
name: Release PR
on:
  push:
    branches: [main]
permissions:
  contents: write
  pull-requests: write
jobs:
  release-pr:
    runs-on: ubuntu-latest
    steps:
      - uses: skaphos/actions/release-pr@v1
        with:
          token: ${{ secrets.RELEASE_BOT_TOKEN }}
```

```yaml
# .github/workflows/release-tag.yml
name: Release Tag
on:
  pull_request:
    types: [closed]
permissions:
  contents: write
jobs:
  release-tag:
    if: github.event.pull_request.merged && contains(github.event.pull_request.labels.*.name, 'release-pr')
    runs-on: ubuntu-latest
    steps:
      - uses: skaphos/actions/release-tag@v1
        with:
          token: ${{ secrets.RELEASE_BOT_TOKEN }}
```

```json
// .release-please-manifest.json (kept at repo root; name is historical)
{ ".": "0.1.0" }
```

On every push to `main`, `release-pr` recomputes the next version and opens or force-updates a `release/v<next>` PR labelled `release-pr` that changes only the manifest. Merge the PR when you're ready to cut; `release-tag` reads the new version from the manifest and pushes `v<next>`. Your `goreleaser` (or equivalent) workflow listening on `push: tags: 'v*'` takes it from there.

For an end-to-end reference with `goreleaser`, Homebrew cask publishing, SBOMs, `cosign` signatures, and GitHub attestations, see [`examples/goreleaser-go-cli`](./examples/goreleaser-go-cli).

## Actions

### `skaphos/actions/release-pr`

Detects commits beyond the last `v*` tag, computes the next semver, and opens or updates a release PR.

#### Inputs

| Name | Default | Required | Description |
|---|---|---|---|
| `token` | — | yes | GitHub token with `contents: write` and `pull-requests: write`. Prefer a GitHub App installation token so release PRs are authored under a bot identity that can trigger downstream workflows. |
| `manifest-path` | `.release-please-manifest.json` | no | Path to the JSON manifest that holds the version. Any JSON file with string values works; the name is kept for release-please compatibility. |
| `manifest-key` | `.` | no | JSON key inside the manifest. Monorepos can run this action multiple times with different keys. |
| `bump` | `auto` | no | One of `auto`, `major`, `minor`, `patch`. `auto` runs `svu next` (infers from Conventional Commits since the last tag). Anything else forces that bump via `svu <kind>`. |
| `tag-prefix` | `v` | no | Prefix prepended to the version when reading/writing tags. |
| `branch-prefix` | `release/v` | no | Prefix for the release PR branch name. |
| `pr-label` | `release-pr` | no | Label applied to the release PR. Keep this in sync with the `release-tag` trigger filter. |
| `pr-title-template` | `chore(release): release {{tag}}` | no | Mustache-ish template. Available: `{{version}}`, `{{tag}}`, `{{manifest-key}}`. |
| `pr-body-template` | built-in | no | Override the PR body. See "PR body" below for the default. |
| `committer-name` | derived from `token` | no | Name for the API-created commit. |
| `committer-email` | derived from `token` | no | Email for the API-created commit. |
| `svu-version` | pinned in `action.yml` | no | Override the `svu` release pulled into the runner. |

#### Outputs

| Name | Description |
|---|---|
| `version` | Computed next version, without prefix (e.g. `1.2.3`). |
| `tag` | Computed tag that `release-tag` will push (e.g. `v1.2.3`). |
| `pr-number` | The opened or updated PR number. Empty when `skipped=true`. |
| `pr-url` | HTML URL of the PR. Empty when `skipped=true`. |
| `created` | `true` if the PR was opened in this run; `false` if an existing PR was updated. |
| `skipped` | `true` if HEAD matches the last `v*` tag (no commits to release). |

#### Behavior

1. Install `svu` (pinned binary, SHA-verified).
2. Read the last `v*` tag. If it's missing (first release), treat as `v0.0.0`.
3. Compare `git rev-list $LAST_TAG..HEAD`. If empty, set `skipped=true` and exit.
4. Compute the next version from `svu next` (or `svu <kind>` when `bump` is forced).
5. If the next version equals the current manifest version, set `skipped=true` and exit (keeps idempotent on re-runs).
6. Use `gh api PUT /repos/$REPO/contents/$MANIFEST_PATH` to commit the updated manifest onto `release/v<next>`. The request branch is created on the fly if it doesn't exist; otherwise the API updates it. The resulting commit is signed by GitHub's web-flow key — verified, no SSH key plumbing needed.
7. The commit message includes a DCO `Signed-off-by:` trailer derived from the committer identity.
8. If a PR with `head = release/v<next>` exists, capture its number; otherwise create a new PR with the label, title, and body.

### `skaphos/actions/release-tag`

On merge of the release PR, reads the bumped manifest and pushes an annotated tag.

#### Inputs

| Name | Default | Required | Description |
|---|---|---|---|
| `token` | — | yes | GitHub token with `contents: write`. Use an App token (not `GITHUB_TOKEN`) so downstream workflows (`on: push: tags: 'v*'`) will fire. |
| `manifest-path` | `.release-please-manifest.json` | no | Path to the manifest committed by `release-pr`. |
| `manifest-key` | `.` | no | JSON key inside the manifest. |
| `tag-prefix` | `v` | no | Prefix prepended to the version. |
| `tagger-name` | derived from `token` | no | Name for the annotated-tag tagger. |
| `tagger-email` | derived from `token` | no | Email for the annotated-tag tagger. |

#### Outputs

| Name | Description |
|---|---|
| `tag` | The pushed tag (e.g. `v1.2.3`). |
| `version` | The version without prefix (e.g. `1.2.3`). |
| `pushed` | `true` if the tag was created by this run; `false` if it already existed. |

#### Trigger shape

Meant to run from `pull_request: closed`. Always guard with `github.event.pull_request.merged` and a label check so it only fires on merged release PRs:

```yaml
on:
  pull_request:
    types: [closed]
jobs:
  release-tag:
    if: github.event.pull_request.merged && contains(github.event.pull_request.labels.*.name, 'release-pr')
```

## Recommended token setup

Create a GitHub App in your org (skaphos uses `skaphos-release-bot`) with these repository permissions:

- **Contents:** Read & Write (to commit the manifest and push the tag)
- **Pull requests:** Read & Write (to open/update the release PR)
- **Metadata:** Read (mandatory)

Install the app on each consumer repo, mint a short-lived installation token via [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token), pass it to both actions. See [`examples/goreleaser-go-cli`](./examples/goreleaser-go-cli) for the full pattern.

The default `GITHUB_TOKEN` works for `release-pr` but has a known limitation for `release-tag`: tag pushes made with it won't trigger downstream `on: push: tags:` workflows. Use an App token for `release-tag`.

## Versioning

This repo follows the GitHub Actions versioning convention used by `actions/checkout`, `actions/setup-go`, `sigstore/*`:

- **Pinned semver tags** — `v1.0.0`, `v1.1.0`, `v2.0.0`. Immutable; each corresponds to one commit.
- **Floating major tag** — `v1`, `v2`. Reassigned on every release inside that major so consumers who pin `@v1` pick up non-breaking improvements.
- **Breaking changes** — new major tag, new floating major; announcement in the release notes and this README.

Consumers choose:

- `@v1` — convenience; you get bug fixes automatically.
- `@v1.2.3` — strict; you pin a specific release.
- `@<commit-sha>` — maximum supply-chain safety; Dependabot can bump.

**Do not pin to `main`.** Main is the development branch; it will break you.

## Security notes

- Commits on the release PR branch are created via the Contents API and are signed by GitHub's web-flow GPG key (verified). No SSH key needs to be provisioned to the bot.
- The actions themselves pin the `svu` release by SHA and verify the download. Update the pin by cutting a new action release.
- Consumers should pin these actions by SHA in production (`@<sha>`) and let Dependabot update the pin via PR. `@v1` is fine for lower-risk projects.
- No secrets are logged. `token` is marked sensitive; debug output redacts it.

## Examples

- [`examples/goreleaser-go-cli`](./examples/goreleaser-go-cli) — complete reference pipeline mirroring `skaphos/repokeeper`: Go CLI, `goreleaser` on tag push, Homebrew cask, SBOMs, `cosign` signatures, GitHub attestations, app-token minting.

## Contributing

Issues and PRs welcome. See [`AGENTS.md`](./AGENTS.md) for commit style (signed + DCO) and the lint/test commands CI runs.

## License

[MIT](./LICENSE).
