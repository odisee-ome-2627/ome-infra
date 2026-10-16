# ADR 0004 – Configuration with Ansible, bootstrap with cloud-init

- Status: Accepted
- Date: 15-10-2026
- Deciders: infra team

## Context

We want identical servers for every environment, reproducible from code, where adding a
server only means adding its IP. The Rocky Linux template's default user needs an unknown
sudo password.

## Decision

- **cloud-init** (`cloud-init/bootstrap.yml`, registered in CloudStack's User Data Library
  as `ome-bootstrap`) runs once at first boot and creates the user `ansible`:
  SSH key only (`lock_passwd`), passwordless sudo, and SSH password login disabled.
- **Ansible** configures everything else with one playbook (`playbooks/site.yml`) and three roles:

  | Role | What it does |
  |---|---|
  | `common` | base packages, timezone, firewalld enabled with SSH allowed |
  | `docker` | Docker CE from the official repository (GPG fingerprint checked), `kernel-modules-extra` for Rocky 10, conditional reboot |
  | `ome` | fork checkout, secrets, NNTP build, compose stack with Watchtower, health checks |

- **One inventory per environment** (`inventories/dev|staging|production`). Only the IP in
  `hosts.yml`, `ome_env` in `vars.yml` and the secrets in `vault.yml` differ.
- `ansible.cfg` defaults to the **dev** inventory: staging and production need an explicit `-i`.
- Secrets are stored with **Ansible Vault**; the Vault password lives outside the repository.
- The playbook must be **idempotent**: a second run reports `changed=0`.

## Consequences

- A new server = VM with `ome-bootstrap` + one IP + one playbook run.
- Findings are fixed once in code, for every future server (example: the template ships
  with firewalld disabled; Rocky 10 cloud images lack `kernel-modules-extra`).
- Only the bootstrap key is set by cloud-init. Team keys will be managed by Ansible
  (`ansible.posix.authorized_key`, `exclusive: true`) so access stays in sync on every run
  (issue #63).
- Passwordless sudo is acceptable because the account has no password and only accepts
  SSH keys; the same model as the default users of AWS and GCP images.

## Alternatives considered

- **Manual configuration**: not reproducible, undocumented drift.
- **Waiting for the template's sudo password**: blocks the work and leaves a shared
  default password on every VM.
- **Terraform for VM creation**: planned; VMs are created manually for now and documented in the runbook.
