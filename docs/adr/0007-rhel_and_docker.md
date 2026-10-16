# ADR 0007 – Rocky Linux 10 with Docker CE (iptables backend)

- Status: Accepted
- Date: 15-10-2026
- Deciders: infra team

## Context

CloudStack offers no Ubuntu template, only an Ubuntu ISO and templates for Rocky Linux,
Fedora and CentOS. Docker CE 29 failed to start on a fresh Rocky 10 VM:

```
Extension addrtype revision 0 not supported, missing kernel module?
```

Rocky/RHEL 10 cloud images do not ship `kernel-modules-extra`, which contains `xt_addrtype`.
The repository only offers that package for the newest kernel, so installing it also pulled
in a newer kernel than the one running.

## Decision

- Use the **Rocky Linux 10 template** (not the Ubuntu ISO): it boots in a minute and accepts
  cloud-init and SSH keys. An ISO needs a manual installation per VM.
- Keep **SELinux enforcing** and **firewalld enabled**.
- Install Docker CE from **Docker's official RHEL repository**, verifying the GPG fingerprint
  from Docker's documentation; remove Podman and other conflicting packages first.
- The `docker` role installs `kernel-modules-extra`, checks whether `xt_addrtype` exists for
  the **running** kernel, **reboots only if it does not**, and loads `xt_addrtype` and
  `br_netfilter` now and at every boot (`/etc/modules-load.d/docker.conf`).

## Consequences

- The first playbook run on a new VM may reboot it once. Later runs skip the reboot.
- Kernel updates keep `kernel-modules-extra` in step (installed alongside the kernel).
- Docker CE runs without SELinux integration by default (`selinux-enabled: false`):
  containers are not SELinux-confined. Enabling it is a hardening item for the security audit.

## Alternatives considered

- **Docker's nftables firewall backend**: avoids `xt_addrtype`, but is experimental in
  Docker 29. Rejected for stable environments.
- **Podman** (default on Rocky): upstream and the team use Docker Compose; Watchtower targets Docker.
- **Disabling SELinux/firewalld**: easier, but weakens the servers for no real gain.
