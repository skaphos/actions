# Example: Go CLI with goreleaser, Homebrew, SBOMs, and cosign

The reference pipeline used by `skaphos/repokeeper`. Copy-paste it into any Go CLI project and adjust the binary name, module path, and Homebrew tap.

## What this gives you

On every merge to `main`:

- **Release PR auto-maintained** — one PR labelled `release-pr` that bumps `.release-please-manifest.json` to the next semver inferred from Conventional Commits. Force-updated in place as more commits land. Human reviews and merges when ready.

On merge of that release PR:

- **Tag pushed** — annotated `v<next>` tag created and pushed.

On tag push:

- **Binaries built** for `{linux,darwin,windows}/{amd64,arm64}` via `goreleaser`.
- **SBOMs generated** for every archive via `syft`.
- **Checksums signed** with `cosign` keyless (Sigstore OIDC against the GitHub Actions runner identity).
- **GitHub release created** with notes generated from Conventional Commits (grouped Features / Bug Fixes / Performance / Others). `goreleaser` owns the release body end-to-end.
- **Homebrew cask published** to a tap repo in the same org (app-token minted on the fly).
- **Build provenance attestations** attached via `actions/attest-build-provenance`.

No `CHANGELOG.md` in the consumer repo; the GitHub release body is the changelog surface.

## Architecture at a glance

```
Conventional commits land on main
          │
          ▼
  ┌─────────────────────┐
  │ release-pr.yml      │  (skaphos/actions/release-pr)
  │ triggers: push main │  Opens/updates release/v<next> PR.
  └─────────────────────┘  Changes only .release-please-manifest.json.
          │
          │  human reviews, merges
          ▼
  ┌─────────────────────┐
  │ release-tag.yml     │  (skaphos/actions/release-tag)
  │ triggers: PR closed │  Reads new version from manifest.
  └─────────────────────┘  Pushes annotated v<next> tag.
          │
          ▼
  ┌─────────────────────┐
  │ release.yml         │  goreleaser --clean
  │ triggers: push v*   │  Builds, signs, SBOMs, GitHub release, Homebrew.
  └─────────────────────┘
```

Three workflows, one `.goreleaser.yaml`, one manifest file, and a single installed GitHub App for the bot identity. That is the entire release pipeline.

## Prerequisites

Before dropping these workflows in, you need:

1. **A Go module** with a `main` package (CLI entry point) at the path `.goreleaser.yaml` points to.
2. **Conventional Commits** on the `main` branch. If existing history doesn't follow the convention, it's fine — `svu` will compute the next version from whatever it finds; the very first release often needs a forced `bump: major` or `bump: minor` to get on a clean footing.
3. **A GitHub App** installed on the repo with these repository permissions:
   - **Contents:** Read & Write
   - **Pull requests:** Read & Write
   - **Metadata:** Read

   Store `app-id` as the repo/org Variable `RELEASE_APP_ID`. Store the private key as the repo/org Secret `RELEASE_APP_PRIVATE_KEY`. Both actions mint installation tokens on the fly via [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token); there are no long-lived PATs in CI.
4. **A Homebrew tap repo** (optional, only if you want `brew install`) — a sibling repo named e.g. `<org>/homebrew-<tap>`. The same GitHub App needs Contents: Write on the tap repo, or a second app if you prefer isolation.
5. **An empty manifest file** at `.release-please-manifest.json` in the repo root:
   ```json
   { ".": "0.1.0" }
   ```
   The first manifest version seeds the starting point; `release-pr` will bump from there.
6. **Branch protection on `main`** — recommended: require PR reviews, require the checks your CI produces, *allow* the `release-pr` label-bearing PR to bypass any "require linear history" rules if those are enabled.

## Files in this example

| File | Destination in consumer repo | Purpose |
|---|---|---|
| [`workflows/release-pr.yml`](./workflows/release-pr.yml) | `.github/workflows/release-pr.yml` | Open/update the release PR on every `main` push. |
| [`workflows/release-tag.yml`](./workflows/release-tag.yml) | `.github/workflows/release-tag.yml` | Push the annotated tag when the release PR merges. |
| [`workflows/release.yml`](./workflows/release.yml) | `.github/workflows/release.yml` | Run `goreleaser` on tag push: build, sign, publish. |
| [`.goreleaser.example.yaml`](./.goreleaser.example.yaml) | `.goreleaser.yaml` | `goreleaser` configuration: builds, archives, SBOMs, `cosign`, Homebrew cask, release notes. |

Copy them, rename the example file to `.goreleaser.yaml`, and replace the placeholders marked `<<<REPLACE>>>` with your project's values:

- Project name, binary name, Go module path
- Homebrew tap owner/name + project homepage, description, license
- The `on_macos`/`postflight` block if you don't ship a macOS binary

## Step-by-step: cutting your first release

1. **Install the GitHub App** on the repo (and on the Homebrew tap repo if you're using one).
2. **Seed the manifest** — commit `.release-please-manifest.json` with `{ ".": "0.1.0" }` (or your desired starting version).
3. **Drop in the three workflows** and `.goreleaser.yaml`. Commit with a Conventional Commit subject like `ci(release): bootstrap goreleaser + skaphos/actions pipeline`.
4. **Push to main.** `release-pr` fires and opens PR #N — `release/v0.1.1` — changing only the manifest. Label `release-pr`.
5. **Review the PR.** The diff is one file; the PR body lists every commit since the last tag. Edit the manifest in the PR if you want a different version than `svu` computed.
6. **Merge the PR.** `release-tag` fires on `pull_request: closed` with `merged=true` and the `release-pr` label. It reads `0.1.1` from the manifest and pushes `v0.1.1` as an annotated tag.
7. **Watch `release.yml`** — `goreleaser` builds binaries, generates SBOMs, signs checksums with `cosign`, creates the GitHub release, and publishes the Homebrew cask.
8. **Verify:**
   - `gh release view v0.1.1 --json assets | jq '.assets[].name'` — should list six archives, `checksums.txt`, `checksums.txt.sigstore.json`, a `.sbom.json` per archive, and the attestation bundles.
   - `cosign verify-blob --bundle checksums.txt.sigstore.json --certificate-identity-regexp 'https://github.com/<owner>/<repo>/.github/workflows/release.yml@' --certificate-oidc-issuer https://token.actions.githubusercontent.com checksums.txt`.
   - `brew install <org>/<tap>/<name>` — should fetch the new version.

## Customizing the pipeline

### Force a major / minor bump on a specific release

Add `bump: minor` (or `major` / `patch`) to the `release-pr` step. Leave it on `auto` most of the time; flip it for one release cycle when you know Conventional Commit history doesn't reflect the intended bump.

```yaml
- uses: skaphos/actions/release-pr@v1
  with:
    token: ${{ steps.app-token.outputs.token }}
    bump: minor
```

### Run the release PR behind a protected maintenance branch

Change the `on:` trigger in `release-pr.yml` to `branches: [release/1.x]` and the `base:` of the generated PR by setting a `target-branch:` input (on the roadmap — for v1, the PR always targets the default branch).

### Ship without Homebrew

Delete the `homebrew_casks:` block from `.goreleaser.yaml`, delete the "Mint Homebrew app token" step from `release.yml`, and remove the `HOMEBREW_TAP_GITHUB_TOKEN` env var from the `goreleaser/goreleaser-action` step.

### Ship without SBOMs or cosign

Delete the `sboms:` and `signs:` blocks from `.goreleaser.yaml` and the corresponding installer steps (`anchore/sbom-action/download-syft`, `sigstore/cosign-installer`) from `release.yml`. We don't recommend this — both are cheap, declarative, and meaningfully raise the bar for supply-chain auditability.

## What happens on edge cases

- **Second push to main while the release PR is still open.** `release-pr` recomputes the next version, force-pushes the branch, and the existing PR updates in place. The manifest version may change if the newly added commits bumped the inferred kind (e.g. a `feat:` landed on top of a `fix:`-only history and bumped minor instead of patch). The PR body refreshes to show the new commit list.
- **Manual edit to the manifest in the release PR.** `release-tag` uses whatever version is in the manifest at merge time. If you hand-edited `0.1.1` to `0.2.0`, the tag pushed is `v0.2.0`. `svu`'s computation is advisory, not authoritative.
- **First release on an empty repo.** If no `v*` tag exists, the last-tag check treats it as `v0.0.0` and `svu next` computes `v0.0.1` by default (it will bump major on `!` / `BREAKING CHANGE:`). Manually seed the manifest to your desired starting version and merge that seed as a regular non-release PR, then let `release-pr` take over from the next push.
- **Tag already exists.** `release-tag` is idempotent: if the tag is already at the expected SHA, it exits cleanly with `pushed=false`. If it's at a different SHA, it errors — that's a state you need to resolve manually.
- **Immutable releases enabled at the org level.** `goreleaser` creates the release; `release-tag` only pushes the tag. The tag-push → `release.yml` → `goreleaser` path never has a pre-existing release object to collide with, so immutable-releases enforcement doesn't break anything here.

## Further reading

- [`skaphos/repokeeper` ADR-0007](https://github.com/skaphos/repokeeper/blob/main/docs/adr/0007-release-binaries-and-homebrew.md) — why we landed on `goreleaser` owns the release + a separate PR gate, rather than `brews:`, `svu` on every merge, or relaxed immutable-release enforcement.
- [`goreleaser` docs: homebrew_casks](https://goreleaser.com/customization/homebrew_casks/)
- [`cosign` keyless signing](https://docs.sigstore.dev/cosign/signing/signing_with_blobs/)
- [`actions/attest-build-provenance`](https://github.com/actions/attest-build-provenance)
