# Ansible

This directory contains the Ansible configuration for the local Ubuntu server
named `foundry`.

## Bootstrap

The inventory uses the dedicated `ansible` user for normal provisioning runs.
Bootstrap that user from an already provisioned account through the
`foundry_bootstrap` inventory alias:

```sh
cd ansible
ansible-playbook playbooks/bootstrap.yml --ask-become-pass
```

The `foundry_bootstrap` alias connects as `josh`. If that account already has
passwordless sudo, omit `--ask-become-pass`.

The bootstrap playbook expects the Ansible provisioning user's public key at
`~/.ssh/ansible.pub`. SSH authentication is expected to use your local SSH
configuration or SSH agent, such as the 1Password SSH agent.

## Connectivity Check

After bootstrap, run Ansible from this directory so it picks up the repo-local
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

## User Configuration

Configure the human users, passwordless sudo, SSH access from
`~/.ssh/foundry.pub`, their home-directory `code` folders, shared
`xterm-ghostty` terminfo, zsh, and Oh My Zsh after bootstrap:

```sh
cd ansible
ansible-playbook playbooks/users.yml
```
