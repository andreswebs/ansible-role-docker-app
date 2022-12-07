# ansible-role-docker-app

Installs a Docker Compose app as a systemd service.

## Role Variables

- `app_name`: Name applied to the app systemd service and dedicated user;
  optional; default: `docker-app`
- `app_src`: Path to app source on Ansible controller; required
- `app_dest`: Path do app destination on Ansible host; optional; default:
  `/opt/docker-app`
- `app_uid`: UID for the app user
- `app_gid`: GID for the app user

## Example Playbook

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

## Authors

**Andre Silva** [@andreswebs](https://github.com/andreswebs)

## License

[Unlicense](UNLICENSE.md)
