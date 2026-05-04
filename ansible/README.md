# Ansible

This directory contains the Ansible configuration for the local Ubuntu server
named `foundry`. Run commands from this directory so the repo-local
`ansible.cfg` is used.

The steady-state inventory target `foundry` connects through the local SSH host
alias `foundry-admin`, which should log in as the provisioning `ansible` user.

## Layout

```text
ansible/
  ansible.cfg
  inventory.yml
  group_vars/all.yml
  foundry.yml
  playbooks/
  roles/
  vault/
```

`playbooks/` contains the command entrypoints. `roles/` contains the reusable
implementation. `vault/` contains encrypted secret files that are loaded only by
the playbook that needs them, so ordinary inventory operations do not require a
vault password.

## Vault Password Helper

`ansible.cfg` reads the vault password from `.vault-password`. This file is
gitignored. If it fetches the password from 1Password, it must be executable and
start with a shebang:

```sh
#!/bin/sh
op read "op://<vault>/<item>/<field>"
```

If `.vault-password` contains the raw password instead, remove the executable
bit so Ansible reads it as a plain password file.

## Manual Server Prerequisites

Complete these one-time steps directly on the server before running the normal
playbooks.

Prevent the server from suspending when the laptop lid is shut by updating
`/etc/systemd/logind.conf`:

```ini
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Restart `systemd-logind` or reboot the server so the setting takes effect.

Create the `admin-sudo` group, allow that group to use passwordless sudo, and
add the provisioning `ansible` user to it:

```sh
sudo groupadd --force admin-sudo
sudo usermod -aG admin-sudo ansible
printf '%%admin-sudo ALL=(ALL) NOPASSWD:ALL\n' | sudo tee /etc/sudoers.d/admin-sudo
sudo chmod 0440 /etc/sudoers.d/admin-sudo
sudo visudo -cf /etc/sudoers.d/admin-sudo
```

## Install Collections

Install required collections once:

```sh
cd ansible
ansible-galaxy collection install -r requirements.yml
```

## Connectivity Check

```sh
cd ansible
ansible foundry -m ansible.builtin.ping
ansible-playbook playbooks/connectivity.yml
```

## Baseline Provisioning

Run the normal baseline:

```sh
cd ansible
ansible-playbook foundry.yml
```

`foundry.yml` imports the ordinary playbooks in order:

1. `playbooks/users.yml`
2. `playbooks/storage.yml`
3. `playbooks/docker.yml`

Tailscale is not part of the baseline because enrollment needs an auth key.
Run it explicitly when you want to enroll or refresh Tailscale.

## Individual Playbooks

```sh
cd ansible
ansible-playbook playbooks/terminfo.yml
ansible-playbook playbooks/users.yml
ansible-playbook playbooks/storage.yml
ansible-playbook playbooks/docker.yml
ansible-playbook playbooks/tailscale.yml
```

## User Configuration

The `users` role manages:

- human users from `foundry_users`
- membership in the `admin-sudo` passwordless sudo group
- SSH access from `~/.ssh/foundry.pub`
- each user's `/home/<user>/code` directory
- zsh, Oh My Zsh, `.zshrc`, and `.gitconfig`
- shared `xterm-ghostty` terminfo through the `terminfo` role dependency

User names and email addresses live in `group_vars/all.yml`.

## External Storage

The `storage` role authorizes the G-Technology G-DRIVE Thunderbolt device,
mounts filesystem UUID `c072b758-e4e4-44ea-8251-ef0891930805` at `/mnt/store`,
persists the mount in `/etc/fstab`, and prepares the Plex directories used by
`docker/plex-media-server.yml`.

## Docker

The `docker` role installs Docker Engine from Docker's official apt repository,
starts the `docker` service, adds existing human users to the `docker` group,
and deploys the Plex Compose file to `/srv/docker/plex-media-server.yml`.

Users may need to start a new login session before Docker group membership is
active.

## Tailscale

One-shot with an environment variable:

```sh
cd ansible
TAILSCALE_AUTHKEY=tskey-auth-... ansible-playbook playbooks/tailscale.yml
```

Persistent encrypted secret:

```sh
cd ansible
ansible-vault create vault/foundry.yml
```

Add this variable:

```yaml
vault_tailscale_authkey: tskey-auth-...
```

Then run:

```sh
cd ansible
ansible-playbook playbooks/tailscale.yml --ask-vault-pass
```

To pass extra flags to `tailscale up`, override `foundry_tailscale_args`:

```sh
cd ansible
ansible-playbook playbooks/tailscale.yml \
  -e '{"foundry_tailscale_args":"--hostname=foundry --ssh"}'
```
