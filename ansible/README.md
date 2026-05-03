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
