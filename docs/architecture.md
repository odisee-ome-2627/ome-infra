# Architecture

Last updated: 17-10-2026

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
    STG[ome-staging<br/>tag staging<br/>+ Uptime Kuma]
    PRD[ome-production<br/>tag production]
  end

  LAPTOP[Team laptop<br/>ansible-playbook] -->|SSH, configure| DEV & STG & PRD
  INFRA -.-> LAPTOP
  DEV & STG & PRD -->|Watchtower pulls new image| GHCR
  STG -.->|HTTP checks every 60 s| DEV & PRD
  STG -.->|down / up alerts| DISCORD[(Discord<br/>#ome-alerts)]
```

**Who does what**

| Component | Responsibility | Runs when |
|---|---|---|
| `publish-images` (GitHub Actions, fork) | Builds the API and frontend images, pushes them to GHCR | push to `main`, `staging`, `production` |
| `promotion-guard` (GitHub Actions, fork) | Only allows `main → staging` and `staging → production` PRs | PR into `staging` / `production` |
| cloud-init | Creates the `ansible` user on a new VM | first boot only |
| Ansible (`ome-infra`) | Configures the server: packages, firewall, Docker, config, app, monitoring | when infrastructure changes |
| Watchtower (on each VM) | Replaces API/frontend containers when a new image appears for its tag | every 5 minutes |
| Uptime Kuma (on `ome-staging`) | Checks frontend and API of every environment, alerts in Discord | every 60 seconds |

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
    CFG[/opt/ome/src/config.json<br/>rendered from Vault, mode 0600/]
  end

  FE -->|/api/ proxy| API
  API -->|NNTP + auth| NNTP
  CFG -.->|secret ome_config<br/>/run/secrets/ome_config| API & NNTP
  WT -.->|updates labelled containers| FE & API
  GHCR[(GHCR)] -.-> WT
```

At start, each container reads `/run/secrets/ome_config` with its entrypoint script: the API
renders its `.env`, NNTP renders the INN configuration. No image contains `config.json`
(ADR 0009).

### Ports

| Port | Where | Reachable from |
|---|---|---|
| 22/tcp | every VM | school network (SSH, Ansible) |
| 80/tcp | `imls-react` | school network (the application) |
| 3001/tcp | Uptime Kuma, `ome-staging` only | school network (monitoring dashboard) |
| 8000/tcp | `fastapi-server` | Docker network only (through nginx `/api/`) |
| 119/tcp | `nntp-server` | Docker network only. **Never published**: NNTP peering has no authentication. |

Docker publishes ports with its own firewall rules, bypassing firewalld. That is why only the
frontend (and Uptime Kuma) have a `ports:` entry; the firewalld rules document the intent.

### Where the configuration comes from

| Item | Source | Contains secrets? |
|---|---|---|
| API and frontend images | GHCR, built in CI from the fork | No |
| NNTP image | built on the VM by Ansible from the fork | No |
| `/opt/ome/src/config.json` | Ansible template + Vault, mode 0600 | Yes; mounted as the `ome_config` secret |
| Vault password | each admin's `~/.ansible/ome-vault-password` | never in Git |
| Discord webhook (alerts) | Uptime Kuma data volume on `ome-staging` | never in Git |

## Deployment flow (dev)

```mermaid
sequenceDiagram
  actor Dev as Developer
  participant GH as GitHub (fork)
  participant CI as publish-images
  participant R as GHCR
  participant WT as Watchtower (ome-development)

  Dev->>GH: PR feature → main
  GH->>GH: CI checks + approval (ruleset)
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

## More

- [`monitoring.md`](./monitoring.md): Uptime Kuma, alerts, logging
- [`scalability.md`](./scalability.md): what scaling concretely means for OME
- [`runbook.md`](./runbook.md): add a server, deploy, roll back, troubleshoot
- [`adr/`](./adr/): the decisions behind all of this