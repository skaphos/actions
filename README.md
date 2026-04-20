# skaphos/actions

Reusable composite GitHub Actions for the `skaphos` organization's release automation.

## Available actions

| Action | Purpose |
|---|---|
| [`release-pr`](./release-pr) | Detect commits beyond the last `v*` tag, compute the next semver with [`svu`](https://github.com/caarlos0/svu), and open or update a release PR that bumps `.release-please-manifest.json`. |
| [`release-tag`](./release-tag) | On merge of a release PR, read the bumped manifest and push the annotated `vX.Y.Z` tag that triggers downstream release workflows (e.g. `goreleaser`). |

Together, the two actions replace `googleapis/release-please-action` with a thin, purpose-built gate: **one PR per release, merged when a human is ready, tag pushed automatically on merge**. No changelog generation (we defer to `goreleaser`), no release object creation (`goreleaser` again), no multi-package complexity — just the semver gate.

> **Scaffolding in progress.** The two `action.yml` files are landing in follow-up PRs. This README will be updated with full input/output reference and copy-paste consumer snippets once `v1.0.0` is cut.

## Versioning

- Pinned semver tags (`v1.0.0`, `v1.1.0`, …) — immutable, each corresponds to one commit.
- Floating major tag (`v1`) — reassigned on every release within that major, for consumers who want automatic non-breaking updates.

Consumers pin to `@v1` (convenience) or `@v1.2.3` (strict). Never pin to `main`.

## License

[MIT](./LICENSE).
