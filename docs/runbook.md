# Runbook

Last updated: 17-10-2026. Run Ansible from `ome-infra/ansible`. Without `-i`, commands target **dev**.

## 1. Setup on your laptop (Linux or WSL)

```bash
git clone git@github.com:odisee-ome-2627/ome-infra.git
cd ome-infra/ansible
ansible-galaxy collection install -r requirements.yml
```

- `~/.ssh/config`: `Host 10.129.80.*` with `User ansible`, `IdentityFile ~/.ssh/<your-key>`, `IdentitiesOnly yes`
- Your public key on the servers: #63
- Vault password from the team password manager (#99) in `~/.ansible/ome-vault-password` (`chmod 600`)
- Keep the repo in your WSL home, not on `/mnt/c` (Ansible ignores `ansible.cfg` there)

## 2. Add a server

1. CloudStack: Rocky Linux 10, `ICT-Linux-Server`, `ICT-Linux-Root` 30 GB, user data **`ome-bootstrap`**
2. Accept the host key once: `ssh ansible@<ip> 'sudo -n true && echo OK'`
3. IP in `inventories/<env>/hosts.yml`, secrets in `vault.yml` (section 5)
4. `ansible-playbook -i inventories/<env> playbooks/site.yml` (twice: second run `changed=0`)
5. Add the frontend and API monitors in Uptime Kuma

The images for `<env>` must exist first (one merge into that branch). The first run may reboot the VM (ADR 0007).

## 3. Deploy and roll back

| Target | How |
|---|---|
| dev | merge a PR into `main` |
| staging | PR `main → staging`, approval, merge commit |
| production | PR `staging → production`, approval, merge commit |

After `publish-images` is green, Watchtower deploys within 5 minutes. Follow or speed it up:

```bash
ssh ansible@<ip> 'sudo docker logs -f --since 10m ome-watchtower-1'
ssh ansible@<ip> 'sudo docker run --rm -v /var/run/docker.sock:/var/run/docker.sock nickfedor/watchtower:1.19.0 --run-once --label-enable --cleanup'
```

Infrastructure changes are applied with `ansible-playbook`. If a fork change affects how the
images start (Dockerfile, entrypoint), update the compose template **before** promoting beyond dev.

**Roll back:** revert on the branch through a PR. Emergency: stop Watchtower, retag a previous
`<env>-<sha>` image as `<env>`, `docker compose up -d <service>`, document it afterwards.

## 4. Status, logs, monitoring

```bash
sudo docker compose -f /opt/ome/compose.yml ps
sudo docker compose -f /opt/ome/compose.yml logs --tail 50 <service>
```

- Uptime Kuma: `http://10.129.80.47:3001/`, alerts in Discord `#ome-alerts` ([monitoring.md](./monitoring.md))
- Update Kuma only: `ansible-playbook -i inventories/staging playbooks/monitoring.yml`
- API docs (not via nginx, #19): `ssh -L 8000:172.21.0.3:8000 ansible@<ip>` → `http://localhost:8000/docs`

## 5. Secrets (Ansible Vault)

```bash
cp inventories/vault.example.yml inventories/<env>/group_vars/all/vault.yml
openssl rand -hex 20                                       # one per password
ansible-vault encrypt inventories/<env>/group_vars/all/vault.yml
```

Edit with `ansible-vault edit`, then run the playbook (containers are recreated, ADR 0009).
Before a commit the file must start with `$ANSIBLE_VAULT;1.1;AES256`. Never paste vault content in a chat.

## 6. Troubleshooting

| Symptom | Fix |
|---|---|
| `Host key verification failed` | `ssh-keygen -R <ip>`, then `ssh ansible@<ip>` once |
| `Permission denied (publickey)` | SSH config with `IdentitiesOnly yes` |
| Docker: `missing kernel module` | handled by role `docker` (reboot) |
| Playbook: `compose.yml: ... false schema` | typo in a key of `compose.yml.j2` |
| Playbook: image `... not found` | merge into that branch first |
| API `Restarting`: `Config not found at /run/secrets/ome_config` | the `ome_config` secret is missing in the compose template (ADR 0009) |
| API calls return 502 | `docker compose logs fastapi-server` |
| Watchtower: `Failed to retrieve image info` for NNTP | harmless after an NNTP rebuild (#30) |

## 7. Known limitations

One node per environment (#60) · no NNTP volume (#30) · team SSH keys (#63) ·
no DNS/HTTPS (#108, #109) · VMs by hand (#104) · production on the school cloud (#107).
