# ADR 0009 – Configuration mounted as a Docker secret at runtime

- Status: Accepted (supersedes ADR 0006)
- Date: 17-10-2026
- Deciders: infra team

## Context

Fork PR #80 (issue #72) removed all credentials from the Docker images. Both images now
render their configuration **at container start** from `config.json`, which they expect at
`/run/secrets/ome_config` (`server_config/*/docker-entrypoint.sh`). Without that file the
container exits with `Config not found at /run/secrets/ome_config`.

Before this change, ADR 0006 worked around the credentials in the images: the API was built
with `config.json.example` and `api.env` was mounted over `/app/.env`; the NNTP image was
built on the server with the real `config.json` inside.

On 16-10-2026 the new images reached dev before the infrastructure followed, and the API
kept restarting (see `docs/incidents/2026-10-16-config-secret.md`).

## Decision

- Ansible renders `config.json` per environment from Ansible Vault
  (`/opt/ome/src/config.json`, mode 0600), as before.
- The compose file mounts it as the Docker secret **`ome_config`** in `nntp-server` and
  `fastapi-server`. Docker places it read-only at `/run/secrets/ome_config`.
- `INN_SERVER_NAME` is passed as an environment variable (no longer a build argument).
- When `config.json` changes, Ansible recreates the containers, because they only read it at start.

## Consequences

- **No image contains credentials anymore**, including NNTP. The NNTP image could now be
  published to a registry like the others.
- One configuration file for API and NNTP instead of two (`api.env` is no longer used and
  will be removed, issue #93).
- The same compose secret is used in the upstream `docker-compose.yml`, so local and server
  setups work the same way.
- Infrastructure must follow changes in how the images start. Changes to Dockerfiles or
  entrypoints need an infra review before they are promoted beyond dev.

## Alternatives considered

- **Keep mounting `api.env` over `/app/.env`**: the entrypoint now writes `/app/.env` itself;
  a read-only mount would make it fail.
- **Environment variables for every setting**: `config.json` contains nested data (peers,
  users) that the templates expect as JSON.
