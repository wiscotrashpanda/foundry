# Foundry

## How Configuration Updates Work

This machine manages itself with `ansible-pull`. A systemd timer runs
`foundry-pull` automatically every 30 minutes, with an additional run scheduled
shortly after boot.

To apply a configuration change, commit and push the change to the `foundry`
repo. The machine will pull and apply it on the next timer cycle. To apply a
change immediately without waiting, SSH into the machine and run
`foundry-pull`.

Encrypted secrets use `ansible-vault`. The vault password lives at
`/etc/ansible/vault-pass` on the managed machine and is never stored in this
repository. To update a vaulted value, edit it locally with
`ansible-vault edit <file>`, then commit and push the encrypted change. The
machine will decrypt it automatically on the next pull using its local password
file.

The private inventory and any unvaulted host-specific configuration are set up
once manually on the machine and are not managed by this repo.

If the machine needs to be rebuilt from scratch, run `bootstrap.yml` from a
local machine with Ansible installed. This bootstrap run is the only time local
Ansible is required.
