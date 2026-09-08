# Environment variables and secrets

The role offers two mutually exclusive ways to get environment variables into a
Compose stack. Pick one; setting both is rejected.

## Option 1: `app_env` (rendered by the role)

Set `app_env` to a key/value map, typically from vaulted `group_vars` or
`host_vars`. The role renders it to `app_env_file` and passes that path to
Compose as `--env-file`.

```yaml
app_env:
  DATABASE_PASSWORD: !vault |
    $ANSIBLE_VAULT;1.2;AES256;prod
    36656464...
```

Properties:

- Written at mode `0400`, owned by the app user.
- The task carries `no_log`, so values do not appear in output or `--diff`.
- Values are reproducible from the repository plus the vault, so the host holds
  no unrecoverable state.

## Option 2: a `.env` shipped inside `app_src`

Leave `app_env` empty and place a `.env` file alongside the compose file in a
**directory** `app_src`. The role copies the tree verbatim, and Compose reads
`.env` from the project directory on its own.

This works because for a directory `app_src` the project directory is
`app_dest`, which is exactly where the copy lands. It does **not** apply to a
file `app_src`, whose project directory is `app_home` instead.

Properties:

- The file inherits `0750` from the `copy` task, not `0400`.
- No `no_log`, so a run with `--diff` can echo secret values.
- The file becomes the only copy of those secrets unless you store it
  elsewhere. An instance replacement loses them.

## Why both at once is refused

When `app_env` is set and `app_src` is a directory, the role asserts that the
source tree does not already contain a `.env`:

```text
'app_env' is set, but 'app_src' already contains a .env that would be
overwritten at {{ app_env_file }}. Move those values into 'app_env' or drop
the file.
```

Both paths target the same destination, so one would silently overwrite the
other. Failing is preferable to picking a winner.

## If the compose file declares `env_file`

A stack whose compose file contains an explicit `env_file:` entry makes the
file mandatory rather than optional. With `app_env` empty and no shipped
`.env`, nothing supplies it and the stack is invalid. This surfaces early, at
the role's `docker compose config` validation step, before the unit is written,
rather than as an `ExecStart` failure in the journal.

Set `app_validate: false` only if the referenced path genuinely exists only at
runtime.

## Converting a shipped `.env` to `app_env`

A common path is to start with a hand-filled `.env` and move to vaulted values
later.

1. Put the same keys and values into vaulted `group_vars` or `host_vars`.
2. Set `app_env` to that map.
3. **Delete the `.env` from `app_src`.** The assert above will otherwise refuse
   to run. That guardrail is the conversion checklist.
4. Run the play.

No stale file is left on the host. `app_env_file` defaults to
`{{ app_dest }}/.env`, the same path the copy was using, so the template
overwrites it in place and the mode tightens from `0750` to `0400`.

### The conversion restarts the stack

Expect an interruption, not an in-place reconcile.

`--env-file` is conditional in the unit template, present only when `app_env`
is non-empty. So adding `app_env` changes the unit file itself, which notifies
`Restart app`. On a `RemainAfterExit` oneshot, `systemctl restart` runs
`ExecStop` first, taking the whole stack down and back up.

Bind-mounted state is untouched, so nothing is at risk, but do not schedule the
change mid-session.

For contrast, changing only the _contents_ of the env file triggers a reload,
which reconciles in place and recreates just the containers whose configuration
changed. Note that a converted file's content differs from the copied one even
with identical values, because the template emits keys sorted by `dictsort` and
drops comments. That difference alone would be a reload; it is the unit change
that escalates it to a restart.

## Callers shipping a `.env`

Add the file to `.gitignore` in the calling repository. It sits beside a
committed compose file, and an example file such as `env.example` is the usual
companion. Note that `env.example` has no leading dot, so the two names do not
collide.
