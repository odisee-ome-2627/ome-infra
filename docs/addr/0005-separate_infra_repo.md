# ADR 0005 – Separate infrastructure repository; CI workflows in the fork

- Status: Accepted
- Date: 15-10-2026
- Deciders: infra team

## Context

Our hosting (school CloudStack, Ansible, IPs) is specific to our team. ISKME's repository
must not receive it. At the same time, image builds must start when code changes, which
GitHub can only trigger from the repository that contains the code.

## Decision

- **`odisee-ome-2627/ome-infra`** (this repository) contains Ansible, cloud-init and the
  infrastructure documentation.
- **The fork** contains only two team workflows: `publish-images.yml` and `promotion-guard.yml`.
  Both jobs have `if: github.repository_owner == 'odisee-ome-2627'`, so they are skipped
  anywhere else.
- **Contributions to ISKME branch from `upstream/main`**, never from the fork's `main`, so
  our workflows never appear in an upstream PR:

  ```bash
  git remote add upstream git@github.com:ISKME/Open-Metadata-Exchange.git   # once
  git fetch upstream
  git switch -c fix/something upstream/main
  ```

## Consequences

- The fork stays close to upstream; PRs to ISKME stay clean.
- Two repositories to review. Both use rulesets (PR + approval, no bypass).
- The documentation of this repository is also published on the school GitLab
  (group `2627_ATP_ISKME`).
  