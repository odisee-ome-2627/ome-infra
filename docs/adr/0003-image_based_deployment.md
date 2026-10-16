# ADR 0003 – Image-based deployment: GHCR + Watchtower

- Status: Accepted
- Date: 16-10-2026
- Deciders: infra team

## Context

A merge into an environment branch must reach its server automatically. The VMs have private
IPs behind the school network, so GitHub cannot connect to them.

## Decision

1. The `publish-images` workflow in the fork builds `ome-api` and `ome-frontend` on every
   push to `main`, `staging` or `production` and pushes them to GitHub Container Registry
   (`ghcr.io/odisee-ome-2627/...`) with two tags: `<env>` and `<env>-<short-sha>`.
2. Each server runs **Watchtower**, which checks every 5 minutes whether a new image exists
   for its tag and recreates only the labelled containers (API, frontend).
3. We use the maintained fork **`nickfedor/watchtower`**, pinned to a version, because
   `containrrr/watchtower` was archived in December 2025.

Details of the workflow:

- Permissions: `contents: read`, `packages: write`, using the built-in `GITHUB_TOKEN`.
  No personal tokens or stored secrets.
- All third-party actions are pinned to a commit SHA (enforced by zizmor in pre-commit).
- Production builds are never cancelled (`cancel-in-progress` only for `main`/`staging`).
- The job only runs when the repository owner is `odisee-ome-2627`, so it does nothing if
  the file ever ends up in ISKME's repository.

## Consequences

- Servers do not build anything except the NNTP image (see ADR 0006). Deploys are fast.
- The servers only make outgoing connections (pull). Nothing has to reach them from the internet.
- Delay between merge and running: build time + up to 5 minutes.
- Rollback: revert on the branch, or pin a previous `<env>-<sha>` tag.
- Watchtower does not roll back automatically when a new image is broken.
- Each environment builds its own image from its own branch. Promoting the exact same image
  (dev → staging → production) is a planned improvement.

## Alternatives considered

| Option | Why not |
|---|---|
| GitHub Actions pushes via SSH | GitHub cannot reach the private VMs; requires SSH keys in GitHub |
| Self-hosted runner on the VM | GitHub advises against it on public repos: fork PRs could run code on our server |
| Pull agent script (git pull + build on VM) | Every VM builds itself: slow, needs build resources on 2 GB VMs |
| `containrrr/watchtower` | Archived, no security fixes |
