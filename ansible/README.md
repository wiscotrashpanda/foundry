# Ansible

This directory contains the Ansible configuration for the local Ubuntu server
named `foundry`. The inventory target `foundry` connects through your local SSH
host alias `foundry-admin`, which logs in as the provisioning `ansible` user.

## Connectivity Check

Run Ansible from this directory so it picks up the repo-local
`ansible.cfg`:

```sh
cd ansible
ansible foundry -m ansible.builtin.ping
```

The same check is also available as a playbook:

```sh
cd ansible
ansible-playbook playbooks/connectivity.yml
```

## External Storage

Authorize the G-Technology G-DRIVE Thunderbolt device, mount its ext4
filesystem at `/mnt/store`, persist the mount in `/etc/fstab`, and prepare the
Plex directories used by `docker/plex-media-server.yml`:

```sh
cd ansible
ansible-playbook playbooks/storage.yml
```

The playbook enrolls Thunderbolt device
`cd010000-0080-7518-a3c0-2d8c0a218202` with `boltd` and mounts filesystem UUID
`c072b758-e4e4-44ea-8251-ef0891930805`.

If `host_vars/foundry/vault.yml` exists, include `--ask-vault-pass` when using
the normal inventory so Ansible can load the encrypted host variables.

## User Configuration

Configure the human users, membership in the `admin-sudo` passwordless sudo
group, SSH access from
`~/.ssh/foundry.pub`, their home-directory `code` folders, shared
`xterm-ghostty` terminfo, zsh, Oh My Zsh with its standard boilerplate
`.zshrc`, and an `ls` to `eza` alias:

```sh
cd ansible
ansible-playbook playbooks/users.yml
```

## Tailscale

Install the Tailscale collection locally before the first run:

```sh
cd ansible
ansible-galaxy collection install -r requirements.yml
```

Then install Tailscale, enable `tailscaled`, and add the server to your tailnet
using an auth key from the controller environment:

```sh
cd ansible
TAILSCALE_AUTHKEY=tskey-auth-... ansible-playbook playbooks/tailscale.yml
```

To store the key persistently, create an encrypted host vault:

```sh
cd ansible
ansible-vault create host_vars/foundry/vault.yml
```

Add this variable to the vault:

```yaml
vault_tailscale_authkey: tskey-auth-...
```

Then run with the vault password instead of exporting the key each time:

```sh
cd ansible
ansible-playbook playbooks/tailscale.yml --ask-vault-pass
```

The playbook uses `artis3n.tailscale.machine` and sets the Tailscale node name
to `foundry`. To pass extra flags to `tailscale up`, set
`foundry_tailscale_args`:

```sh
cd ansible
TAILSCALE_AUTHKEY=tskey-auth-... ansible-playbook playbooks/tailscale.yml \
  -e '{"foundry_tailscale_args":"--hostname=foundry --ssh"}'
```
