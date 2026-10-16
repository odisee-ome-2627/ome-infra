# Runbook

Last updated: 16-10-2026. All Ansible commands run from `ome-infra/ansible`.

## 1. One-time setup on your laptop (Linux or WSL)

```bash
sudo apt install -y ansible git            # or: pipx install ansible-core
git clone git@github.com:odisee-ome-2627/ome-infra.git
cd ome-infra/ansible
ansible-galaxy collection install -r requirements.yml
```

`~/.ssh/config` (your own key, never in the repository):

```
Host 10.129.80.*
    IdentityFile ~/.ssh/<your-key>
    IdentitiesOnly yes
```

Vault password (ask an infra team member through a private channel, never Git or a public channel):

```bash
mkdir -p ~/.ansible
nano ~/.ansible/ome-vault-password     # paste the password, one line
chmod 600 ~/.ansible/ome-vault-password
```

> Do not keep the repository on the Windows drive (`/mnt/c/...`): Ansible ignores
> `ansible.cfg` in world-writable directories. Keep it in your WSL home and edit it
> with the VS Code "WSL" extension.

## 2. Add a new server

1. **CloudStack → Compute → Instances → Add Instance**
   - Template: Rocky Linux 10
   - Compute offering: `ICT-Linux-Server` (2 vCPU, 2 GB)
   - Root disk: `ICT-Linux-Root`, 30 GB
   - Network: the team network
   - SSH key pair: the team key
   - Advanced → User data: **`ome-bootstrap`**
   - Name: `ome-<environment>`, group `ome`
2. Wait until *Running* + 1 minute. Note the IP.
3. Accept the host key once (Ansible cannot answer the prompt):
   ```bash
   ssh ansible@<ip> 'sudo -n true && echo sudo OK'
   ```
4. Put the IP in `inventories/<env>/hosts.yml`.
5. Create the secrets if this environment has none yet (section 6).
6. Configure it:
   ```bash
   ansible-playbook -i inventories/<env> playbooks/site.yml
   ansible-playbook -i inventories/<env> playbooks/site.yml   # must report changed=0
   ```
   The first run may reboot the VM once (kernel modules for Docker, see ADR 0007).
7. Open `http://<ip>/imls/` from the school network.

## 3. Deploy a change

| Target | How |
|---|---|
| dev | Merge a PR into `main`. |
| staging | Open a PR `main → staging`, get an approval, merge (merge commit). |
| production | Open a PR `staging → production`, get an approval, merge (merge commit). |

Then:

1. GitHub → Actions → `publish-images`: wait until green.
2. Within 5 minutes Watchtower replaces the containers. Follow it on the server:
   ```bash
   ssh ansible@<ip> 'sudo docker logs -f ome-watchtower-1'
   ```

Infrastructure changes (this repository) are applied with `ansible-playbook`, not by Watchtower.

## 4. Roll back

**Preferred:** revert the faulty commit on the environment branch through a PR. CI publishes
a new image and Watchtower deploys it.

**Emergency (bypasses Git, document it afterwards):** run a previous image explicitly.

```bash
# Find tags: GitHub → org → Packages → ome-api → versions (e.g. dev-a1b2c3d)
ssh ansible@<ip>
sudo docker compose -f /opt/ome/compose.yml stop watchtower
sudo docker pull ghcr.io/odisee-ome-2627/ome-api:dev-a1b2c3d
sudo docker tag  ghcr.io/odisee-ome-2627/ome-api:dev-a1b2c3d ghcr.io/odisee-ome-2627/ome-api:dev
sudo docker compose -f /opt/ome/compose.yml up -d fastapi-server
# Restart Watchtower only once the branch is fixed:
sudo docker compose -f /opt/ome/compose.yml start watchtower
```

## 5. Logs and status

```bash
ssh ansible@<ip>
sudo docker compose -f /opt/ome/compose.yml ps
sudo docker compose -f /opt/ome/compose.yml logs -f fastapi-server
sudo docker compose -f /opt/ome/compose.yml logs -f nntp-server
sudo journalctl -u docker -b --no-pager | tail -n 50
```

From the laptop for all servers of an environment:

```bash
ansible -i inventories/<env> all -m command -a 'docker compose -f /opt/ome/compose.yml ps' --become
```

## 6. Secrets (Ansible Vault)

Create the secrets of an environment:

```bash
cp inventories/vault.example.yml inventories/<env>/group_vars/all/vault.yml
openssl rand -base64 24      # run twice, paste the values into the file
ansible-vault encrypt inventories/<env>/group_vars/all/vault.yml
```

| Task | Command |
|---|---|
| View | `ansible-vault view inventories/<env>/group_vars/all/vault.yml` |
| Edit | `ansible-vault edit inventories/<env>/group_vars/all/vault.yml` |
| Check before commit | the file must start with `$ANSIBLE_VAULT;1.1;AES256` |

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Host key verification failed` | first connection from this laptop, or VM recreated with the same IP | `ssh-keygen -R <ip>`, then `ssh ansible@<ip>` once |
| `Permission denied (publickey)` | SSH offers the wrong key | add the `Host 10.129.80.*` block with `IdentitiesOnly yes` |
| `No inventory was parsed` | not in the `ansible/` directory | `cd ome-infra/ansible` |
| Docker: `Extension addrtype ... missing kernel module` | Rocky 10 cloud image without `kernel-modules-extra` | handled by the `docker` role (install + reboot) |
| Docker: `Start request repeated too quickly` | systemd start limit after failures | `ansible all -m command -a 'systemctl reset-failed docker' --become` |
| CI: `npm error code ECONNRESET` | network glitch while downloading packages | Actions → Re-run failed jobs |
| Frontend loads, API calls return 502 | API container not running or wrong port | `docker compose logs fastapi-server` |
| Watchtower: `client version ... is too old` | Docker API version mismatch | update the pinned Watchtower version |

## 8. Known limitations and backlog

- Single node per environment; multi-node NNTP peering in staging (one node per team) is next.
- NNTP data is not on a persistent volume (same as upstream); it is lost when the container is recreated.
- Team SSH keys still to be managed by Ansible (issue #63).
- No swap on the 2 GB VMs yet (role `common`, needs SELinux label `swapfile_t`).
- Cockpit (port 9090) is allowed by the default firewalld zone and unused: remove it.
- Docker CE runs without SELinux integration (`selinux-enabled: false`).
- No HTTPS yet (reverse proxy such as Caddy).
- Production to GCP with a budget alert.
- Terraform for VM creation.
- Promote the same image through the environments instead of building per branch.
