# Neverness maintenance notes

Neverness is the maintained deployment/fork of `IanUtley/hex-server` used for the HEX PvE server. These notes exist to keep local changes, review findings, deployment assumptions, and future upstream integration decisions in the repository rather than only in chat history or on the running host.

## Branch model

- `main` tracks IanUtley's upstream `main` as closely as possible. Do not place Neverness-only fixes directly on `main`.
- `neverness` is the long-lived branch for code actually maintained for the Neverness deployment.
- Short-lived `fix/*`, `review/*`, or `setup/*` branches may be used for isolated work before it is merged into `neverness`.
- Deployment tags should identify known-good Neverness builds once the fork becomes the deployment source of truth.

The initial Neverness baseline is upstream commit `ee0f06b3ee04b4eb678fe9e464e4d207271f67da` (upstream 0.3.0-era rules-engine rewrite) plus two already-tested local fixes from the running server:

1. `3fd0d24` — `Fix Docker bootstrap SQLite self-lock`
2. `b8e9e9d` — `Fix unopened chest import during profile login`

Those two commit objects currently exist only in the CT103 checkout. Their exact diffs are documented in `CODE_REVIEW.md`, but they have not yet been imported into the GitHub `neverness` branch. Until that import is complete, treat the branch as the review/documentation source of truth rather than a deployment-ready replacement for the current CT103 checkout.

## Repository policy

The fork is the durable source of truth for Neverness code and documentation. The running CT103 checkout should eventually be deployed from a specific commit/tag in this repository rather than becoming an independent source of changes.

`main` is an upstream mirror, not the production branch. New upstream work should first be reviewed against `neverness`, classified, and deliberately merged or adapted. See [UPSTREAM_CHANGES.md](UPSTREAM_CHANGES.md).

No review finding should be patched directly during the discovery phase unless it is required to make the review possible. Findings are recorded first, then grouped into coherent repair batches after the review is complete.

## Documentation index

- [CODE_REVIEW.md](CODE_REVIEW.md) — review scope, confirmed findings, open investigations, and review progress.
- [KNOWN_ISSUES.md](KNOWN_ISSUES.md) — concise operational and user-visible issue register.
- [UPSTREAM_CHANGES.md](UPSTREAM_CHANGES.md) — upstream tracking and merge/adaptation log.
- [DEPLOYMENT.md](DEPLOYMENT.md) — deployment model and current container assumptions.
- [DATABASE.md](DATABASE.md) — SQLite architecture, schema ownership, and migration concerns.

## Review rule

Before changing a file for Neverness, inspect the exact version on the `neverness` branch (and the deployed revision when deployment state matters). Upstream code is reference material only. Patches should use exact-match safety checks and backups when applied to a live checkout; if the deployed code differs from the reviewed version, stop and reconcile the difference instead of forcing the patch.
