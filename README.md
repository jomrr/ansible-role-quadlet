# Ansible Role: quadlet

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-quadlet) ![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-quadlet) ![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-quadlet) [![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-quadlet/dev.yml?branch=dev&event=push&label=dev)](https://github.com/jomrr/ansible-role-quadlet/actions/workflows/dev.yml?query=branch%3Adev) [![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-quadlet/main.yml?branch=main&event=push&label=main)](https://github.com/jomrr/ansible-role-quadlet/actions/workflows/main.yml?query=branch%3Amain)

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

- Podman package installation
- Dedicated Unix service users and matching primary groups
- Subordinate UID/GID allocation for newly created service users through system shadow-utils defaults
- Root-owned user Quadlet directories below `/etc/containers/systemd/users/<UID>/`
- Root-owned Quadlet `.pod` files for rootless Podman pods
- Root-owned Quadlet `.container` files for rootless user services
- Non-secret EnvironmentFiles below `/srv/containers/<container>/env/` by default
- Default application data directories below `/srv/containers/<container>/data`
- Rootless Podman secrets when `value` or `value_file` is supplied
- Systemd linger for every managed service user
- Generated user service enablement and runtime state

### Not Managed

- Rootful or system-wide Quadlets below `/etc/containers/systemd/`
- Secrets stored in Quadlet files, EnvironmentFiles, defaults, or templates
- Container image building or image publishing
- Firewall policy
- Reverse proxy or TLS certificate lifecycle
- Application-specific database provisioning
- `login.defs`, `/etc/subuid`, or `/etc/subgid` range management
- Retrofitting subordinate UID/GID mappings for pre-existing service users
- Purging unmanaged Quadlet files or Podman secrets

## Requirements

- Target hosts need Podman with Quadlet support and systemd user services.
- Target hosts need a shadow-utils `useradd` implementation supporting `--add-subids-for-system`.
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
| `quadlet_users` | `list` | `false` | [] | Service users and rootless Podman Quadlets managed by this role. |

## Managed Files

- `/etc/containers/systemd/users/<UID>/<container>.container` root-owned rootless user Quadlet
- `/etc/containers/systemd/users/<UID>/<pod>.pod` root-owned rootless pod Quadlet
- `/srv/containers/<container>/env/<container>.env` non-secret EnvironmentFile
- `/srv/containers/<container>/data` default application data directory

## Check Mode

File, template, group, package, and systemd tasks follow the check-mode
behavior of their underlying Ansible modules. Service user creation and
rootless Podman secret creation use commands because Ansible modules do
not cover the required rootless Podman and subordinate-ID behavior.

## Service Behavior

Generated container services are user services named `<name>.service` from
`<name>.container`. Generated pod services default to `<name>-pod.service`
from `<name>.pod`, unless the pod sets `service_name`. When a Quadlet changes
and service management is active, the role reloads the service user's systemd
manager through the `ansible.builtin.systemd_service` module with `scope: user`.
Container and pod items support `restart_policy`, `restart_sec`,
`timeout_start_sec`, and repeated `exec_start_pre` commands in the generated
`[Service]` section.

## Security Notes

- Rootless Podman reduces runtime privileges by running application containers in a non-root user namespace owned by the dedicated service user.
- Service users are created with `useradd --system --add-subids-for-system`, so subordinate UID/GID mappings are allocated by the target system's shadow-utils defaults.
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
- Declare shared pod-level port publishing, networks, DNS settings, and volumes under `pods`.
- Set a container `pod` value to a pod unit base name such as `app` or to the explicit Quadlet unit name `app.pod`; both render `Pod=app.pod`.
- Containers with `pod` set render `StartWithPod=true` by default, so starting the pod service starts the associated containers.
- When a pod owns the lifecycle, set pod `enabled` and `state` on the pod and use container `enabled: false` with `state: created` for file-only container units.
- Use `host.containers.internal` from a container to reach services on the container host.
- Use `127.0.0.1` for a database only when the database runs in the same container or in another container joined to the same pod network namespace.
- Omit `uid` unless a fixed service-user UID is required; `useradd --system` otherwise allocates a system UID from the target host defaults.
- Use `exec_start_pre` for local dependency checks such as `pg_isready` before Podman starts the container service.
- `restart_policy` renders `Restart=`, `restart_sec` renders `RestartSec=`, and `timeout_start_sec` renders `TimeoutStartSec=`.
- `type: mount` renders `Secret=<name>` or `Secret=<name>,target=<target>` and lets Podman mount the secret as a file.
- `type: env` renders `Secret=<name>,type=env,target=<ENV_NAME>` and exposes the secret through the container environment.
- EnvironmentFile entries render ordinary `KEY="value"` settings and must not contain passwords, tokens, API keys, or private material.
- Use `value: "{{ vault_secret_name }}"` with encrypted inventory or Ansible Vault when the role should create a Podman secret.
- `value_file` is a remote Managed Host path and should only be used when another trusted process provisions that root-readable file before this role runs.
- If neither `value` nor `value_file` is set for a secret, the role only references the secret in the Quadlet and assumes it already exists in the service user's rootless Podman context.
- The role does not parse or modify `/etc/login.defs`; subordinate ID count and ranges come from the target system's shadow-utils defaults.
- `useradd` allocates subordinate IDs only for newly created users. Existing service users must already have suitable rootless Podman mappings.
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

### Vikunja with host-local PostgreSQL

PostgreSQL runs on the container host. The container reaches it through
Podman's `host.containers.internal` host gateway name. The EnvironmentFile
contains only `_FILE` references, and the Quadlet contains `Secret=`
references.

```yaml
---
- name: Deploy Vikunja with host-local PostgreSQL
  hosts: quadlet
  gather_facts: true
  roles:
    - role: jomrr.quadlet
      vars:
        quadlet_users:
          - name: vikunja
            containers:
              - name: vikunja
                image: docker.io/vikunja/vikunja:latest
                container_name: vikunja
                environment:
                  VIKUNJA_DATABASE_TYPE: postgres
                  VIKUNJA_DATABASE_HOST: host.containers.internal
                  VIKUNJA_DATABASE_USER: vikunja
                  VIKUNJA_DATABASE_DATABASE: vikunja
                  VIKUNJA_DATABASE_PASSWORD_FILE: /run/secrets/vikunja_db_password
                  VIKUNJA_SERVICE_SECRET_FILE: /run/secrets/vikunja_service_secret
                secrets:
                  - name: vikunja_db_password
                    value: "{{ vault_vikunja_db_password }}"
                  - name: vikunja_service_secret
                    value: "{{ vault_vikunja_service_secret }}"
                volumes:
                  - /srv/containers/vikunja/data:/app/vikunja/files:Z
                ports:
                  - 127.0.0.1:3456:3456
                exec_start_pre:
                  - /bin/bash -c 'until pg_isready -h POSTGRES_HOST_IP -p 5432 -U vikunja; do sleep 10; done;'
                restart_policy: on-failure
                restart_sec: 10s
                timeout_start_sec: 300
```
### Vikunja pod with PostgreSQL container

PostgreSQL runs in a second container in the same Podman pod. Containers
in a pod share one network namespace, so the application reaches the
database through `127.0.0.1`.

```yaml
---
- name: Deploy Vikunja and PostgreSQL in one rootless pod
  hosts: quadlet
  gather_facts: true
  roles:
    - role: jomrr.quadlet
      vars:
        quadlet_users:
          - name: vikunja
            pods:
              - name: vikunja
                ports:
                  - 127.0.0.1:3456:3456
                networks:
                  - pasta
            containers:
              - name: vikunja-db
                image: docker.io/library/postgres:16-alpine
                container_name: vikunja-db
                pod: vikunja
                environment:
                  POSTGRES_DB: vikunja
                  POSTGRES_USER: vikunja
                  POSTGRES_PASSWORD_FILE: /run/secrets/vikunja_db_password
                secrets:
                  - name: vikunja_db_password
                    value: "{{ vault_vikunja_db_password }}"
                volumes:
                  - /srv/containers/vikunja-db/data:/var/lib/postgresql/data:Z,U
                enabled: false
                state: created
              - name: vikunja
                image: docker.io/vikunja/vikunja:latest
                container_name: vikunja
                pod: vikunja
                environment:
                  VIKUNJA_DATABASE_TYPE: postgres
                  VIKUNJA_DATABASE_HOST: 127.0.0.1
                  VIKUNJA_DATABASE_USER: vikunja
                  VIKUNJA_DATABASE_DATABASE: vikunja
                  VIKUNJA_DATABASE_PASSWORD_FILE: /run/secrets/vikunja_db_password
                  VIKUNJA_SERVICE_SECRET_FILE: /run/secrets/vikunja_service_secret
                secrets:
                  - name: vikunja_db_password
                    value: "{{ vault_vikunja_db_password }}"
                  - name: vikunja_service_secret
                    value: "{{ vault_vikunja_service_secret }}"
                volumes:
                  - /srv/containers/vikunja/data:/app/vikunja/files:Z
                enabled: false
                state: created
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
