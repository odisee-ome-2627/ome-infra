# ome-infra

Infrastructure as Code for the Open Metadata Exchange (OME) environments of team
`odisee-ome-2627` (Odisee Agile Team Project, client ISKME).

The application code lives in our fork
[`odisee-ome-2627/Open-Metadata-Exchange`](https://github.com/odisee-ome-2627/Open-Metadata-Exchange).
This repository only holds how we **host** it, so nothing here ends up in ISKME's repository.

## Environments

| Environment | Server | Follows branch | Image tag | URL (school network) |
|---|---|---|---|---|
| dev | `ome-development` | `main` | `dev` | http://10.129.80.46/ |
| staging | `ome-staging` | `staging` | `staging` | http://10.129.80.47/ |
| production | `ome-production` | `production` | `production` | http://10.129.80.54/ |

Monitoring: Uptime Kuma on http://10.129.80.47:3001/, alerts in Discord `#ome-alerts`.

## What is in here

| Path | Purpose |
|---|---|
| `cloud-init/bootstrap.yml` | First boot of a new VM: key-only `ansible` user with sudo |
| `ansible/playbooks/site.yml` | Configures every server (roles `common`, `docker`, `ome`) and monitoring |
| `ansible/playbooks/monitoring.yml` | Uptime Kuma only (role `monitoring`, on staging) |
| `ansible/inventories/<env>/` | Servers and encrypted secrets per environment |
| `docs/architecture.md` | How the pieces fit together |
| `docs/runbook.md` | Setup, deploy, roll back, troubleshoot |
| `docs/monitoring.md` | Monitoring, alerts, logging |
| `docs/scalability.md` | What scaling means for OME |
| `docs/upstream-findings.md` | Issues found in ISKME's code |
| `docs/adr/` | Architecture Decision Records |
| `docs/incidents/` | Incident notes |

## Quick start

Requirements: Linux or WSL, `ansible-core` ≥ 2.16, school network or VPN, your SSH key on the
servers, and the Vault password in `~/.ansible/ome-vault-password`.

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible all -m ping                                          # dev (default)
ansible-playbook playbooks/site.yml                          # dev
ansible-playbook -i inventories/staging playbooks/site.yml   # staging
```

A second run must report `changed=0`. Everything else: [`docs/runbook.md`](./docs/runbook.md).
