# Ansible Role: quadlet

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-quadlet) ![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-quadlet) ![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-quadlet) [![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-quadlet/dev-push-smoke.yml?branch=dev&event=push&label=dev)](https://github.com/jomrr/ansible-role-quadlet/actions/workflows/dev-push-smoke.yml?query=branch%3Adev) [![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-quadlet/main-full-gate.yml?branch=main&event=push&label=main)](https://github.com/jomrr/ansible-role-quadlet/actions/workflows/main-full-gate.yml?query=branch%3Amain)

Ansible role for deploying rootless Podman Quadlet application services.

## Purpose

This role deploys small application services as rootless Podman Quadlets.
Each service runs under a dedicated Unix service user, while the generated
Quadlet files are owned by root below `/etc/containers/systemd/users/<UID>/`.

The role separates non-secret application configuration from secrets:
EnvironmentFiles contain only non-secret values, while sensitive values are
referenced through rootless `podman secret` entries in the service user's
Podman context.

## Scope

### Managed

- Podman package installation when `quadlet_manage_packages` is enabled
- Dedicated Unix service users and matching primary groups
- Root-owned user Quadlet directories below `/etc/containers/systemd/users/<UID>/`
- Root-owned Quadlet `.container` files for rootless user services
- Non-secret EnvironmentFiles below `/srv/containers/<container>/env/` by default
- Default application data directories below `/srv/containers/<container>/data`
- Rootless Podman secrets when `value` or `value_file` is supplied
- Systemd linger for users with enabled services
- Generated user service enablement and runtime state

### Not Managed

- Rootful or system-wide Quadlets below `/etc/containers/systemd/`
- Secrets stored in Quadlet files, EnvironmentFiles, defaults, or templates
- Container image building or image publishing
- Firewall policy
- Reverse proxy or TLS certificate lifecycle
- Application-specific database provisioning
- Purging unmanaged Quadlet files or Podman secrets

## Requirements

- Target hosts need Podman with Quadlet support and systemd user services.
- Rootless Podman secret creation requires a working rootless Podman context for each service user.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
```

## Role Variables

The following variables are part of the public role interface.

| Name | Type | Required | Default | Description |
| ---- | ---- | -------- | ------- | ----------- |
| `quadlet_packages` | `list` | `false` | - podman | Platform packages required for Podman and Quadlet support. |
| `quadlet_manage_packages` | `bool` | `false` | `True` | Whether the role installs `quadlet_packages`. |
| `quadlet_users` | `list` | `false` | [] | Service users and rootless containers managed by this role. |

## Managed Files

- `/etc/containers/systemd/users/<UID>/<container>.container` root-owned rootless user Quadlet
- `/srv/containers/<container>/env/<container>.env` non-secret EnvironmentFile
- `/srv/containers/<container>/data` default application data directory

## Check Mode

File, template, user, group, package, and systemd tasks follow the check-mode
behavior of their underlying Ansible modules. Rootless Podman secret
creation uses `podman secret` commands and is not a pure file operation.

## Service Behavior

Generated services are user services named `<name>.service` from
`<name>.container`. When a Quadlet changes and service management is active,
the role reloads the service user's systemd manager through the
`ansible.builtin.systemd_service` module with `scope: user`.

## Security Notes

- Rootless Podman reduces runtime privileges by running application containers in a non-root user namespace owned by the dedicated service user.
- Quadlet files are written below `/etc/containers/systemd/users/<UID>/` with owner `root`, group `root`, and mode `0644`, so the service user cannot modify the unit definition.
- The role does not use `/etc/containers/systemd/` for managed containers; that path is reserved for rootful or system-wide Quadlets.
- EnvironmentFiles are only for non-secret application configuration and are written owner `root`, group service-user, mode `0640`.
- Secret values are never rendered into Quadlet files or EnvironmentFiles by the role.
- Podman secrets with `type: mount` expose a secret as a file below `/run/secrets/` and are preferred over environment-variable secrets.
- Mounted Podman secrets reduce accidental leak surfaces compared with ENV secrets, but they do not protect against a full compromise of the container process.
- Podman secrets with `type: env` are supported only as an explicit fallback for applications that cannot consume file-based secrets.
- Existing Podman secrets are not overwritten unless the secret item sets `recreate: true`.
- Docker Compose-specific escaping rules for `$` are not applied to EnvironmentFiles or Quadlet templates.

## Operational Notes

- Use file-based application settings such as `*_FILE=/run/secrets/<secret>` whenever the application supports them.
- `type: mount` renders `Secret=<name>` or `Secret=<name>,target=<target>` and lets Podman mount the secret as a file.
- `type: env` renders `Secret=<name>,type=env,target=<ENV_NAME>` and exposes the secret through the container environment.
- EnvironmentFile entries render ordinary `KEY="value"` settings and must not contain passwords, tokens, API keys, or private material.
- If neither `value` nor `value_file` is set for a secret, the role only references the secret in the Quadlet and assumes it already exists in the service user's rootless Podman context.
- Set `state: created` and `enabled: false` for file-only rendering without user service management.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Leap | latest | [jomrr/molecule-opensuse-leap:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-leap) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### Vikunja with mounted Podman secrets

Vikunja can consume file-based secrets. The EnvironmentFile contains
only `_FILE` references, and the Quadlet contains `Secret=` references.

```yaml
---
- name: Deploy Vikunja as a rootless Quadlet
  hosts: quadlet
  gather_facts: true
  roles:
    - role: jomrr.quadlet
      vars:
        quadlet_users:
          - name: vikunja
            uid: 24010
            containers:
              - name: vikunja
                image: docker.io/vikunja/vikunja:latest
                container_name: vikunja
                environment:
                  VIKUNJA_DATABASE_TYPE: postgres
                  VIKUNJA_DATABASE_HOST: 127.0.0.1
                  VIKUNJA_DATABASE_PASSWORD_FILE: /run/secrets/vikunja_db_password
                  VIKUNJA_SERVICE_SECRET_FILE: /run/secrets/vikunja_service_secret
                secrets:
                  - name: vikunja_db_password
                    value_file: /root/ansible-secrets/vikunja_db_password
                  - name: vikunja_service_secret
                    value_file: /root/ansible-secrets/vikunja_service_secret
                volumes:
                  - /srv/containers/vikunja/data:/app/vikunja/files:Z
                ports:
                  - 127.0.0.1:3456:3456
```
### Vaultwarden ADMIN_TOKEN as explicit env fallback

Use `type: env` only when a clean file-based variant is not available for
the application setting. Prefer a `value_file` or encrypted inventory
source over plaintext values.

```yaml
---
- name: Deploy Vaultwarden as a rootless Quadlet
  hosts: quadlet
  gather_facts: true
  roles:
    - role: jomrr.quadlet
      vars:
        quadlet_users:
          - name: vaultwarden
            uid: 24020
            containers:
              - name: vaultwarden
                image: docker.io/vaultwarden/server:latest
                container_name: vaultwarden
                environment:
                  DOMAIN: https://vault.example.org
                  SIGNUPS_ALLOWED: "false"
                secrets:
                  - name: vaultwarden_admin_token
                    type: env
                    env_target: ADMIN_TOKEN
                    value_file: /root/ansible-secrets/vaultwarden_admin_token
                volumes:
                  - /srv/containers/vaultwarden/data:/data:Z
                ports:
                  - 127.0.0.1:8080:80
```

## References

- [Podman Quadlet documentation](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
- [Podman secrets documentation](https://docs.podman.io/en/latest/markdown/podman-secret.1.html)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
