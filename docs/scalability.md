# Scalability

Last updated: 16-10-2026

What "scaling" concretely means for the OME stack, what we can do today, and what each
step costs. OME has no central database: every node stores the metadata it exchanges in
its own NNTP server (INN2). That shapes everything below.

## The components and how each one scales

| Component | Holds state? | How it scales | Effort |
|---|---|---|---|
| `imls-react` (nginx + static React build) | No | More replicas or more servers; static files can also go to a CDN | Low |
| `fastapi-server` (API) | No (reads/writes NNTP, config from `.env`) | More replicas behind nginx; more Uvicorn workers per container | Low |
| `nntp-server` (INN2) | **Yes** (articles, history, overview) | Vertically (CPU, RAM, disk) per node; **horizontally by adding nodes that peer with each other** — the core design of OME | Medium |
| Watchtower, Ansible | No | Unchanged for 1 or 100 servers: every server pulls its own images, one playbook configures any number of hosts | None |

The API and frontend are **stateless**: all state lives in NNTP and in the per-server
`api.env`. Any number of copies can run side by side.

## Step 1 – Vertical: a bigger VM (minutes, no code change)

Today every environment runs on `ICT-Linux-Server` (2 vCPU, 2 GB RAM, 30 GB disk).

1. CloudStack → instance → **Stop** → **Change service offering** (more vCPU/RAM) → **Start**.
2. If articles need more space: **Resize volume** and grow the filesystem.
3. Nothing else changes: Docker and all containers start automatically after the reboot
   (`restart: unless-stopped`, Docker enabled, see ADR 0007).

On Google Cloud (planned for production) this is the same operation: change the machine type.

## Step 2 – More API capacity on the same VM (one line)

FastAPI runs one Uvicorn worker today. More workers use more CPU cores:

```yaml
command: ["uv", "run", "--env-file=.env", "fastapi", "run", "--workers=4", ...]
```

This is a change in `roles/ome/templates/compose.yml.j2` and one playbook run.
(Several API containers on one VM would also work, but the upstream compose file gives each
service a fixed IP address, so workers are the simpler first step.)

## Step 3 – Horizontal: more servers (one inventory line per server)

Because the API and frontend are stateless, a second application server is:

1. Create a VM with the `ome-bootstrap` user data (runbook §2).
2. Add one line to `inventories/<env>/hosts.yml`.
3. `ansible-playbook -i inventories/<env> playbooks/site.yml`.

A load balancer (for example HAProxy or a cloud load balancer on GCP) then spreads visitors
over the servers. Watchtower on each server keeps them on the same image tag.

## Step 4 – More NNTP nodes: how OME is meant to grow

OME is a **decentralised** network: each organisation runs its own node, and nodes exchange
metadata through NNTP peering (`peers` in `config.json`, variable `ome_nntp_peers` in our
role). More users or more organisations means **more nodes**, not one bigger central server.

- Reads are served by the local node, so load is spread over all nodes.
- A node that goes down does not stop the others; it catches up when it comes back.
- Planned: several peered nodes in staging (one per team), then production.

## Current limits (honest)

| Limit | Impact | Next step |
|---|---|---|
| One VM per environment | A VM failure takes that environment down | Second node + peering; production to GCP |
| School VMs do not run continuously | Availability outside school hours | Production to GCP (ADR 0001) |
| No load balancer yet | Horizontal scaling needs one | HAProxy role or GCP load balancer |
| NNTP storage on the root disk | Disk can fill up | Separate data volume, monitored (see monitoring) |
| Single Uvicorn worker | Uses one CPU core | `--workers` (step 2) |

## Summary

- **Today, in minutes:** bigger VM (vertical), no code change.
- **Today, one playbook run:** more workers or a second server (horizontal).
- **By design:** more peered NNTP nodes, which is exactly how OME is meant to grow.