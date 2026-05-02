# Ansible

This directory contains the Ansible configuration for the local Ubuntu server
named `foundry`.

## Connectivity Check

Run Ansible from this directory so it picks up the repo-local `ansible.cfg`:

```sh
cd ansible
ansible foundry -m ansible.builtin.ping
```

The same check is also available as a playbook:

```sh
cd ansible
ansible-playbook playbooks/connectivity.yml
```

The inventory intentionally does not set `ansible_user`, so it follows the same
default user behavior as `ssh foundry`.
