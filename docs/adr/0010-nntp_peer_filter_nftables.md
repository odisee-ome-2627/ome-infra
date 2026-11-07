# ADR 0010 – Only NNTP peers reach port 119, filtered with an own nftables table

- Status: Accepted
- Date: 06-11-2026
- Deciders: infra team

## Context

For peering (#132, #133) port 119 of the NNTP container must be reachable from the other
nodes, and only from them: otherwise anyone on the network can connect to INN.

Port 119 is published with Docker (`ports: "119:119"`, only when a node has peers).
Docker forwards published ports to the container with DNAT, so this traffic takes the
FORWARD path, not INPUT. The normal firewalld rules of zone `public` (INPUT) never see it.

Tested on staging (Rocky Linux 10, Docker 29.9, firewalld with the nftables backend):

1. Port 119 published, no rule for it in zone `public`: a laptop that is not a peer
   got the INN greeting (`200 ... NNRP server`).
2. A firewalld **policy** (ingress `public`, egress `docker`, priority -10, reject port 119
   unless the source is in an ipset of peers): active and correct according to
   `firewall-cmd`, but the laptop still got the greeting. Cause: firewalld's forward chain
   accepts all DNAT traffic (`ct status dnat accept`) **before** it evaluates policies.
   Firewalld cannot filter ports published by Docker.

INN itself sees the real source IP of a peer (DNAT keeps the source), so `incoming_ip` in
`config.json` can be the peer's IP (risk 3 of the peering research does not occur here).

## Decision

- An own nftables table **`inet ome_nntp`**, written by the role `ome` from
  `ome-nntp-peers.nft.j2` to `/etc/nftables/ome-nntp-peers.nft`:
  - a set `peers` with the IPv4 addresses of the peers (`ome_nntp_peers`, built from the inventory);
  - a chain on the **forward** hook, priority `filter - 5`, policy accept, that rejects
    **DNAT traffic to port 119** from every source that is not in `peers`, and all IPv6
    traffic to port 119.
- A reject in any nftables chain is final, so Docker's and firewalld's accepts cannot undo it.
- Only DNAT traffic is matched (`ct status dnat`): the API reaches INN inside the Docker
  network without DNAT and is not affected.
- A systemd service **`ome-nntp-firewall`** loads the file at boot, after firewalld and
  **before Docker**, so the port is never open without the filter, also not after a reboot.
- A drop-in for `docker.service` (`docker.service.d/ome-nntp-firewall.conf`, `Requires=` and
  `After=`) makes Docker depend on that service: if the filter cannot be loaded, Docker and the
  OME stack do not start (fail closed). `Before=` in the service alone only sets the order.
- Ansible validates the file with `nft -c` before placing it, and reloads it with `nft -f`
  when it changes, before the stack starts (`flush_handlers`).

## Consequences

- Tested on staging: a non-peer gets `Connection refused`, a peer gets the INN transit
  greeting; the table survives `firewall-cmd --reload` and a reboot; second run `changed=0`.
- Adding a node to the inventory updates the set of peers on every node at the next run.
- A third firewall layer next to firewalld and Docker. Check it with
  `nft list table inet ome_nntp`, not with `firewall-cmd`.
- `nft list` does not show the IPv6 check of the second rule, but it is there
  (verified with `nft --debug=netlink`).
- Stopping the service does not remove the rules (fail closed). Removing them is a
  deliberate action: `nft delete table inet ome_nntp`.
  - If the filter fails to load, the whole node is down (also the frontend) and Uptime Kuma
  alerts: a node that is down is better than a node that is open without anyone knowing.
  Tested: with a broken `.nft` file Docker refused to start
  (`A dependency job for docker.service failed`).
- `Requires=` also works the other way: `systemctl stop` or `restart ome-nntp-firewall` stops or
  restarts Docker. Reload the rules with `systemctl reload ome-nntp-firewall` or `nft -f`.

## Alternatives considered

- **Normal firewalld rule (zone `public`)**: not applied to published ports (INPUT vs FORWARD).
- **Firewalld policy on forwarded traffic**: tested, does not work (`ct status dnat accept`).
- **Docker's `DOCKER-USER` chain** (iptables): the way Docker documents, but the rules
  are lost at reboot and need an own script to come back, after Docker has already started.
- **`network_mode: host` for NNTP**: firewalld would work as usual, but the API would no
  longer reach INN through the Docker network, and the setup would differ more from upstream.