# Foundry

Foundry is the configuration repository for a personal Ubuntu homelab server.
The repo is centered on Ansible-managed host provisioning, with Docker service
configuration kept alongside the playbooks that deploy it.

## What This Manages

- Baseline Ansible connectivity for the `foundry` host
- Human user accounts, shell setup, SSH access, and developer CLIs
- Tailscale enrollment
- Thunderbolt-attached storage mounted at `/mnt/store`
- Docker Engine and the Plex Compose service under `docker/`
- Explicit system package maintenance through a dedicated update playbook

## Repository Layout

```text
.
  ansible/    Ansible inventory, playbooks, roles, and operator docs
  docker/     Compose files deployed by the Ansible Docker role
```

The detailed runbook lives in [ansible/README.md](ansible/README.md). Start
there for manual server prerequisites, collection installation, baseline
provisioning, individual playbook commands, vault usage, and maintenance tasks.

## Common Commands

Run commands from the Ansible directory so the repo-local `ansible.cfg` is used:

```sh
cd ansible
ansible-playbook foundry.yml
```

Run package maintenance explicitly:

```sh
cd ansible
ansible-playbook playbooks/system-updates.yml
```

The update playbook reports whether a reboot is required. To reboot
automatically only when required:

```sh
cd ansible
ansible-playbook playbooks/system-updates.yml -e system_updates_reboot=true
```

## Local Private Values

Committed Ansible defaults are intentionally public-safe. Host-specific human
user details belong in `ansible/group_vars/all/99-private.yml`, which is ignored
by git and loaded by Ansible when present.
