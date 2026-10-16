# Architecture

Last updated: 16-10-2026

## Overview

Three environments, each on its own VM on the school's Apache CloudStack. Code flows
through the branches of our fork; every environment follows one branch.

```mermaid
flowchart LR
  subgraph GitHub["GitHub (odisee-ome-2627)"]
    F[feature/*] -->|PR + review + CI| M[main]
    M -->|PR, promotion-guard| S[staging]
    S -->|PR, promotion-guard| P[production]
    CI[publish-images workflow]
    GHCR[(GHCR<br/>ome-api, ome-frontend<br/>tags: dev / staging / production)]
    M & S & P -->|push| CI -->|build + push| GHCR
    INFRA[ome-infra<br/>Ansible + cloud-init + docs]
  end

  subgraph CloudStack["School CloudStack (10.129.80.0/24)"]
    DEV[ome-development<br/>tag dev]
    STG[ome-staging<br/>tag staging]
    PRD[ome-production<br/>tag production]
  end

  LAPTOP[Team laptop<br/>ansible-playbook] -->|SSH, configure| DEV & STG & PRD
  INFRA -.-> LAPTOP
  DEV & STG & PRD -->|Watchtower pulls new image| GHCR
```

**Who does what**

| Component | Responsibility | Runs when |
|---|---|---|
| `publish-images` (GitHub Actions, fork) | Builds the API and frontend images, pushes them to GHCR | push to `main`, `staging`, `production` |
| `promotion-guard` (GitHub Actions, fork) | Only allows `main → staging` and `staging → production` PRs | PR into `staging` / `production` |
| cloud-init | Creates the `ansible` user on a new VM | first boot only |
| Ansible (`ome-infra`) | Configures the server: packages, firewall, Docker, app, secrets | when infrastructure changes |
| Watchtower (on each VM) | Replaces API/frontend containers when a new image appears for its tag | every 5 minutes |

## Inside one server

```mermaid
flowchart TB
  USER[Browser] -->|HTTP :80| FE

  subgraph VM["VM (Rocky Linux 10, firewalld, Docker CE)"]
    subgraph NET["Docker network ome_net 172.21.0.0/24"]
      FE[imls-react<br/>nginx + React build<br/>172.21.0.4]
      API[fastapi-server<br/>FastAPI :8000<br/>172.21.0.3]
      NNTP[nntp-server<br/>INN2 :119<br/>172.21.0.2]
    end
    WT[watchtower]
    ENV[/opt/ome/api.env<br/>secrets, mode 0600/]
    SRC[/opt/ome/src<br/>fork checkout + config.json/]
  end

  FE -->|/api/ proxy| API
  API -->|NNTP + auth| NNTP
  ENV -.->|mounted read-only as /app/.env| API
  SRC -.->|docker build Dockerfile-nntp| NNTP
  WT -.->|updates labelled containers| FE & API
  GHCR[(GHCR)] -.-> WT
```

### Ports

| Port | Where | Reachable from |
|---|---|---|
| 22/tcp | VM | school network (SSH, Ansible) |
| 80/tcp | `imls-react` | school network (the application) |
| 8000/tcp | `fastapi-server` | Docker network only (through nginx `/api/`) |
| 119/tcp | `nntp-server` | Docker network only. **Never published**: NNTP peering has no authentication. |

Docker publishes ports with its own firewall rules, bypassing firewalld. That is why only the
frontend has a `ports:` entry in the compose file; the firewalld rule for port 80 documents the intent.

### Where the configuration comes from

| Item | Source | Contains secrets? |
|---|---|---|
| API and frontend images | GHCR, built in CI with `config.json.example` | No (example values only) |
| `/opt/ome/api.env` | Ansible template + Vault | Yes, mode 0600, mounted over `/app/.env` |
| NNTP image | built on the VM by Ansible from the fork + rendered `config.json` | Yes, never pushed to a registry |
| Vault password | each admin's `~/.ansible/ome-vault-pass` | — never in Git |

## Deployment flow (dev)

```mermaid
sequenceDiagram
  actor Dev as Developer
  participant GH as GitHub (fork)
  participant CI as publish-images
  participant R as GHCR
  participant WT as Watchtower (ome-development)

  Dev->>GH: PR feature → main
  GH->>GH: CI checks + 1 approval (ruleset)
  Dev->>GH: merge
  GH->>CI: push event on main
  CI->>R: push ome-api:dev, ome-frontend:dev (+ dev-<sha>)
  loop every 5 minutes
    WT->>R: new digest for :dev?
  end
  WT->>WT: pull, recreate container, remove old image
```

Promotion to staging or production is the same flow, started by a PR `main → staging`
or `staging → production`.
