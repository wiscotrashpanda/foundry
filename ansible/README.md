# Ansible

This directory contains the Ansible configuration for the local Ubuntu server
named `foundry`.

## Bootstrap

The inventory uses the dedicated `ansible` user and the local
`~/.ssh/foundry_ansible` key for normal provisioning runs. Bootstrap that user
from an already provisioned account through the `foundry_bootstrap` inventory
alias:

```sh
cd ansible
ansible-playbook playbooks/bootstrap.yml --ask-become-pass
```

The `foundry_bootstrap` alias connects as `josh`. If that account already has
passwordless sudo, omit `--ask-become-pass`.

The bootstrap playbook expects the public key at `~/.ssh/foundry_ansible.pub`.
If you only have the private key, create the public key first:

```sh
ssh-keygen -y -f ~/.ssh/foundry_ansible > ~/.ssh/foundry_ansible.pub
```

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
