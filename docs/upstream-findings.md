# Findings in the upstream code (to discuss with ISKME)

Found while deploying the OME stack outside a developer laptop (15-10-2026).
Each item will be proposed to ISKME as an issue or PR, branched from `upstream/main`.

| # | Finding | Impact | Our workaround | Proposal |
|---|---|---|---|---|
| 1 | `Dockerfile-api` does not copy `server/`, `static/` or `templates/`; `docker-compose.yml` mounts them as volumes | The API image cannot run anywhere without the source tree | CI appends `COPY` lines before building | Add the `COPY` lines to `Dockerfile-api` |
| 2 | The API container starts with `fastapi dev` (auto-reload, development mode) | Not suited for deployment | We override the command with `fastapi run` | Use `fastapi run` in the image, `fastapi dev` only in compose for development |
| 3 | `Dockerfile-api` and `Dockerfile-nntp` copy `config.json` with passwords into the image | Credentials in every image | API built with `config.json.example`, real `.env` mounted at runtime | Team issue #72 |
| 4 | Upstream CI never builds the Docker images | Dockerfile breakage goes unnoticed | Our `publish-images` workflow builds them | Add a build-only job on PRs |
| 5 | The `front-end` CI job ends lint and unit tests with `\|\| true` and does not build | The check is always green | — | Existing TODO in `build.yml`: remove `\|\| true`, add `npm run build:prod` |
| 6 | Frontend uses Node 18 (end of life) and many deprecated npm packages | Security and maintenance risk | — | Upgrade to a supported Node LTS |
| 7 | `docker-compose-two-hosts.yml` refers to files that no longer exist (`Dockerfile`, `fe2/`) | Confusing for new contributors | — | Remove or update it (ask Christian first) |