# The vault password from Bitwarden

[Environment variables and secrets](ansible-env.md) covers getting secrets into
a Compose stack, and recommends vaulted `group_vars` for `app_env`. That leaves
one secret the vault cannot hold: its own password. This document covers
sourcing it from Bitwarden instead of a plaintext file on disk.

Everything here belongs in the repository that *uses* the role, not in the role
itself. The role only cares that `app_env` arrives decrypted.

## The layering

```text
Bitwarden item  ->  ansible-vault password  ->  vault.yml  ->  app_env
```

Bitwarden holds exactly one secret, the vault password. The vault holds
everything else, encrypted, committed alongside the code. Nothing is stored in
plaintext outside a running Ansible process.

The alternative is to skip `ansible-vault` and read every secret from Bitwarden
directly with a lookup plugin. That removes a layer, at the cost of needing an
unlocked vault for every run including `--check`, and of no longer having the
secrets versioned with the code. Both shapes are defensible; this document
describes the first.

## Prerequisites

Install the CLI:

```sh
brew install bitwarden-cli        # or: npm install -g @bitwarden/cli
```

Log in once per workstation, and point at a self-hosted server first if you use
one:

```sh
bw config server "${BW_SERVER_URL}"
bw login
```

Create an item holding the vault password, then note its ID:

```sh
bw list items --search "ansible vault" | jq -r '.[] | "\(.id)  \(.name)"'
```

Use the **ID**, not the name. A rename in Bitwarden then cannot break a
deployment. The ID is a UUID and is inert without vault access, so it is safe
to commit.

## The client script

Save as `bin/vault-bw-client.sh` and make it executable:

```sh
chmod 0755 bin/vault-bw-client.sh
```

The `-client` suffix is load-bearing. Ansible treats a password file as a
client script only when its name ends in `-client` or `-client.EXT`, and then
invokes it with the requested label:

```text
bin/vault-bw-client.sh --vault-id live
```

Without that suffix the file is still executed and its stdout still used, but
no label is passed, so a single script cannot serve more than one vault.

```sh
#!/usr/bin/env bash
##
## ansible-vault password client script, backed by the Bitwarden CLI.
##
## Ansible's contract: print the password to stdout, diagnostics to stderr,
## and exit non-zero on failure. Nothing else may reach stdout.
##
set -euo pipefail

VAULT_ID="default"

while [ "$#" -gt 0 ]; do
  case "$1" in
    --vault-id) VAULT_ID="${2:-}"; shift 2 ;;
    --vault-id=*) VAULT_ID="${1#*=}"; shift ;;
    *) shift ;;
  esac
done

## Map each vault-id label to the item holding its password. Environment
## overrides keep the committed IDs from being the only way to point this
## somewhere else, which matters for a fork or a restored vault.
case "${VAULT_ID}" in
  live) ITEM_ID="${ANSIBLE_VAULT_BW_ITEM_LIVE:-REPLACE_WITH_ITEM_UUID}" ;;
  dev) ITEM_ID="${ANSIBLE_VAULT_BW_ITEM_DEV:-REPLACE_WITH_ITEM_UUID}" ;;
  *) ITEM_ID="${ANSIBLE_VAULT_BW_ITEM:-}" ;;
esac

if [ -z "${ITEM_ID}" ] || [ "${ITEM_ID}" = "REPLACE_WITH_ITEM_UUID" ]; then
  echo "vault-bw-client: no Bitwarden item mapped for vault-id '${VAULT_ID}'." >&2
  exit 2
fi

if ! command -v bw > /dev/null 2>&1; then
  echo "vault-bw-client: 'bw' not found. Install the Bitwarden CLI:" >&2
  echo "  brew install bitwarden-cli   # or: npm install -g @bitwarden/cli" >&2
  exit 2
fi

## Session resolution, in order of preference. This deliberately never prompts:
## Ansible may run it without a controlling terminal, and may run it more than
## once per playbook, so a prompt either hangs or repeats.
if [ -z "${BW_SESSION:-}" ] && [ -n "${BW_PASSWORD:-}" ]; then
  BW_SESSION="$(bw unlock --passwordenv BW_PASSWORD --raw 2> /dev/null || true)"
fi

if [ -z "${BW_SESSION:-}" ]; then
  echo "vault-bw-client: vault is locked and BW_SESSION is unset." >&2
  echo "Unlock once for this shell:" >&2
  echo '  export BW_SESSION="$(bw unlock --raw)"' >&2
  exit 2
fi

export BW_SESSION

## Opt-in only. A sync costs seconds, and this script runs at least once per
## playbook invocation. Set it after changing the item.
if [ "${ANSIBLE_VAULT_BW_SYNC:-0}" = "1" ]; then
  bw sync > /dev/null 2>&1 || true
fi

PASSWORD="$(bw get password "${ITEM_ID}" --raw --nointeraction 2> /dev/null || true)"

if [ -z "${PASSWORD}" ]; then
  echo "vault-bw-client: no password returned for item '${ITEM_ID}'." >&2
  echo "Check the item ID, and that the item has a password field. If the" >&2
  echo "item is newer than the local cache, retry with" >&2
  echo "ANSIBLE_VAULT_BW_SYNC=1." >&2
  exit 2
fi

printf '%s\n' "${PASSWORD}"
```

## Wiring it up

In `ansible.cfg`:

```ini
[defaults]
vault_identity_list = live@bin/vault-bw-client.sh
```

Relative paths here resolve against the working directory, not against
`ansible.cfg`. Running Ansible from the directory that holds `ansible.cfg` is
the usual arrangement and makes this work. From anywhere else, pass an absolute
path:

```sh
export ANSIBLE_VAULT_IDENTITY_LIST="live@${REPO_ROOT}/bin/vault-bw-client.sh"
```

## Daily use

Unlock once per shell. The session key is what the script consumes:

```sh
export BW_SESSION="$(bw unlock --raw)"
ansible-playbook playbooks/site.yml
```

With `direnv`, putting that export in a gitignored `.envrc` removes the step
entirely, at the cost of an unlocked vault for the life of the shell.

Encrypting and editing use the same label:

```sh
ansible-vault encrypt --encrypt-vault-id live group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
```

`edit` and `view` find the password through `vault_identity_list` without any
extra flags, so day-to-day use never mentions Bitwarden again.

## Unattended runs

`bw unlock` needs either an interactive prompt or a master password in the
environment, neither of which suits a build agent. Two options:

- Set `BW_PASSWORD` from the runner's own secret store. The script picks it up
  and unlocks without a prompt. Simple, but the master password is now a CI
  secret, which grants access to the entire personal vault.
- Use Bitwarden Secrets Manager instead. `bws` authenticates with a scoped
  machine-account token in `BWS_ACCESS_TOKEN` and needs no unlock step, which
  is what it exists for. Swap the `bw get password` line for the equivalent
  `bws secret get` call, and check the output format against the installed
  version before trusting a parse.

The second is the better shape for anything long-lived, because the token can
be scoped to one secret rather than standing in for the whole vault.

## What to commit

| Path | Commit | Why |
| ---- | ------ | --- |
| `bin/vault-bw-client.sh` | Yes | Holds no secret. Item IDs are inert without vault access. |
| `group_vars/**/vault.yml` | Yes | Encrypted at rest. This is the point of `ansible-vault`. |
| `group_vars/**/vault.yml.example` | Yes | Documents the required keys. |
| A plaintext password file | Never | Defeats the exercise. Add `.vault_pass` and similar to `.gitignore`. |

Git preserves the executable bit, so the script stays executable for whoever
clones next. A non-executable password file is read for its contents instead of
run, so Ansible would treat the script's own source as the password.

## Troubleshooting

| Symptom | Cause |
| ------- | ----- |
| `Decryption failed` on every file | The script returned the wrong password. Run it by hand and compare. |
| Ansible hangs at start | Something in the chain is prompting. The script must never prompt. |
| `vault is locked and BW_SESSION is unset` | Expected before `bw unlock`. Export the session and retry. |
| `no password returned for item` | Wrong ID, no password field, or a stale local cache. Retry with `ANSIBLE_VAULT_BW_SYNC=1`. |
| Password works interactively, fails in CI | `BW_SESSION` is per-machine and not portable. Use `BW_PASSWORD` or `bws`. |
| Works from the repo root, fails elsewhere | The relative path in `ansible.cfg`. Use `ANSIBLE_VAULT_IDENTITY_LIST` with an absolute path. |

Run the script directly to separate its failures from Ansible's:

```sh
bin/vault-bw-client.sh --vault-id live
```

It should print one line and exit `0`.
