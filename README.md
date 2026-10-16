# ome-infra

Infrastructure as Code for the Open Metadata Exchange (OME) environments of team
`odisee-ome-2627` (Odisee Agile Team Project, client ISKME).

The application code lives in our fork
[`odisee-ome-2627/Open-Metadata-Exchange`](https://github.com/odisee-ome-2627/Open-Metadata-Exchange).
This repository only holds how we **host** it, so nothing here ever ends up in ISKME's repository.

## What is in here

| Path | Purpose |
|---|---|
| `cloud-init/bootstrap.yml` | First-boot config for new VMs: key-only `ansible` user with sudo |
| `ansible/` | Configuration of every server: roles `common`, `docker`, `ome` |
| `ansible/inventories/<env>/` | Servers and secrets per environment (dev, staging, production) |
| `docs/architecture.md` | How the pieces fit together (Mermaid diagrams) |
| `docs/adr/` | Architecture Decision Records |
| `docs/runbook.md` | How to add a server, deploy, roll back, read logs |
| `docs/upstream-findings.md` | Issues found in the upstream code, to report to ISKME |

## Environments

| Environment | Server | Follows branch of the fork | Image tag |
|---|---|---|---|
| dev | `ome-development` | `main` | `dev` |
| staging | `ome-staging` | `staging` | `staging` |
| production | `ome-production` | `production` | `production` |

## Quick start

Requirements: Linux or WSL with `ansible-core` ≥ 2.16, SSH access to the school CloudStack,
your key registered on the servers, and the Vault password in `~/.ansible/ome-vault-pass`.

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible all -m ping                                           # dev (default)
ansible-playbook playbooks/site.yml                           # configure dev
ansible-playbook -i inventories/staging playbooks/site.yml    # configure staging
```

Running the playbook a second time must report `changed=0`.

See [`docs/runbook.md`](docs/runbook.md) for everything else.
