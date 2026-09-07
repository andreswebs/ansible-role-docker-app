# ansible-role-docker-app

Installs a Docker Compose app as a systemd service.

## Role Variables

### Required

- `app_src`: Absolute path on the Ansible controller to either a directory
  containing a compose file, or a single compose file. Relative paths are not
  accepted; see [Compose source layout](#compose-source-layout).

### App identity

- `app_name` (default: `docker-app`) - name applied to the systemd unit, the
  dedicated user, (by default) the home/group, and the Compose **project name**.
  Must match `^[a-z0-9][a-z0-9_-]*$`, the character set Compose accepts in a
  project name.
- `app_user` (default: `{{ app_name }}`)
- `app_group` (default: `{{ app_name }}`)
- `app_uid` (default: `2000`)
- `app_gid` (default: `2000`)
- `app_user_overwrite` (default: `false`) - pass `non_unique` to `user`/`group` modules.
- `app_group_overwrite` (default: `false`)
- `app_user_shell` (default: `/bin/bash`)

### Paths

- `app_dest` (default: `/opt/{{ app_name }}`) - where the compose source is copied.
- `app_home` (default: `/var/lib/{{ app_name }}`) - app user's home directory.
- `app_home_decouple` (default: `true`) - create `app_home` as a separate dir owned
  by the app user. When `false`, the user's home is created in-place by the `user`
  module (no separate file task).

### Compose payload

- `app_compose_file` (default: unset) - overrides the compose-file probe for a
  directory `app_src` whose compose file is not one of the names Compose
  recognises. Ignored for a file `app_src`.
- `app_validate` (default: `true`) - run `docker compose config` on the deployed
  payload before writing the unit. Turn off only for a stack whose `env_file:`
  points at a path that exists only at runtime.

### Secrets and configuration

- `app_env` (default: `{}`) - key/value pairs rendered to an env file and passed
  to Compose as `--env-file`. Expected to arrive from vaulted `group_vars` or
  `host_vars`.
- `app_env_file` (default: `{{ app_dest }}/.env`) - where that file is written.

### Runtime

- `app_started` (default: `true`) - start the service now.
- `app_enabled` (default: `true`) - enable the service at boot.
- `app_requires` (default: `[]`) - other unit names added to the unit's
  `Requires=` and `After=`, for a stack that references another stack's
  resources by name (e.g. a network declared `external: true` elsewhere).
- `app_stop_timeout` (default: `120s`) - the unit's `TimeoutStopSec`.
- `docker_bin` (default: `/usr/bin/docker`) - path to the `docker` binary baked
  into the systemd unit. Checked for existence before the unit is written.

## Compose source layout

`app_src` must be an **absolute** path. A relative path cannot be classified
reliably: Jinja's `is directory` test resolves it against the Ansible process's
working directory, while the `copy` module resolves it against `files/` and then
the playbook directory. Those are different places, and the role needs the
answer to decide what to copy where.

The two accepted shapes behave differently, mirroring what Compose itself does:

| `app_src`   | Installed as                              | Project directory | Use for                                           |
| ----------- | ----------------------------------------- | ----------------- | ------------------------------------------------- |
| a directory | its **contents**, verbatim, in `app_dest` | `app_dest`        | stacks shipping config next to the compose file   |
| a file      | `{{ app_dest }}/compose.yaml`             | `app_home`        | a self-contained compose file with nothing beside |

A trailing slash on a directory `app_src` is optional; the role normalises it
either way. Without that normalisation, `copy` would place the directory
*itself* inside `app_dest`, leaving the compose file one level below the unit's
`WorkingDirectory`.

For a directory `app_src`, the role looks for a compose file at the root of
`app_dest`, in Compose's own preference order:

```text
compose.yaml
compose.yml
docker-compose.yaml
docker-compose.yml
```

The first match becomes the unit's explicit `--file` argument. Set
`app_compose_file` for anything outside that set. If nothing matches, the role
fails at deploy time naming `app_src`, rather than leaving the unit to fail at
start.

For a file `app_src`, the payload is installed as `compose.yaml` regardless of
its source name, so the unit's `--file` argument is predictable.

Because `copy` does not purge, switching an existing stack between the two
shapes leaves the old payload behind, and the probe may prefer it. Clear
`app_dest` when changing shape.

### Project directory

Relative bind sources in a compose file (`./data:/data`) resolve against the
project directory, which is not the same in both shapes.

A **directory** `app_src` keeps the default project directory, `app_dest`. Its
whole point is bundling supporting files next to the compose file, and
`./config/nginx.conf` has to keep resolving where those files were copied.

A **file** `app_src` gets `--project-directory {{ app_home }}`, so relative
binds land in the app's own state directory instead of being mixed into the
copied source. `$STATE_DIRECTORY` is exported to the unit in both shapes, so a
compose file can always reference `app_home` explicitly.

`app_home` is load-bearing in a third way: because the unit sets `User=`,
systemd sets `HOME` from passwd, so that is also where `~/.docker` and any
other home-relative writes land.

## Secrets

Setting `app_env` renders an env file to `app_env_file`, owned by the app user
at mode `0400`, with `no_log` on the task. Encrypt the values with
`ansible-vault` in `group_vars`/`host_vars`:

```yaml
app_env:
  POSTGRES_PASSWORD: !vault |
    $ANSIBLE_VAULT;1.2;AES256;prod
    36656464...
```

Two limitations worth stating plainly:

- The rendered file is on disk, not tmpfs. This is better than cleartext
  committed next to the compose file, and weaker than a secret manager that
  never materialises plaintext on a persistent filesystem.
- `--env-file` **replaces** Compose's default `.env` loading rather than adding
  to it. The default `app_env_file` is `{{ app_dest }}/.env`, which for a
  directory `app_src` is the same path Compose would have read on its own, so
  this only becomes a distinction if you move it. A source tree shipping its own
  `.env` alongside a set `app_env` is rejected rather than silently overwritten.

## Docker group caveat

The app user is added to the `docker` group. That is root-equivalence: anyone
who can talk to the Docker socket can start a privileged container. The
dedicated user buys ownership hygiene for bind-mounted state, not a privilege
boundary.

## Runtime gating model

`app_started` and `app_enabled` are independent. Together they describe four
deployment shapes:

| `app_started` | `app_enabled` | Result                                                      |
| ------------- | ------------- | ----------------------------------------------------------- |
| `true`        | `true`        | **Default.** Service is running now and autostarts on boot. |
| `true`        | `false`       | Running now, no autostart. Useful for transient workloads.  |
| `false`       | `true`        | Stopped now, will start on next boot.                       |
| `false`       | `false`       | Service unit is installed but never touched by Ansible.     |

The role separates *deploying the unit* from *managing the unit's runtime*. Both
flags being `false` is a deliberate "install only" mode - the unit file is written
and `daemon-reload` is run, but the service itself is left in whatever state it's
in. Useful for staged rollouts where another tool, a human, or a later play
decides when to start the workload.

### Restart-on-change behavior

Changes are applied at the smallest scope that works, and only if the service is
**currently active** on the host. The handlers check real systemd state via
`systemctl`, not the role's variables.

| What changed                  | What happens                                        |
| ----------------------------- | --------------------------------------------------- |
| Compose payload, or `app_env` | `systemctl reload` - `compose up --detach` in place |
| The unit file itself          | `daemon-reload` + `systemctl restart`               |
| Both, in the same run         | Restart only; it supersedes the reload              |
| Anything, service not active  | Nothing                                             |

The reload path matters because the unit is a `RemainAfterExit` oneshot:
`systemctl restart` runs `ExecStop` first, taking the whole stack down and back
up for a one-line YAML edit. `ExecReload` runs the same
`compose up --detach --remove-orphans` as `ExecStart`, which recreates only the
containers whose configuration actually changed.

Consequences:

- Default flow (`app_started: true`): config changes reconcile in place.
- Install-only flow (`app_started: false`, `app_enabled: false`): nothing is
  touched - Ansible respects the "don't touch runtime" contract.
- Manual-start edge case: if a human ran `systemctl start <app>` outside
  Ansible, the next role run that updates the compose file *will* act on it.
  This is intentional - stale code running against fresh config is the worse
  outcome.

## Project name

The unit passes `--project-name {{ app_name }}` explicitly, on `up`, `down` and
reload alike. Without it, Compose derives the project name from the working
directory's basename, and `down` addressing a different project than `up` is a
silent no-op.

If you are upgrading a stack deployed by an older version of this role **and**
its `app_dest` basename differs from its `app_name`, the running containers
belong to a project named after the basename. Tear them down under the old name
before the first upgraded run:

```sh
cd "${APP_DEST}" && docker compose down
```

Where the two already match - which is the case for every example below - this
is a no-op and no migration is needed.

## Example playbooks

### Default - deploy and run

```yaml
- hosts: servers
  roles:
    - role: andreswebs.docker_app
      vars:
        app_name: jenkins
        app_src: "{{ playbook_dir }}/files/jenkins/"
        app_dest: /opt/jenkins
        app_uid: 1000
        app_gid: 1000
```

### With secrets

```yaml
- hosts: servers
  roles:
    - role: andreswebs.docker_app
      vars:
        app_name: jenkins
        app_src: "{{ playbook_dir }}/files/jenkins/"
        app_env: "{{ jenkins_secrets }}" # from vaulted group_vars
```

### Deploy without running

Install the unit file, drop the compose source in place, but don't start or
enable the service. Useful when you want to inspect or pre-stage hosts before
flipping them live.

```yaml
- hosts: servers
  roles:
    - role: andreswebs.docker_app
      vars:
        app_name: jenkins
        app_src: "{{ playbook_dir }}/files/jenkins/"
        app_started: false
        app_enabled: false
```

A later run with no overrides (or `-e app_started=true -e app_enabled=true`)
brings the service up.

### Run now, don't autostart on boot

For services that should be brought up by a higher-level orchestrator, not by
the host's boot sequence:

```yaml
- hosts: servers
  roles:
    - role: andreswebs.docker_app
      vars:
        app_name: jenkins
        app_src: "{{ playbook_dir }}/files/jenkins/"
        app_enabled: false
```

## Dependencies

- [andreswebs.docker](https://github.com/andreswebs/ansible-role-docker) - installs
  Docker Engine and the `docker compose` plugin on the target host.

## Authors

**Andre Silva** - [@andreswebs](https://github.com/andreswebs)

## License

[Unlicense](UNLICENSE)
