# Runbook

Last updated: 10-11-2026

How to set up, run and test OME nodes with this repository, **on your own servers**. Each chapter
is one task: follow it from top to bottom.

- Values in angle brackets are yours to fill in: `<server-ip>`, `<env>` (`dev`, `staging` or
  `production`), `<your-network>`.
- All Ansible commands run from `ome-infra/ansible`. Without `-i`, they target the inventory set in
  `ansible.cfg` (`inventories/dev`).
- The values of the Odisee environments (IP addresses, school cloud, monitoring) are in the
  [appendix](#appendix-odisee-environments), not in the chapters.

| Chapter | When you need it |
|---|---|
| [1. Prepare your laptop](#1-prepare-your-laptop) | Once, before anything else |
| [2. Make the repository yours](#2-make-the-repository-yours) | Once, before your first server |
| [3. Add a server](#3-add-a-server) | For every new node |
| [4. Peering between nodes](#4-peering-between-nodes) | To understand or change how nodes exchange articles |
| [5. Test peering](#5-test-peering) | After a change, or to check the network |
| [6. Test store and forward](#6-test-store-and-forward) | To prove that a node gets what it missed |
| [7. Updates and roll back](#7-updates-and-roll-back) | For every new version |
| [8. Status, logs and monitoring](#8-status-logs-and-monitoring) | When something looks wrong |
| [9. Secrets](#9-secrets) | When you create an environment or change a password |
| [10. Troubleshooting](#10-troubleshooting) | When a step fails |
| [11. Known limitations](#11-known-limitations) | Before you rely on something |
| [Appendix: Odisee environments](#appendix-odisee-environments) | Only for the Odisee team |

---

## 1. Prepare your laptop

You need:

- Linux or WSL (Windows Subsystem for Linux);
- `ansible-core` 2.16 or newer;
- network access to your servers on port 22 (SSH).

### 1.1 Your copy of the repository

Work in **your own copy** (a fork, or a copy in your own Git server): your inventory and your
encrypted passwords will live in it. Keep it in your Linux home folder. On WSL, **not** under
`/mnt/c`: Ansible ignores `ansible.cfg` in a folder that everybody can write to, and then nothing
below works.

```bash
cd ~
git clone <url-of-your-copy> ome-infra
cd ome-infra/ansible
ansible-galaxy collection install -r requirements.yml
```

To edit the files on Windows, run `code .` in that folder (VS Code with the WSL extension).

### 1.2 SSH key and SSH configuration

Every server gets a user `ansible`. You log in with an SSH key, never with a password.

1. Create a key for your servers. Use a separate key, not your GitHub key:

   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_ome -C "yourname@your-organisation"
   ```

   This creates two files:

   | File | What it is | What you do with it |
   |---|---|---|
   | `~/.ssh/id_ome.pub` | **Public** key, one line starting with `ssh-ed25519` | Share it: it goes on the servers (chapter 2) |
   | `~/.ssh/id_ome` | **Private** key | Keep it on your laptop. Never share it, never put it in Git |

2. Tell SSH which key and user to use for your servers. Add to `~/.ssh/config`, with the address
   range of **your** servers:

   ```text
   Host <your-network>
       User ansible
       IdentityFile ~/.ssh/id_ome
       IdentitiesOnly yes
   ```

   `<your-network>` is a pattern such as `192.168.10.*`, or one block per server with its IP or name.
   `IdentityFile` points to the **private** key, the file without `.pub`. Pointing it to the `.pub`
   file is a common mistake: SSH then fails with `Permission denied (publickey)`.

### 1.3 Vault password

Your passwords are encrypted with Ansible Vault (chapter 9). Choose a strong Vault password for
your organisation, keep it in your password manager, and store it in a file that Ansible reads
(`vault_password_file` in `ansible.cfg`):

```bash
mkdir -p ~/.ansible
nano ~/.ansible/ome-vault-password      # paste only the password, on one line
chmod 600 ~/.ansible/ome-vault-password
```

Never put the Vault password in Git or in a chat. Another location works with
`--vault-password-file <path>`.

### 1.4 Or use a control machine

Instead of your laptop, you can run Ansible from a small Linux server in the same network as your
nodes (a *control machine*). Prepare it like a laptop (chapters 1.1 to 1.3), with **its own** SSH
key. Its public key must be in `admin_keys.yml` and `cloud-init/bootstrap.yml` (chapter 2), like
the key of every team member: Ansible removes every key that is not in the list.

---

## 2. Make the repository yours

The repository contains the inventories, keys and encrypted passwords of the Odisee team. Replace
them with your own before you configure a server. The roles and playbooks stay as they are.

| What | File | Change |
|---|---|---|
| Your servers | `inventories/<env>/hosts.yml` | Names and `ansible_host` (IP or DNS name) of **your** nodes, in the group `ome_nodes`. Keep the group name: the roles and the tests use it. Remove the group `monitoring` if you do not run Uptime Kuma (chapter 8). |
| Environment name | `inventories/<env>/group_vars/all/vars.yml` | `ome_env`: `dev`, `staging` or `production` |
| Passwords | `inventories/<env>/group_vars/all/vault.yml` | Delete the Odisee files and create your own (chapter 9). They cannot be decrypted without the Odisee Vault password anyway. |
| Who may log in | `playbooks/group_vars/all/admin_keys.yml` | The public SSH keys of **your** team. Only these keys are allowed: Ansible removes every other key from the servers. Your own key must be in it. |
| First login on a new server | `cloud-init/bootstrap.yml` | Your own public key under `ssh_authorized_keys` (chapter 3.1) |
| Application and images | `roles/ome/defaults/main.yml` | `ome_repo_url` (the repository the NNTP image is built from), `ome_registry` (where the API and frontend images are pulled from), `ome_branch_map` (which branch each environment follows) |
| Your organisation | `roles/ome/defaults/main.yml` | `ome_organization` (shown in every article you post) and `ome_cms_plugin` (the plugin of your platform) |

The keys are in two files because they are used at different moments:

| File | Used | Effect |
|---|---|---|
| `cloud-init/bootstrap.yml` | Once, at the first boot of a **new** server | Lets Ansible log in the first time |
| `admin_keys.yml` | On **every** run of `site.yml` | Puts exactly this list on every existing server |

So when a team member joins or leaves, change both files and run `site.yml` on every environment.
A changed `bootstrap.yml` has no effect on servers that already exist.

Variables that are not in this table have safe defaults. Read `roles/*/defaults/main.yml` before
you change them.

The Odisee servers are not part of your network: your nodes only peer with each other, and with
other organisations once you agree to peer with them.

---

## 3. Add a server

The result is a node with the full OME stack (NNTP, API, frontend, Watchtower). If the
environment has more than one node, the nodes peer with each other automatically (chapter 4).

### 3.1 Create the server

Any server or VM that you manage works:

| | Requirement |
|---|---|
| Operating system | Rocky Linux 10 (tested). Other RHEL-like systems may work, Debian-based systems need changes in the roles `common` and `docker`. |
| Size | 2 vCPU, 2 GB memory, 30 GB disk |
| Network | SSH (22) from your laptop. HTTP (80) for users. Port 119 between the nodes that peer. |
| First login | A user `ansible`, SSH key only, passwordless `sudo` |

The first login can be set up in two ways:

- **With cloud-init** (most clouds): give the content of `cloud-init/bootstrap.yml`, with your own
  key, as *user data* when you create the VM. If your cloud can store user data under a name
  (for example *Stored User Data* in CloudStack), store it once and choose it for every new VM;
  update the stored copy when `bootstrap.yml` changes.
  - Do **not** also select an SSH key pair in the cloud's menu: that key goes to the image's default
    user (for example `rocky`), not to `ansible`.
  - Wait until cloud-init has finished before you log in. On the console, or once you can log in:
    `cloud-init status --long` must say `status: done`.
  - If the server asks for a **password** when you log in as `ansible`, the user data was not
    applied (or you were too early). Check that the user data was selected, or recreate the VM.
- **By hand** (a server without cloud-init), as root on the new server:

  ```bash
  useradd -m -G wheel ansible
  echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible
  chmod 440 /etc/sudoers.d/ansible
  install -d -m 700 -o ansible -g ansible /home/ansible/.ssh
  echo '<your public key>' > /home/ansible/.ssh/authorized_keys
  chown ansible:ansible /home/ansible/.ssh/authorized_keys
  chmod 600 /home/ansible/.ssh/authorized_keys
  ```

### 3.2 Configure it

1. **Accept the host key** once, from the shell where you run Ansible. Windows and WSL keep
   separate lists of known hosts:

   ```bash
   ssh ansible@<server-ip> 'sudo -n true && echo OK'
   ```

   Answer `yes` and expect `OK`. Do not disable host key checking in `ansible.cfg`: it is what
   protects you against a server that pretends to be yours.
2. **Add the server to the inventory**, in the group `ome_nodes` of `inventories/<env>/hosts.yml`:

   ```yaml
   ome_nodes:
     hosts:
       <node-name>:
         ansible_host: <server-ip>
   ```

   `<node-name>` becomes the name of the node in the NNTP network, so it must be unique. Use a name
   that includes your organisation, for example `nntp1-yourorg`.
3. **Check** that Ansible reads the inventory and reaches the server:

   ```bash
   ansible-inventory -i inventories/<env> --graph
   ansible -i inventories/<env> ome_nodes -m ping
   ```

   Expect `"ping": "pong"`: Ansible can log in and run Python (not a network ping).
4. **Configure the server.** The first run can take several minutes and may reboot the VM once
   (kernel modules for Docker, ADR 0007). `--limit` leaves the other nodes alone:

   ```bash
   ansible-playbook -i inventories/<env> playbooks/site.yml --limit <node-name>
   ansible-playbook -i inventories/<env> playbooks/site.yml --limit <node-name>
   ```

   The second run must report `changed=0`.
5. **Update the other nodes**, so they get the new node as a peer:

   ```bash
   ansible-playbook -i inventories/<env> playbooks/site.yml
   ```

6. **Check the node:**

   ```bash
   ssh ansible@<server-ip> 'sudo docker compose -f /opt/ome/compose.yml ps'
   curl -I http://<server-ip>/imls/
   ```

   Expect four containers `Up` and `HTTP/1.1 200 OK`. In a browser, the frontend is at
   `http://<server-ip>/imls/`, on port 80 (`ome_frontend_port`). Port 4000 in the README of the
   application is only for local development.
7. **Create the newsgroups** the node must carry (chapter 4.4) and **test peering** (chapter 5).

The API and frontend images for the environment must exist in your registry (`ome_registry`) first.

---

## 4. Peering between nodes

### 4.1 How it works

Nodes exchange articles over NNTP (port 119). For every node, the role `ome`:

1. **builds the list of peers** from the inventory: every other host in `ome_nodes`. A node in an
   environment with one server has no peers, and nothing below is switched on;
2. writes that list into `config.json`; at start the NNTP container turns it into the INN
   configuration (`innfeed.conf`: where to send articles, `incoming.conf`: whom to accept them from);
3. **publishes port 119** on the IPv4 addresses of the server, only when the node has peers;
4. **lets only the peers in**, with an own nftables table `inet ome_nntp` (ADR 0010).

Step 4 is needed because Docker forwards published ports past the normal firewalld rules: a
firewalld rule for port 119 does not work, not even a firewalld policy. The table is loaded at
boot by the service `ome-nntp-firewall`, before Docker. If it cannot be loaded, Docker does not
start (fail closed): a node that is down is better than a node that is open.

If there is another firewall between your nodes (a cloud security group, a router), allow TCP 119
between the nodes there as well.

### 4.2 Add or remove a node

Add or remove the server in `inventories/<env>/hosts.yml` and run `site.yml` for the whole
environment. Every node gets the new list of peers and the new firewall table.

### 4.3 Check the firewall of a node

On the node:

```bash
sudo nft list table inet ome_nntp          # the set "peers" holds the IPs of the other nodes
systemctl status ome-nntp-firewall          # "active (exited)" is good
sudo ss -tlnp | grep ':119'                 # only 0.0.0.0:119, no [::]:119
```

`nft list` does not show the IPv6 check of the second rule, but it is there (ADR 0010).

From a machine that is **not** a peer, port 119 must be refused:

```bash
timeout 5 bash -c 'exec 3<>/dev/tcp/<server-ip>/119; head -n 1 <&3'
```

Expect `Connection refused`. From a peer, expect a greeting that ends with `(transit mode)`.

Reload the rules after a manual change with `sudo systemctl reload ome-nntp-firewall`.
Do **not** use `stop` or `restart`: Docker depends on this service and stops or restarts with it.

### 4.4 Newsgroups

A node only stores articles of newsgroups it knows. A new node has none of the OME groups, and
NNTP does not create them by itself: create every group on **every** node. For example, for the
QUBES plugin:

```bash
sudo docker compose -f /opt/ome/compose.yml exec -T nntp-server ctlinnd newgroup ome.qubes
sudo docker compose -f /opt/ome/compose.yml exec -T nntp-server grep '^ome\.' /var/lib/news/active
```

The newsgroups of each plugin are listed in `server/plugins/<plugin>/README.md` of the application
repository. The test of chapter 5 creates its own group `ome.test`.

---

## 5. Test peering

The playbook `playbooks/peering-test.yml` checks that every node receives the articles of every
other node in the inventory:

```bash
ansible-playbook -i inventories/<env> playbooks/peering-test.yml
```

It takes less than a minute and does not change the configuration of the nodes. It needs at least
two nodes in `ome_nodes` and stops at once with a clear message if there are fewer.

**What it does**

1. On every node: create the group `ome.test` if needed, and post one test article with a unique
   Message-ID, for example `<peering-test-20261107140810-nntp1-yourorg@nntp1-yourorg.invalid>`.
2. On every node: wait up to one minute until the test articles of **all the other nodes** are
   in its history (`grephistory`). Normally they are there within a few seconds.
3. If an article is missing: show `innfeed.log` of the senders (filtered on the node that missed
   it) and fail with a message that says which node missed the articles of which senders.

The test articles are real articles: on a network that other organisations peer with, they spread
to their nodes as well.

**Reading the result**

All good:

```text
ok: [nntp1-yourorg] => (item=nntp2-yourorg)
msg: nntp1-yourorg received the test articles of nntp2-yourorg
```

Something is wrong:

```text
FAILED - RETRYING: [nntp2-yourorg]: Wait until the test article of each peer has arrived (12 retries left).
...
failed: [nntp2-yourorg] (item=nntp1-yourorg) => ... stderr: 'grephistory: not found'
```

Read the failed line as: *nntp2-yourorg did not receive the article of nntp1-yourorg*. Only that
direction is broken. `grephistory: not found` means that the article is not in the history of
that node, not that a program is missing.

Below the failed line, the playbook shows `innfeed.log` of the sender. How to read it:

| Line in `innfeed.log` | Meaning |
|---|---|
| `Giving article to peer: <receiver>` | The sender tries to deliver the article |
| `<receiver>:0 Setting up a reopen callback` | It cannot connect and retries later, every time after a longer pause |
| `Processing mode response: 203 Streaming permitted` | It is connected to the receiver |
| `<receiver> Peeled article <…> from backlog` | It delivers an article that was waiting in its backlog |
| `<receiver> All out of articles in the backlog` | Nothing is waiting any more |

Where to look next:

| Check | Where | Command |
|---|---|---|
| Is the sender in the firewall list? | Receiver | `sudo nft list set inet ome_nntp peers` |
| What did the sender do? | Sender | `sudo docker compose -f /opt/ome/compose.yml exec -T nntp-server sh -c 'grep -F <receiver> /var/log/news/innfeed.log \| grep -v hostChkCxns \| tail -n 20'` |
| Does the group exist? | Receiver | `sudo docker compose -f /opt/ome/compose.yml exec -T nntp-server grep '^ome\.test ' /var/lib/news/active` |
| Do all containers run? | Both | `sudo docker compose -f /opt/ome/compose.yml ps` |
| Is there another firewall between the nodes? | Network | Chapter 4.1 |

`grep -F <receiver>` is needed: innfeed logs a lot (debug level 4), and without the filter the last
lines are only `Checkpointing tapes`.

The playbook logs in to every node, so it can only test nodes in **your** inventory. It cannot
test peering with a node of another organisation.

---

## 6. Test store and forward

This test proves that a node that was unreachable still gets the articles posted in the meantime.
It **cuts one node off from its peers for about a minute**, so do not run it on production while
people use it. That is why it only runs when you ask for it:

```bash
ansible-playbook -i inventories/<env> playbooks/peering-test.yml --tags all,store_and_forward
```

`all` runs the normal test of chapter 5 first; `store_and_forward` adds this test. Without the tag
it never runs. It needs at least two nodes.

**What it does**

1. The **receiver** (the last node in the inventory) empties its firewall list of peers: the other
   nodes can no longer reach it.
2. Every other node posts an article.
3. The playbook waits 30 seconds (`peering_test_delivery_window`), longer than a normal delivery
   takes, and checks that the receiver does **not** have the articles. Without that wait, the check
   would also pass if the cut-off did not work.
4. The receiver loads its firewall list again.
5. The playbook waits up to five minutes until the articles arrive on the receiver. The senders
   retry with longer and longer pauses (30 seconds, 1 minute, 2 minutes, ...): on the Odisee
   staging nodes the articles arrived about one minute after step 4.

Step 4 also runs when the test fails (`always`), so the receiver is never left cut off. When a
check fails, the playbook shows `innfeed.log` of the senders, as in chapter 5.

Expected result: `ok: [<receiver>] => (item=<sender>)` for the task *Wait until the articles arrive
after the interruption*, and `failed=0`. In `innfeed.log` of a sender you then see
`Setting up a reopen callback` while the receiver was cut off, and `Peeled article <…> from backlog`
when it was delivered.

Not covered yet: a restart of the **sending** node during the interruption (chapter 11).

---

## 7. Updates and roll back

**Application updates.** Watchtower runs on every node and checks the registry every five minutes
(`ome_watchtower_interval`). When a new API or frontend image is published under the tag your
environment follows (`ome_image_tag`, by default the name of the environment), Watchtower
replaces the container.

The NNTP image is built on the server by Ansible, from the branch of the application repository
that the environment follows (`ome_branch_map`). Run `site.yml` to update it: when that branch has
new commits, Ansible builds a new image and recreates only the NNTP container. Without new commits,
nothing is built or restarted. Until NNTP has a volume, a recreate deletes the articles on that
node (chapter 11).

Follow Watchtower, or make it check now:

```bash
ssh ansible@<server-ip> 'sudo docker logs -f --since 10m ome-watchtower-1'
ssh ansible@<server-ip> 'sudo docker run --rm -v /var/run/docker.sock:/var/run/docker.sock nickfedor/watchtower:1.19.0 --run-once --label-enable --cleanup'
```

**Infrastructure changes** are applied with `ansible-playbook`. Test them on a test environment
first. If a new version of the application changes how the images start (Dockerfile, entrypoint),
update the compose template of the role `ome` before you roll it out.

**Roll back.** A release breaks something: go back to the previous version first, find the cause
after. There are two ways.

1. **Revert in Git (the normal way).** On GitHub, open the merged pull request and click **Revert**.
   Merge the pull request it creates (and promote it, for staging or production). CI publishes a new
   image, and Watchtower puts it on the servers within about ten minutes. Nothing to do on the servers.
2. **By hand on the server (emergency only)**, when the environment cannot wait for that. This applies
   to the API and frontend; every image is also published as `<env>-<sha>`, so a previous version
   stays available.

   1. Stop Watchtower. Otherwise it sees that the image under the tag of the environment differs
      from the registry, and puts the broken version back within five minutes:

      ```bash
      sudo docker compose -f /opt/ome/compose.yml stop watchtower
      ```

   2. Give a previous image the tag of the environment, and restart only that service (here the
      API; for the frontend use `ome-frontend` and `imls-react`):

      ```bash
      sudo docker pull ghcr.io/<org>/ome-api:<env>-<sha>
      sudo docker tag ghcr.io/<org>/ome-api:<env>-<sha> ghcr.io/<org>/ome-api:<env>
      sudo docker compose -f /opt/ome/compose.yml up -d --no-deps fastapi-server
      ```

   3. Revert in Git as in way 1, and wait until the fixed image is published.
   4. Start Watchtower again, and write a short incident note:

      ```bash
      sudo docker compose -f /opt/ome/compose.yml up -d watchtower
      ```

The NNTP image is built on the server, so for NNTP only way 1 applies: revert, then run `site.yml`.

---

## 8. Status, logs and monitoring

On a node:

```bash
sudo docker compose -f /opt/ome/compose.yml ps
sudo docker compose -f /opt/ome/compose.yml logs --tail 50 <service>
sudo docker compose -f /opt/ome/compose.yml exec -T nntp-server sh -c 'grep -F <peer> /var/log/news/innfeed.log | grep -v hostChkCxns | tail -n 20'
sudo docker events --since 1h --filter type=container --filter type=image
```

- Times in the INN logs and in `docker events` are in UTC. Compare times in one time zone.
- The article log of INN (`+` accepted, `-` refused) is in `docker compose logs nntp-server`, but
  it can lag minutes behind: innd runs in debug mode, and `ctlinnd flushlogs` is refused. To check
  whether a node has an article, use `grephistory '<message-id>'` instead.
- `news.notice` and `news.err` stay empty: there is no syslog in the container (chapter 11).
- `docker events` shows what happened to containers and images (built, tagged, stopped, recreated):
  useful to check what a run of `site.yml` really did.

**Monitoring (optional).** The role `monitoring` installs Uptime Kuma on the hosts in the group
`monitoring`. Run it on a different server than the ones it watches: a monitor on a server that
goes down cannot warn you. After installing it, add two monitors per node: `http://<server-ip>/imls/`
(frontend) and `http://<server-ip>/api/list` (API and NNTP behind it). See `docs/monitoring.md`.

```bash
ansible-playbook -i inventories/<env> playbooks/monitoring.yml
```

---

## 9. Secrets

The passwords of an environment are in `inventories/<env>/group_vars/all/vault.yml`, encrypted
with Ansible Vault (with the password of chapter 1.3).

```bash
cp inventories/vault.example.yml inventories/<env>/group_vars/all/vault.yml
openssl rand -hex 20                                    # one per password
nano inventories/<env>/group_vars/all/vault.yml         # paste the generated passwords
ansible-vault encrypt inventories/<env>/group_vars/all/vault.yml
```

Change a password with `ansible-vault edit`, then run the playbook: the containers are recreated
with the new configuration (ADR 0009). Before a commit, the file must start with
`$ANSIBLE_VAULT;1.1;AES256`. Never paste the content of a vault in a chat.

---

## 10. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `Host key verification failed` | The host key is not accepted in this shell: `ssh ansible@<server-ip>` once (chapter 3.2). After a server was recreated: `ssh-keygen -R <server-ip>` first. |
| `Permission denied (publickey)` | Wrong key: check `~/.ssh/config` (`IdentitiesOnly yes`, `IdentityFile` without `.pub`) and that your key is in `admin_keys.yml`. On an existing server, your key only arrives after a run of `site.yml` by someone who can already log in. |
| A new server asks for a password | The cloud-init user data was not applied, or you logged in before cloud-init finished (chapter 3.1). |
| The frontend does not answer on port 4000 | On the servers it is on port 80: `http://<server-ip>/imls/` (chapter 3.2). |
| `Decryption failed` | Wrong Vault password, or a `vault.yml` encrypted with another password (chapter 2). |
| `nntp-server` keeps `Restarting` after a reboot or a restart of Docker | Leftover pid, socket and lock files from the previous run. Recreate the containers: `sudo docker compose -f /opt/ome/compose.yml up -d --force-recreate nntp-server fastapi-server`. A fix in the image is planned. |
| NNTP runs old code, or its image is missing, after a run of `site.yml` that failed halfway | Build and recreate it by hand: `sudo docker compose -f /opt/ome/compose.yml up -d --build --force-recreate --no-deps nntp-server`. A fix is planned: publish the NNTP image instead of building it on the servers. |
| `A dependency job for docker.service failed` | The firewall table could not be loaded, so Docker does not start (on purpose). `systemctl status ome-nntp-firewall` and `sudo nft -c -f /etc/nftables/ome-nntp-peers.nft` show the error. Fix it, then start `ome-nntp-firewall` and `docker`. |
| Peering test fails for one direction | Chapter 5, "Reading the result". |
| A non-peer still reaches port 119 | `sudo nft list table inet ome_nntp` must exist and `ss -tlnp` must only show `0.0.0.0:119` (chapter 4.3). |
| Docker: `missing kernel module` | Handled by the role `docker` (one reboot). |
| Playbook: `compose.yml: ... false schema` | A typo in a key of `compose.yml.j2`. |
| Playbook: image `... not found` | The image does not exist in your registry under the tag of the environment yet. |
| API `Restarting`: `Config not found at /run/secrets/ome_config` | The `ome_config` secret is missing in the compose template (ADR 0009). |
| API calls return 502 | `sudo docker compose -f /opt/ome/compose.yml logs fastapi-server` |
| Watchtower: `Failed to retrieve image info … No such image` for NNTP | The NNTP container runs an image that no longer exists (it was rebuilt, but the container was not recreated). Run `site.yml`, or recreate it: `sudo docker compose -f /opt/ome/compose.yml up -d --force-recreate --no-deps nntp-server`. |

---

## 11. Known limitations

- **No NNTP volume** (#150): articles, history and the innfeed backlog live in the container and
  are lost when it is recreated, which happens after every new build of the NNTP image (chapter 7).
- **Store and forward after a restart of the sender** is not tested: the backlog is written to tape
  files in the container, but whether it survives a restart is not proven.
- **Newsgroups are not created automatically** on new nodes (chapter 4.4).
- **Restart of the NNTP container** needs a recreate (chapter 10). innd does not stop on SIGTERM:
  Docker kills it after 10 seconds, which leaves pid and lock files behind.
- **INN logs to syslog**, which does not run in the container: `news.notice` stays empty. The
  article log lags behind because innd runs in debug mode (chapter 8).
- **The peering test** only covers nodes in one inventory, not peering with another organisation.
- **The NNTP image is built on every server** instead of pulled from a registry.
- No DNS or HTTPS yet (#108, #109).

---

## Appendix: Odisee environments

Only for the Odisee team. Other organisations use their own values (chapter 2).

| Environment | Nodes | Follows branch | Image tag |
|---|---|---|---|
| dev | `ome-development` (10.129.80.46) | `main` | `dev` |
| staging | `ome-staging` (10.129.80.47), `ome-staging-2` (10.129.80.64) | `staging` | `staging` |
| production | `ome-production` (10.129.80.54) | `production` | `production` |

- **Servers:** school cloud (CloudStack), template Rocky Linux 10, offering `ICT-Linux-Server`,
  root disk `ICT-Linux-Root` 30 GB, no data disk, Stored User Data **`ome-bootstrap`** (the content
  of `cloud-init/bootstrap.yml`; no SSH key pair). Reachable from the
  school network or the VPN. The school can stop VMs at night: check the containers in the morning
  (chapter 10, `Restarting`).
- **SSH:** `Host 10.129.80.*` in `~/.ssh/config`.
- **Vault password:** in the team password manager.
- **Repository and images:** fork `odisee-ome-2627/Open-Metadata-Exchange`, images on
  `ghcr.io/odisee-ome-2627`.
- **Releases:** dev follows merges into `main`; staging a PR `main → staging`; production a PR
  `staging → production` (approval, merge commit). Production only runs what is merged.
- **Monitoring:** Uptime Kuma on `http://10.129.80.47:3001/`, alerts in Discord `#ome-alerts`.
- **API docs** (not through nginx): `ssh -L 8000:172.21.0.3:8000 ansible@<server-ip>`, then
  `http://localhost:8000/docs`.
- **Rollback:** revert the pull request on GitHub (chapter 7, way 1). Only in an emergency by hand
  (way 2), with `ghcr.io/odisee-ome-2627` as registry; write the incident note in `docs/incidents/`.