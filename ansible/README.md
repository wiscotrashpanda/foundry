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
  group_vars/all/
  foundry.yml
  playbooks/
  roles/
  vault/
```

`playbooks/` contains the command entrypoints. `roles/` contains the reusable
implementation. `vault/` contains encrypted secret files that are loaded only by
the playbook that needs them, so ordinary inventory operations do not require a
vault password.

`group_vars/all/00-public.yml` contains public-safe defaults that are safe to
commit. Put local human-user details in `group_vars/all/99-private.yml`; that
file is ignored by git and overrides the public defaults when present.

## Vault Password Helper

Ordinary playbooks do not load a vault password by default. When a playbook
needs encrypted values, pass the helper explicitly:

```sh
ansible-playbook --vault-password-file .vault-password playbooks/tailscale.yml
```

`.vault-password` is gitignored. If it fetches the password from 1Password, it
must be executable and start with a shebang:

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

1. `playbooks/connectivity.yml`
2. `playbooks/terminfo.yml`
3. `playbooks/tailscale.yml`
4. `playbooks/users.yml`
5. `playbooks/storage.yml`
6. `playbooks/samba.yml`
7. `playbooks/docker.yml`

## Individual Playbooks

```sh
cd ansible
ansible-playbook playbooks/terminfo.yml
ansible-playbook playbooks/tailscale.yml
ansible-playbook playbooks/users.yml
ansible-playbook playbooks/storage.yml
ansible-playbook playbooks/samba.yml
ansible-playbook playbooks/docker.yml
ansible-playbook playbooks/system-updates.yml
```

## System Updates

Run package maintenance explicitly when you want the server to apply available
Ubuntu package updates:

```sh
cd ansible
ansible-playbook playbooks/system-updates.yml
```

The update playbook runs a safe apt upgrade, removes unused packages, cleans old
package cache files, and reports whether the host needs a reboot. It does not
reboot by default. To reboot automatically only when required:

```sh
cd ansible
ansible-playbook playbooks/system-updates.yml -e system_updates_reboot=true
```

## User Configuration

`playbooks/users.yml` runs the `developer_clis` role before the `users` role.
Together they manage:

- human users from `foundry_users`
- system-wide developer CLIs installed once per host (see below)
- membership in the `admin-sudo` passwordless sudo group
- SSH access from `~/.ssh/foundry.pub`
- each user's `/home/<user>/code` directory
- per-user CLI config homes under `~/.config/{gh,tea,gcloud,opencode}` and
  `~/{.aws,.claude,.codex,.terraform.d}`
- zsh, Oh My Zsh, `.zshrc`, and `.gitconfig`
- shared `xterm-ghostty` terminfo through the `terminfo` role dependency

### Developer CLIs

The `developer_clis` role installs each tool once per host. Per-user state
(auth tokens, project context, command history) lives under each user's home
directory and is created by the `users` role. The role installs:

- `git` and GitHub CLI (`gh`) — apt-managed
- Gitea CLI (`tea`) — pinned binary from Gitea releases
  (`developer_clis_tea_version`)
- HashiCorp Terraform — apt-managed via HashiCorp's apt repo
- AWS CLI v2 — installed from AWS's official zip, re-installed only when the
  upstream GitHub release tag is newer than the local copy
- Google Cloud CLI — apt-managed (`google-cloud-cli` plus
  `google-cloud-cli-gke-gcloud-auth-plugin` for kubectl/GKE)
- Node.js — apt-managed via NodeSource (`developer_clis_nodejs_major`
  selects the major version)
- Claude Code (`@anthropic-ai/claude-code`) and OpenAI Codex (`@openai/codex`)
  — npm globals tracked at `@latest`
- opencode (`sst/opencode`) — installed from the latest GitHub release tarball

Apt-managed tools follow `developer_clis_apt_package_state` (default
`latest`). Set it to `present` to install missing packages without upgrading
existing ones. The npm globals always re-resolve `@latest`; bump
`developer_clis_npm_global_packages` to add or remove packages. AWS CLI and
opencode compare the installed version against the upstream release feed and
only download when the host is behind.

User names and email addresses live in the ignored
`group_vars/all/99-private.yml` file. No auth tokens are stored by Ansible;
each user runs `gh auth login`, `tea login add`, `aws configure`,
`gcloud auth login`, `claude`, `codex`, `opencode auth`, etc. from their own
account when credentials are available.

## External Storage

The `storage` role authorizes the G-Technology G-DRIVE Thunderbolt device,
mounts filesystem UUID `c072b758-e4e4-44ea-8251-ef0891930805` at `/mnt/store`,
persists the mount in `/etc/fstab`, and prepares the Plex directories used by
`docker/plex-media-server.yml`.

## SMB

The `samba` role installs Samba, allows only local and Tailnet source addresses,
disables NetBIOS name service, and serves the `store` share from
`/mnt/store/share`. Samba still opens TCP port `445` on the host, but
non-Tailnet clients are rejected by Samba's `hosts allow`/`hosts deny` rules.

No new partition is required for this setup. SMB serves a directory from the
existing `/mnt/store` filesystem; create a separate partition only if you want
hard isolation, different mount options, a quota boundary, or a separate backup
and lifecycle policy.

Access is limited to users in `samba_users`, which defaults to `foundry_users`.
Those users must also have Samba passwords because Samba does not reuse SSH
keys.

Set the Samba password once on the server:

```sh
ssh foundry
sudo smbpasswd -a josh
sudo smbpasswd -e josh
```

Then connect from macOS Finder with `smb://foundry/store`, choose
`Registered User`, and log in as `josh` with the Samba password from
`smbpasswd`.

Or store `samba_user_passwords` in the encrypted vault if you want Ansible to
manage SMB passwords, then run with `-e samba_manage_passwords=true`. From a
Tailnet client, connect to:

```text
smb://foundry/store
```

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
ansible-playbook --vault-password-file .vault-password playbooks/tailscale.yml
```

To advertise the Foundry LAN to Tailscale for Plex and other local services,
put the subnet route in the ignored private vars file:

```yaml
# ansible/group_vars/all/99-private.yml
foundry_tailscale_advertise_routes:
  - 192.168.1.0/24
```

Use the real CIDR for the Foundry network. The `tailscale` role enables Linux
IPv4 and IPv6 forwarding when advertised routes are configured, then passes the
routes to `tailscale up` as `--advertise-routes=...`.

After the playbook runs, approve the advertised route in the Tailscale admin
console before Tailnet clients can use it.

To pass extra flags to `tailscale up`, prefer `foundry_tailscale_extra_args`:

```sh
cd ansible
ansible-playbook playbooks/tailscale.yml \
  -e '{"foundry_tailscale_extra_args":["--ssh"]}'
```

When `foundry_tailscale_state` is `present`, the local `tailscale` role skips
package-manager work if `/usr/bin/tailscale` already exists. This avoids
recurring apt cache refreshes on an already-enrolled host. Set
`foundry_tailscale_state: latest` if you want a run to check for package
updates.

## Self-Managed (ansible-pull)

The `self_pull` role turns foundry into a self-managing host: a systemd timer
runs `ansible-pull` hourly against this repo and applies the same `foundry.yml`
that the Mac runs in push mode. Once it is set up, the Mac is no longer
required for routine converges - push mode is preserved only for
re-bootstrapping if foundry breaks badly.

The role installs:

- `/usr/local/bin/foundry-pull` - wrapper that fetches the repo, installs
  Galaxy collections, and runs `ansible-pull` with the pull-mode inventory.
- `/etc/systemd/system/foundry-pull.service` - oneshot unit that invokes the
  wrapper.
- `/etc/systemd/system/foundry-pull.timer` - 10 minutes after boot, then
  hourly, with a 5-minute randomized delay.

The wrapper expects two files that the role intentionally does **not**
manage, because they hold secrets and require interactive setup:

- `/etc/foundry/.vault-password` - vault password helper (mode `0700`,
  owned by root). Either an executable shebang script that prints the
  password, or a plain password file (mode `0400`).
- `/etc/foundry/private.yml` - same content as the Mac's
  `group_vars/all/99-private.yml` (mode `0600`, owned by root). Holds
  `foundry_users`, `foundry_user_emails`, advertise routes, etc.

### One-time bootstrap on foundry

1. Install 1Password CLI (optional - skip to step 3 if you prefer a plain
   password file). Add the 1Password apt repo, then:

   ```sh
   sudo apt install 1password-cli
   ```

   Authenticate non-interactively with a service account token. Either set
   `OP_SERVICE_ACCOUNT_TOKEN` in the helper's environment, or run
   `op signin` once and persist the session under `/root/.config/op/`.

2. Write `/etc/foundry/.vault-password` as an executable shebang script:

   ```sh
   sudo install -d -m 0750 -o root -g root /etc/foundry
   sudo tee /etc/foundry/.vault-password >/dev/null <<'EOF'
   #!/bin/sh
   exec op read "op://<vault>/<item>/<field>"
   EOF
   sudo chmod 0700 /etc/foundry/.vault-password
   ```

   **Fallback:** if 1Password CLI on a headless box is painful (no biometric
   unlock; service-account-only mode is rate-limited), drop the script and
   write the raw vault password to the same path with mode `0400`. The
   wrapper does not care which form is used.

3. Write `/etc/foundry/private.yml` with the same content as the Mac's
   `group_vars/all/99-private.yml`:

   ```sh
   sudo install -m 0600 -o root -g root \
     ~/group_vars-99-private.yml /etc/foundry/private.yml
   ```

4. From the Mac, run the converge one final time so the timer is installed:

   ```sh
   cd ansible
   ansible-playbook foundry.yml
   ```

   Or just the self-pull bits:

   ```sh
   cd ansible
   ansible-playbook playbooks/self-pull.yml
   ```

5. Confirm the timer is armed and trigger one manual run end-to-end:

   ```sh
   ssh foundry
   sudo /usr/local/bin/foundry-pull
   systemctl list-timers foundry-pull.timer
   sudo systemctl start foundry-pull.service
   journalctl -u foundry-pull.service -f
   ```

From then on the timer fires hourly. To verify pull mode is working
end-to-end, push a trivial commit and either wait for the next firing or
trigger `foundry-pull.service` manually - foundry should pick up and apply
the change without any action from the Mac.

### Pull-mode inventory

`ansible/inventory-pull.yml` pins the host to `localhost` over the local
connection plugin. The wrapper passes it explicitly, so push-mode
`inventory.yml` (with `ansible_host: foundry-admin`) is left untouched and
the Mac workflow keeps working.
