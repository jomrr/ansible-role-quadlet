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
- Non-secret EnvironmentFiles at the explicitly configured `environment_file` paths
- Default application data directories below `quadlet_base_dir/<container>/data`
- Rootless Podman secrets from a supplied `value` or `value_file`
- Systemd linger for every managed service user
- Generated user service installation and runtime state
- Rootless Podman auto-update timers when containers request auto-update

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
  - name: ansible.posix
  - name: community.general
  - name: containers.podman
```

## Role Variables

The following variables are part of the public role interface.

| Name | Type | Required | Default | Description |
| ---- | ---- | -------- | ------- | ----------- |
| `quadlet_base_dir` | `path` | `false` | `/srv/containers` | Base directory for default container data paths. |
| `quadlet_users` | `list` | `false` | [] | Service users and rootless Podman Quadlets managed by this role. |

## Managed Files

- `/etc/containers/systemd/users/<UID>/<container>.container` root-owned rootless user Quadlet
- `/etc/containers/systemd/users/<UID>/<pod>.pod` root-owned rootless pod Quadlet
- `<environment_file>` explicitly configured non-secret EnvironmentFile
- `/srv/containers/<container>/data` default application data directory

## Check Mode

File, template, group, package, and systemd tasks follow the check-mode
behavior of their underlying Ansible modules. Initial service-user
creation uses `useradd` because `ansible.builtin.user` cannot request
subordinate IDs for system accounts.

## Service Behavior

Generated container services are user services named `<name>.service` from
`<name>.container`. Generated pod services default to `<name>-pod.service`
from `<name>.pod`, unless the pod sets `service_name`. The role renders every
pod and container for a service user before reloading the user manager once.
Changed Quadlets, EnvironmentFiles, and managed secrets restart services
whose requested state is `started` for the affected service user.

The `enabled` value controls `WantedBy=` in the generated `[Install]`
section. The role does not call `systemctl enable` for generated units.
Enabled items use `default.target` unless `install_options.WantedBy` supplies
another target. Disabled items cannot set `install_options.WantedBy`.

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
- Podman secrets are updated only when their supplied `value` or `value_file` content differs.
- Docker Compose-specific escaping rules for `$` are not applied to EnvironmentFiles or Quadlet templates.

## Operational Notes

- Use file-based application settings such as `*_FILE=/run/secrets/<secret>` whenever the application supports them.
- Declare shared pod-level port publishing, networks, and volumes under `pods`.
- Set a container `pod` value to a pod unit base name such as `app` or to the explicit Quadlet unit name `app.pod`; both render `Pod=app.pod`.
- Containers with `pod` set render `StartWithPod=true` by default, so starting the pod service starts the associated containers.
- When a pod owns the lifecycle, set pod `enabled` and `state` on the pod and use container `enabled: false` with `state: created` for file-only container units.
- Use `host.containers.internal` from a container to reach services on the container host.
- Use `127.0.0.1` for a database only when the database runs in the same container or in another container joined to the same pod network namespace.
- Omit `uid` unless a fixed service-user UID is required; `useradd --system` otherwise allocates a system UID from the target host defaults.
- Use `service_options.ExecStartPre` for dependency checks such as `pg_isready` before Podman starts the container service.
- `restart_policy` renders `Restart=`; additional service settings such as `RestartSec` belong in `service_options`.
- `ports`, `volumes`, `networks`, and `tmpfs` contain native Quadlet values rendered without Compose-style conversion.
- `unit_options`, `service_options`, and `install_options` extend their named systemd sections. `container_options` and `pod_options` extend the matching Quadlet section. A list value repeats the key once per item.
- Options rendered directly by the role cannot be overridden through an options map. Multi-value keys such as `Secret`, `Volume`, `PublishPort`, `Network`, and `DropCapability` may be added through a map.
- `exec` renders `Exec=` and supplies arguments after the image entrypoint.
- `auto_update` renders `AutoUpdate=` and causes the role to enable and start `podman-auto-update.timer` for the service user. Registry updates require a fully-qualified image reference.
- `type: mount` renders `Secret=<name>` or `Secret=<name>,target=<target>` and lets Podman mount the secret as a file.
- `type: env` renders `Secret=<name>,type=env,target=<ENV_NAME>` and exposes the secret through the container environment.
- EnvironmentFile entries render ordinary `KEY="value"` settings and must not contain passwords, tokens, API keys, or private material.
- Every container must set an explicit `environment_file` path.
- Use `value: "{{ vault_secret_name }}"` with encrypted inventory or Ansible Vault when the role should create a Podman secret.
- `value_file` is a remote Managed Host path and should only be used when another trusted process provisions that root-readable file before this role runs.
- Every secret requires exactly one source: `value` or `value_file`.
- The role does not parse or modify `/etc/login.defs`; subordinate ID count and ranges come from the target system's shadow-utils defaults.
- `useradd` allocates subordinate IDs only for newly created users. Existing service users must already have suitable rootless Podman mappings.
- Set `state: created` for file-only rendering without runtime service management.
- Set `enabled: false` to omit the default `WantedBy=default.target` installation target.

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
                environment_file: /srv/containers/vikunja/env/vikunja.env
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
                restart_policy: on-failure
                service_options:
                  ExecStartPre: >-
                    /bin/bash -c 'until pg_isready -h POSTGRES_PUBLIC_IP -p 5432 -U vikunja; do sleep 10; done;'
                  RestartSec: 10s
                  TimeoutStartSec: "300"
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
                environment_file: /srv/containers/vikunja-db/env/vikunja-db.env
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
                environment_file: /srv/containers/vikunja/env/vikunja.env
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
### Vaultwarden with explicit environment secret fallback

Vaultwarden consumes `ADMIN_TOKEN` as an environment variable when no
suitable file-based setting is available. The token remains a Podman
secret and is not written to the EnvironmentFile or Quadlet.

```yaml
---
- name: Deploy Vaultwarden
  hosts: quadlet
  gather_facts: true
  roles:
    - role: jomrr.quadlet
      vars:
        quadlet_users:
          - name: vaultwarden
            containers:
              - name: vaultwarden
                image: docker.io/vaultwarden/server:latest
                container_name: vaultwarden
                environment_file: /srv/containers/vaultwarden/env/vaultwarden.env
                environment:
                  DOMAIN: https://vaultwarden.example.invalid
                  SIGNUPS_ALLOWED: "false"
                secrets:
                  - name: vaultwarden_admin_token
                    type: env
                    env_target: ADMIN_TOKEN
                    value: "{{ vault_vaultwarden_admin_token }}"
                volumes:
                  - /srv/containers/vaultwarden/data:/data:Z
                ports:
                  - 127.0.0.1:8080:80
```
### ntfy with entrypoint arguments and auto-update

The container receives `serve` after its image entrypoint. Registry
auto-update enables the service user's Podman auto-update timer, while
the application-specific health command uses `container_options`.

```yaml
---
- name: Deploy ntfy
  hosts: quadlet
  gather_facts: true
  roles:
    - role: jomrr.quadlet
      vars:
        quadlet_users:
          - name: ntfy
            containers:
              - name: ntfy
                image: docker.io/binwiederhier/ntfy:latest
                container_name: ntfy
                environment_file: /srv/containers/ntfy/env/ntfy.env
                secrets: []
                exec: serve
                auto_update: registry
                environment:
                  NTFY_BASE_URL: https://ntfy.example.invalid
                  NTFY_CACHE_FILE: /var/cache/ntfy/cache.db
                volumes:
                  - /srv/containers/ntfy/data:/var/cache/ntfy:Z
                ports:
                  - 127.0.0.1:8081:80
                container_options:
                  HealthCmd: wget -q --tries=1 http://127.0.0.1:80/v1/health -O -
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
