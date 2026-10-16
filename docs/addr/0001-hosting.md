# ADR 0001 – Hosting on the school CloudStack, production to GCP later

- Status: Accepted
- Date: 15-10-2026
- Deciders: infra team (Mattis, Maxim, Jasper)

## Context

We need dev, staging and production environments that run the OME stack (INN2, FastAPI,
frontend) with Docker Compose. Available: the school's Apache CloudStack, Google Cloud
(about €50 credits), AWS (Academy credits) and Azure.

INN2 needs a persistent process and raw TCP port 119, which rules out serverless platforms
such as Cloud Run.

## Decision

- **For now all three environments run on the school CloudStack**, one VM each
  (Rocky Linux 10, 2 vCPU, 2 GB RAM, 30 GB disk).
- **Production moves to GCP later**, once it must be reachable for ISKME.

## Consequences

- No cost, and the team already knows CloudStack.
- VMs have private addresses (10.129.80.0/24): reachable only from the school network/VPN.
  ISKME cannot see the environments yet.
- The school states that VMs do not run continuously. Every service therefore starts
  automatically after a reboot (`systemctl enable docker`, `restart: unless-stopped`).
  This is also the main argument to move production to GCP.
- Because the servers are configured with Ansible, moving production to GCP means
  creating a VM with the same cloud-init and changing one IP in the inventory.

## Alternatives considered

| Option | Why not (now) |
|---|---|
| GCP for everything | €50 covers roughly one small VM for two months, not three |
| AWS Academy | Lab sessions stop after a few hours: unusable for always-on environments |
| Azure | No advantage over GCP, no team experience |
| Kubernetes | Too much complexity for one week and a team new to it |

## Open questions

- Helpdesk (helpdesk.ikdoeict.be): when are VMs shut down, and can project VMs keep running?
- ISKME: do they need access to staging/production, and do they have a hosting preference?