# ADR 0002 – Environments follow branches; promotion only through pull requests

- Status: Accepted
- Date: 15-10-2026
- Deciders: infra team

## Context

The team works in a fork of `ISKME/Open-Metadata-Exchange`. The fork's `main` is regularly
synced with ISKME's `main`. We need a predictable way to get changes into dev, staging and
production, without untested code reaching production.

## Decision

| Branch | Environment | How code gets in |
|---|---|---|
| `main` | dev | PR from a feature branch, CI green, 1 approval |
| `staging` | staging | PR **from `main` only** |
| `production` | production | PR **from `staging` only** |

Enforced with GitHub rulesets (no bypass list):

- `main`: PR required, 1 approval, approval of the most recent push by someone else,
  conversations resolved, required checks (`inn-docker_uv (cclauss/inn)`, `shellcheck`,
  `front-end (18.19.0)`, `build_docs`), no force push, no deletion.
  Merge methods: merge commit (needed for upstream syncs) and squash.
- `staging`, `production`: PR required, 1 approval, merge commit only, no force push,
  no deletion, required check `check-source-branch`.

The `promotion-guard` workflow provides `check-source-branch`: it fails unless the PR is
`main → staging` or `staging → production`, and it re-runs when a PR's base branch is edited.

## Consequences

- An upstream sync from ISKME lands in dev first, never directly in production.
- Every change in staging/production was tested on `main` first.
- Promotions use merge commits so the branches share history and do not conflict.
- ISKME's CI only runs for PRs into `main`, which is why `staging`/`production` rely on
  `check-source-branch` instead of the upstream checks.
- The fork's "Sync fork" button no longer works on `main`; upstream syncs are PRs from
  `ISKME:main`, merged with a merge commit.

## Alternatives considered

- **`main` = production**: every upstream sync would go live untested.
- **Tags/releases instead of branches**: more powerful, but harder to explain and operate
  for a new team. Possible later.
