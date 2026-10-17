# Findings in the upstream code (to discuss with ISKME)

Last updated: 17-10-2026. Found while deploying the OME stack outside a developer laptop.
Each open item will be proposed to ISKME as an issue or PR, branched from `upstream/main`,
once we know where they want them (ask Christian).

| # | Finding | Impact | Status in our fork | Proposal for ISKME |
|---|---|---|---|---|
| 1 | `Dockerfile-api` does not copy `server/`, `static/`, `templates/` (compose mounts them as volumes) | The API image only runs next to the source tree | Workaround: CI appends `COPY` lines (#106) | Add the `COPY` lines |
| 2 | API image starts with `fastapi dev` (auto-reload) | Not suited for deployment | Workaround: compose runs `fastapi run` | `fastapi run` in the image, `dev` only in compose |
| 3 | Dockerfiles copied `config.json` (with passwords) into the images | Credentials in every image | **Fixed** in fork PR #80: config mounted at start as `ome_config` secret (ADR 0009) | Offer PR #80 upstream |
| 4 | Upstream CI never builds the Docker images | Dockerfile breakage goes unnoticed | Our `publish-images` builds them | Build-only job on PRs |
| 5 | `front-end` CI job ends lint and tests with `\|\| true`, no build | The check is always green | Tracked in #89 | Remove `\|\| true`, add `npm run build:prod` |
| 6 | Frontend on Node 18 (end of life), deprecated npm packages | Security and maintenance risk | Tracked in #69 | Upgrade to a supported Node LTS |
| 7 | `docker-compose-two-hosts.yml` refers to removed files | Confusing for new contributors | — | Remove or update |
| 8 | Landing page links and API docs (`/docs`, `/channels`) are not forwarded by nginx | 404 on the servers; Swagger only via SSH tunnel | Tracked in #19 | Forward these routes, or link to `/api/...` |
| 9 | reCAPTCHA site key only valid for ISKME's domain | "Invalid domain for site key" on every other node | — | Site key from config, per node |
| 10 | No health endpoint | Monitoring must use `/api/list` (also tests NNTP) | — | Add `/api/health` |
| 11 | `static/index.html` had no `<meta charset>` | Characters such as "–" shown as "â€“" | **Fixed** in fork PR #84 | Offer the one-line fix |