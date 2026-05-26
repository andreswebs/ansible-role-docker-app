# ansible-role-docker-app

Installs a Docker Compose app as a systemd service.

## Role Variables

### Required

- `app_src`: Path to app source on the Ansible controller (a directory containing
  `docker-compose.yml`).

### App identity

- `app_name` (default: `docker-app`) — name applied to the systemd unit, the
  dedicated user, and (by default) the home/group. Must match `^[a-z0-9][a-z0-9._-]*$`.
- `app_user` (default: `{{ app_name }}`)
- `app_group` (default: `{{ app_name }}`)
- `app_uid` (default: `2000`)
- `app_gid` (default: `2000`)
- `app_user_overwrite` (default: `false`) — pass `non_unique` to `user`/`group` modules.
- `app_group_overwrite` (default: `false`)
- `app_user_shell` (default: `/bin/bash`)

### Paths

- `app_dest` (default: `/opt/{{ app_name }}`) — where the compose source is copied.
- `app_home` (default: `/var/lib/{{ app_name }}`) — app user's home directory.
- `app_home_decouple` (default: `true`) — create `app_home` as a separate dir owned
  by the app user. When `false`, the user's home is created in-place by the `user`
  module (no separate file task).

### Runtime

- `app_started` (default: `true`) — start the service now.
- `app_enabled` (default: `true`) — enable the service at boot.
- `docker_bin` (default: `/usr/bin/docker`) — path to the `docker` binary baked
  into the systemd unit.

## Runtime gating model

`app_started` and `app_enabled` are independent. Together they describe four
deployment shapes:

| `app_started` | `app_enabled` | Result                                                       |
|---------------|---------------|--------------------------------------------------------------|
| `true`        | `true`        | **Default.** Service is running now and autostarts on boot.  |
| `true`        | `false`       | Running now, no autostart. Useful for transient workloads.   |
| `false`       | `true`        | Stopped now, will start on next boot.                        |
| `false`       | `false`       | Service unit is installed but never touched by Ansible.      |

The role separates *deploying the unit* from *managing the unit's runtime*. Both
flags being `false` is a deliberate "install only" mode — the unit file is written
and `daemon-reload` is run, but the service itself is left in whatever state it's
in. Useful for staged rollouts where another tool, a human, or a later play
decides when to start the workload.

### Restart-on-change behavior

When `app_src` content or the service template changes, a restart is *only*
performed if the service is **currently active** on the host. The handler checks
real systemd state via `systemctl`, not the role's variables. Consequences:

- Default flow (`app_started: true`): service is running, so config changes
  produce a rolling restart.
- Install-only flow (`app_started: false`, `app_enabled: false`): if the service
  isn't running, no restart fires — Ansible respects the "don't touch runtime"
  contract.
- Manual-start edge case: if a human ran `systemctl start <app>` outside Ansible,
  the next role run that updates the compose file *will* restart the service.
  This is intentional — stale code running against fresh config is the worse
  outcome.

## Example playbooks

### Default — deploy and run

```yaml
- hosts: servers
  roles:
    - role: andreswebs.docker_app
      vars:
        app_name: jenkins
        app_src: jenkins-docker
        app_dest: /opt/jenkins
        app_uid: 1000
        app_gid: 1000
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
        app_src: jenkins-docker
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
        app_src: jenkins-docker
        app_enabled: false
```

## Dependencies

- [andreswebs.docker](https://github.com/andreswebs/ansible-role-docker) — installs
  Docker Engine and the `docker compose` plugin on the target host.

## Authors

**Andre Silva** - [@andreswebs](https://github.com/andreswebs)

## License

[Unlicense](UNLICENSE)
