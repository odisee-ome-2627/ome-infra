# ADR 0008 – Monitoring with Uptime Kuma, notifications in Discord

- Status: Accepted
- Date: 16-10-2026
- Deciders: infra team

## Context

Failing deployments and unreachable environments must be reported automatically. The team
already uses Discord. The VMs have private IPs, so external (SaaS) uptime checkers cannot
reach them: the checker must run inside the school network.

## Decision

- **Uptime Kuma** (pinned version) on `ome-staging`, deployed by the Ansible role
  `monitoring` (own playbook `playbooks/monitoring.yml`, also imported by `site.yml`),
  checks the frontend and API of every environment every 60 seconds.
- Not on production: a monitor on the server it watches goes down with it. Not on dev:
  dev is allowed to break.
- Uptime Kuma sends **down/up alerts to Discord** (`#ome-alerts`). The webhook is kept in
  Uptime Kuma's data volume, never in Git.
- **CI failures** are reported by GitHub itself (red check, e-mail to the author).
- **Deployments** are logged by Watchtower on each server; a broken deployment shows up as a
  red monitor within a minute. Watchtower → Discord is the next step.

## Consequences

- One place (`#ome-alerts`) shows the outages of all environments.
- Monitors are configured once in the Uptime Kuma UI, not in Git. Backlog: Gatus, whose
  checks are a YAML file that Ansible can render from the inventories.
- If the staging VM is down, there is no monitoring (and no alert about staging itself).
  Acceptable for now; a dedicated monitoring host (or GCP) is on the backlog.

## Alternatives considered

- **Prometheus + Grafana + Alertmanager**: the standard for metrics, but four extra
  components for three small servers. Backlog, together with central logging (Loki).
- **External uptime services** (UptimeRobot, Better Stack): cannot reach private IPs.
- **E-mail only**: slower to notice than the team chat.
