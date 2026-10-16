# ADR 0006 – Secrets stay out of published images

- Status: Accepted
- Date: 15-10-2026
- Deciders: infra team

## Context

Upstream `Dockerfile-api` and `Dockerfile-nntp` copy `config.json` (with NNTP passwords)
into the image at build time. Published images on GHCR are public: anyone could extract
the passwords. Team issue #72 works on removing credentials from the images upstream.

## Decision

- **API and frontend** are built in CI with `config.json.example` (example values only).
  The real API settings are rendered by Ansible to `/opt/ome/api.env` (mode 0600) and
  mounted read-only over `/app/.env`.
- **NNTP** is built on the server itself by Ansible, from the fork and a `config.json`
  rendered from Vault. That image is never pushed to a registry.
- Secrets are generated per environment (`openssl rand -base64 24`) and stored in
  `inventories/<env>/group_vars/all/vault.yml`, encrypted with Ansible Vault.

- The GHCR packages `ome-api` and `ome-frontend` are **public**. The project is open source
  (AGPL) and the images contain no secrets, so making them private would protect nothing
  while forcing every server and Watchtower to store a registry token (another secret that
  expires and is tied to one person's account).

## Consequences

- Public images can be shared freely; servers pull them without credentials.
- If the `.env` mount were missing, the API would run with example credentials and fail to
  authenticate against NNTP; the playbook's health check would catch that.
- Temporary workaround until issue #72 is merged upstream; then the build no longer needs
  `config.json` at all.
  