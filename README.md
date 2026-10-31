# Ansible SSH Key Distribution

Let one host log in to other hosts with an SSH key — set up entirely by Ansible, without typing a single password on any host.

Ansible already has trusted access to every managed host (SSH key of the control node plus `become` with the password from Ansible Vault). These playbooks use that trust to:

1. **generate an SSH key pair on a host you choose** (or reuse the existing one),
2. **authorize its public key on all other hosts** (or a group),
3. **write the targets' SSH host keys into the source host's `known_hosts`**, so the first `ssh target` asks no fingerprint question,
4. **verify** that the source host can log in to every target.

Afterwards `ssh user@target` works from the source host immediately — no password, no `yes` to a fingerprint.

The host keys are taken from Ansible facts, i.e. over the already trusted Ansible connection. This is safer than accepting unknown keys on first contact (`StrictHostKeyChecking=accept-new`).

## Requirements

- `ansible-core` on the control node and an inventory in which Ansible can reach all hosts with `become`. The [ansible-inventory-generator](https://github.com/Michdo93/ansible-inventory-generator) creates exactly that (SSH keys + become password in Ansible Vault).
- Linux targets: only `ansible-core` modules are used.
- Windows targets (OpenSSH Server): the `ansible.windows` collection:

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

## Usage

With the inventory of `ansible-inventory-generator`:

```bash
export ANSIBLE_CONFIG=/path/to/inventory/ansible.cfg

# pi-lab gets a key pair; every other host in the inventory accepts it
ansible-playbook distribute.yml -e source_host=pi-lab

# only hosts of the group "lab", key goes to the account "sysadmin" there,
# and is only accepted from pi-lab's address
ansible-playbook distribute.yml -e source_host=pi-lab -e target_hosts=lab \
  -e target_user=sysadmin -e restrict_from=true

# undo: remove pi-lab's key from the targets again (the key pair itself stays)
ansible-playbook revoke.yml -e source_host=pi-lab -e target_hosts=lab -e target_user=sysadmin
```

### Other accounts than the Ansible account

Ansible logs in as its own account (e.g. `sysadmin`) and works as root via `become`. Root can write into every home directory, so the key pair can belong to any account on the source host and the public key can go to any account on the targets — the Ansible account itself is not involved.

```bash
# openhab's key pair (created in /var/lib/openhab/.ssh), accepted by "pi" on every target
ansible-playbook distribute.yml -e source_host=openhab-vm -e source_user=openhab -e target_user=pi
```

A system account with `/usr/sbin/nologin` as shell (like `openhab`) works as source: outgoing `ssh` needs no login shell. Target accounts need a real shell.

**Different or several accounts per target (optional).** Without `-e target_user`, each target uses its own `key_targets` from `host_vars`/`group_vars` (or its `ansible_user` if it has none):

```yaml
# inventory/host_vars/pi-lab/keys.yml
key_targets: [pi, mqtt]
```

```yaml
# inventory/group_vars/nas.yml
key_targets: backup
```

```bash
ansible-playbook distribute.yml -e source_host=openhab-vm -e source_user=openhab -e skip_missing_users=true
ansible-playbook revoke.yml     -e source_host=openhab-vm -e source_user=openhab
```

Order of precedence per target: `-e target_user` → the host's `key_targets` → its `ansible_user`. Do not put `key_targets` into `defaults.yml`: that file is loaded with `vars_files` and would override all host-specific values.

Without a stored Vault password add `--ask-vault-pass`. Use any other inventory with `-i`.

`source_host` is required; the playbooks fail with "Keyword 'hosts' is required" if it is missing.

## Variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `source_host` | – (required) | Inventory name of the host that gets the key pair |
| `source_user` | `ansible_user` of the source host | Account that owns the key pair |
| `key_type` | `ed25519` | `ed25519`, `ecdsa` or `rsa` |
| `key_file_name` | `id_<key_type>` | File in `~source_user/.ssh/`; an existing key is reused |
| `key_comment` | `<user>@<host> (ansible-ssh-key-distribution)` | Comment of a newly generated key |
| `target_hosts` | `all` | Inventory pattern of the receiving hosts; the source is always excluded |
| `target_user` | `ansible_user` of each target | Account on **all** targets that accepts the key |
| `key_targets` | – (optional, per host) | One or more accounts on **this** target; set in `host_vars`/`group_vars`, used when `target_user` is not given |
| `skip_missing_users` | `false` | Skip accounts that do not exist on a target instead of failing |
| `restrict_from` | `false` | Prefix the key with `from="<source ansible_host>"` |
| `distribute_known_hosts` | `true` | Add the targets' host keys to the source user's `known_hosts` |
| `verify_logins` | `true` | Test `ssh -o BatchMode=yes` from the source to every target |

All defaults live in `defaults.yml`; override them with `-e`.

## What happens

| Step | Linux | Windows |
| --- | --- | --- |
| Key pair on the source | `ssh-keygen` only if the key does not exist; a missing `.pub` is derived from the private key | (source must be Linux) |
| Authorize on targets | `~target_user/.ssh/authorized_keys`, created with mode 0600; a line with the same key is replaced, so re-runs are idempotent | Administrators: `C:\ProgramData\ssh\administrators_authorized_keys`; other users: `.ssh\authorized_keys` in their profile; ACLs are set as OpenSSH requires |
| Host keys for `known_hosts` | from facts (`ansible_ssh_host_key_*_public`) | read from `C:\ProgramData\ssh\ssh_host_*_key.pub` |
| Address in `known_hosts` | `ansible_host` (or `[ansible_host]:port` for non-22 ports) — the same address the source uses to connect | same |

Only a small record per target (user, address, port, host keys) is passed back to the source host; host variables such as vaulted passwords never appear in the output.

## Security

Distributing a key creates **transitive trust**: whoever controls the source host (or its private key) can log in to every target as `target_user`.

- Prefer a dedicated, unprivileged `target_user` over `root`.
- Use `restrict_from=true` so the key only works from the source host's address.
- Keep the source key in a dedicated file (`-e key_file_name=id_ed25519_backup`) when it serves one purpose, so it can be revoked on its own.
- `revoke.yml` removes the key from all targets; delete the key pair on the source host if it is no longer needed.

## Repository layout

```text
ansible-ssh-key-distribution/
├── defaults.yml               default variables
├── distribute.yml             generate/reuse key, authorize, known_hosts, verify
├── revoke.yml                 remove the key from the targets
├── requirements.yml           ansible.windows (Windows targets only)
└── tasks/
    ├── linux_authorize.yml          loops over the target accounts
    ├── linux_authorize_user.yml
    ├── linux_revoke.yml
    ├── linux_revoke_user.yml
    ├── windows_authorize.yml
    ├── windows_authorize_user.yml
    ├── windows_revoke.yml
    └── windows_revoke_user.yml
```

## Tested

Linux: generating a new key, reusing an existing one, a different `target_user` (written via `become`), a `nologin` system account (`openhab`) as source, per-host `key_targets` with several accounts, `skip_missing_users` (missing account skipped, without it the run fails), `restrict_from`, `known_hosts` (login with `BatchMode=yes` without any prompt), idempotent re-runs (`changed=0`) and `revoke.yml` (login refused afterwards). The Windows tasks are syntax-checked but not yet tested on a Windows host.
