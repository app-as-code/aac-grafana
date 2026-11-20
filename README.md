<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/icon-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="assets/icon-light.svg">
    <img src="assets/icon.svg" alt="Grafana logo" width="128">
  </picture>
</p>

<h1 align="center">aac-grafana</h1>

<p align="center">Self-hosted deployment definition for Grafana, the open-source platform for monitoring and observability.</p>

<p align="center">
  <a href="https://github.com/grafana/grafana"><img alt="Upstream" src="https://img.shields.io/badge/upstream-grafana%2Fgrafana-blue?logo=github"></a>
  <img alt="Image" src="https://img.shields.io/badge/image-grafana%2Fgrafana--enterprise%3A13.2.3--boringcrypto-F46800?logo=docker&logoColor=white">
  <a href="https://github.com/grafana/grafana/blob/main/LICENSE"><img alt="License" src="https://img.shields.io/badge/license-AGPL--3.0-green"></a>
</p>

## About

Grafana is a web application for analytics and interactive visualization. It turns time-series
and other data into dashboards, charts and alerts, and connects to a wide range of data sources.
This repository deploys the Grafana Enterprise image, which runs with the open-source feature set
when no enterprise license is installed. Datasources and dashboards are kept as provisioning files
in the repository and rolled out with Grafana's file provisioning mechanism.

Key features (as provided by upstream):

- Dashboards with panels, variables and interactive exploration
- Plugin-based data source support (time series, logs, traces and more)
- Alerting with rules and notification channels
- User management with OAuth/OpenID Connect login (this deployment is prepared for an
  external identity provider)
- Provisioning of data sources, dashboards and alerting from files

## Upstream

- Project: <https://github.com/grafana/grafana>
- Documentation: <https://grafana.com/docs/grafana/latest/>
- Container image: `grafana/grafana-enterprise:13.2.3-boringcrypto` (tag from `service.yml`)
- License: AGPL-3.0 for the open-source project; the Enterprise image is subject to the Grafana
  Enterprise license terms

## Quick start (Docker Compose)

The Compose file is rendered from `service.yml` by the App-as-Code template engine. For a
manual run, the external networks used for routing must exist first:

```bash
docker network create services-secured
docker network create services-exposed
docker network create docker-default   # only if it does not exist yet

# Provide the secrets listed under "Configuration" in deployments/docker_compose/.env
# (or the equivalent stack.env) - they are not stored in this repository.
docker compose -f deployments/docker_compose/docker-compose.yml up -d
```

Secrets are not stored in this repository. Provide them through the deployment pipeline or
create the environment files yourself. Do not commit them.

## Configuration

Environment variables (defaults from `service.yml`; `CHANGE_ME` must be set per deployment,
secrets are marked as such):

| Variable | Default | Description |
|---|---|---|
| `GF_SECURITY_ADMIN_USER` | `admin` | Initial Grafana admin user |
| `GF_SECURITY_ADMIN_PASSWORD` | `CHANGE_ME` (secret) | Initial Grafana admin password |
| `GF_SERVER_ROOT_URL` | `CHANGE_ME` | Public root URL of the Grafana instance |
| `GF_USERS_ALLOW_SIGN_UP` | `true` | Allow users to sign up themselves |
| `GF_AUTH_ANONYMOUS_ENABLED` | `true` | Enable anonymous access (fixed in the compose file) |
| `GF_AUTH_ANONYMOUS_ORG_ROLE` | `Viewer` | Role for anonymous users (fixed in the compose file) |
| `GF_AUTH_ANONYMOUS_ORG_NAME` | `Main Org.` | Organization for anonymous users (fixed in the compose file) |
| `GF_AUTH_GENERIC_OAUTH_ENABLED` | `true` | Enable generic OAuth login |
| `GF_AUTH_GENERIC_OAUTH_NAME` | `authentik` | Label of the OAuth login button |
| `GF_AUTH_GENERIC_OAUTH_CLIENT_ID` | `CHANGE_ME` (secret) | OAuth client ID |
| `GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET` | `CHANGE_ME` (secret) | OAuth client secret |
| `GF_AUTH_GENERIC_OAUTH_AUTH_URL` | `CHANGE_ME` | OAuth authorization endpoint |
| `GF_AUTH_GENERIC_OAUTH_TOKEN_URL` | `CHANGE_ME` (secret) | OAuth token endpoint |
| `GF_AUTH_GENERIC_OAUTH_API_URL` | `CHANGE_ME` | OAuth user-info endpoint |
| `GF_AUTH_GENERIC_OAUTH_SCOPES` | `openid profile email` | Requested OAuth scopes |
| `GF_AUTH_GENERIC_OAUTH_ROLE_ATTRIBUTE_PATH` | group-based JMESPath expression | Maps OAuth groups to Grafana roles (`GrafanaAdmin`, `Admin`, `Viewer`) |
| `GF_AUTH_GENERIC_OAUTH_ALLOW_ASSIGN_GRAFANA_ADMIN` | `true` | Allow the OAuth role mapping to grant Grafana server admin |
| `GF_AUTH_OAUTH_AUTO_LOGIN` | `false` | Redirect straight to the OAuth login |
| `GF_AUTH_SIGNOUT_REDIRECT_URL` | `CHANGE_ME` | Redirect target after sign-out |
| `GF_DS_INFLUX_ONPREM_TOKEN` | `CHANGE_ME` (secret) | Read-only token for the on-premises InfluxDB data source (`$VAR` expansion in provisioning) |
| `GF_DS_INFLUX_HETZNER_TOKEN` | `CHANGE_ME` (secret) | Read-only token for the second InfluxDB data source (`$VAR` expansion in provisioning) |
| `TZ` | `Europe/Berlin` | Time zone |
| `PUID` / `PGID` | `1000` | User and group ID the container runs as |

Ports:

| Port | Protocol | Purpose |
|---|---|---|
| `3000` | TCP | Web interface (`web`), routed through Traefik |

Volumes:

| Mount | Description |
|---|---|
| `data` -> `/var/lib/grafana` | Grafana data, dashboards and plugins |
| `provisioning` -> `/etc/grafana/provisioning` | Provisioning files for data sources, dashboards and alerting |

Networks (Compose): `services-secured` and `services-exposed` (external, used for Traefik
routing), `docker-default` (external, interconnect) and the internal bridge `stack_internal`.

## Provisioning layout

```
custom_templates/files/provisioning/
├── datasources/datasources.yaml.j2     data sources (fixed UIDs, tokens via environment variables)
├── dashboards/dashboards.yaml.j2       dashboard provider (folders map to Grafana folders)
└── dashboards/json/<folder>/<name>.json   dashboard exports, shipped verbatim
scripts/export-dashboards.sh            export dashboards from the UI back into the repository
scripts/validate.sh                     syntax checks (local and CI)
```

Dashboard JSON files are not Jinja templates; they are delivered verbatim by the template engine
so that Grafana expressions are not interpreted.

## Deploy with App-as-Code

1. Edit `service.yml`. It is the single source of truth for image, ports, environment,
   volumes, dependencies and per-stage overrides. Do not edit the rendered output.
2. Pushing to `main` (or a configured stage branch) runs the service pipeline, which renders
   the manifests with `aac-template-engine` and triggers the deployment.
3. Releases of this definition are published under <https://github.com/app-as-code/aac-grafana>.

## Credits and license

Grafana is developed by Grafana Labs and the Grafana community. The open-source project is
licensed under AGPL-3.0. This repository only contains the deployment definition for Grafana
and is not a fork of the application. The logo is provided by the Grafana project; see
[`assets/ICON_SOURCE.md`](assets/ICON_SOURCE.md) for icon sources and licenses.
