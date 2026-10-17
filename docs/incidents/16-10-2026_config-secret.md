# Incident 16-10-2026 – API on dev restarting after fork PR #80

- Environment: dev (`ome-development`)
- Duration: about 1 hour (10:15 – ±11:15)
- Impact: API on dev unavailable (502). Staging and production not affected.
- Detected by: Uptime Kuma, alert `dev-api went down` in Discord `#ome-alerts` within a minute

## What happened

1. Fork PR #80 (issue #72, *Remove credentials from Docker images*) was merged into `main`.
   The images no longer contain `config.json`; at start, an entrypoint script reads it from
   `/run/secrets/ome_config` and renders the configuration.
2. `publish-images` built `ome-api:dev`, Watchtower deployed it on dev.
3. The compose file on the server still mounted `api.env` over `/app/.env` and provided
   nothing at `/run/secrets/ome_config`. The API exited with
   `Config not found at /run/secrets/ome_config` and kept restarting; nginx returned 502.

## Why staging and production were safe

They follow their own branches. The change was only on `main`, so only dev received the new
image. This is the reason the environments are separated (ADR 0002).

## Fix

- Compose template in role `ome`: `config.json` (rendered from Vault) mounted as the Docker
  secret `ome_config` in API and NNTP, `INN_SERVER_NAME` as environment variable,
  containers recreated when `config.json` changes (issue #90, ADR 0009).
- Playbook run on dev: API green again, alert `dev-api is up` in Discord.

## What we learned

- Monitoring worked: the problem was reported automatically, before anyone noticed.
- A change in **how the images start** (Dockerfile, entrypoint) needs the infrastructure to
  follow. Such PRs now get an infra review before they are promoted beyond dev (runbook §3).
- The fix was first applied by hand on dev; it must always end up in a reviewed PR (#90).

## Follow-up

| Action | Issue |
|---|---|
| Put the fix in a reviewed PR | #90 |
| Roll out to staging, then production | #91, #92 |
| Remove the unused `api.env` flow | #93 |
| Update the docs (ADR 0009, architecture) | #97 |
