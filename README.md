# invenio-observability

Self-hosted observability stack for InvenioRDM. Key components include:

- Vector for log collection and forwarding
- VictoriaLogs for log storage and querying
- Grafana for visualization and notification

## Linux host: volume permissions

This stack uses bind mounts under `/data/coolify/services/...`. Make sure the host directories exist and are writable by the user IDs used inside the containers.

Container default users:

- VictoriaLogs (`victoriametrics/victoria-logs:v1.47.0`): (container default user)
- Grafana (`grafana/grafana:12.4.0`): UID/GID `472:0`
- Vector (`timberio/vector:0.53.0-alpine`): runs as `root` by default (needed for `/var/run/docker.sock` in this setup)

Create directories and apply ownership/permissions:

```bash
sudo mkdir -p \
	/data/coolify/services/victorialogs/vl-data \
	/data/coolify/services/grafana/grafana-data \
	/data/coolify/services/vector

# VictoriaLogs data directory (read/write)
sudo chmod -R u+rwX /data/coolify/services/victorialogs/vl-data

# Grafana data directory (read/write)
sudo chown -R 472:0 /data/coolify/services/grafana/grafana-data
sudo chmod -R u+rwX /data/coolify/services/grafana/grafana-data

# Vector config (read-only inside the container)
sudo chmod 644 /data/coolify/services/vector/vector.toml
```

## Grafana dashboard (logs)

This repository includes importable dashboard JSONs for exploring logs in VictoriaLogs using the labels produced by `vector.toml`.

File:

- `dashboards/inveniordm-logs.json`
- `dashboards/inveniordm-requests.json`

Import steps:

1. In Grafana go to **Dashboards** → **New** → **Import**
2. Upload `dashboards/inveniordm-logs.json` (Services) or `dashboards/inveniordm-requests.json` (Requests)
3. When prompted, select your **VictoriaLogs** data source

The dashboards provide variables for `app`, `environment`, `component`, `level`, `container`, and `stream`.

Note: the dashboard variable is named `level`, but the underlying log label is `log_level` (Vector computes `.severity` and maps it to the `log_level` label when sending).

Notes:

- The `Search` variable is treated as a regex. Default is `.*` (match all). For a simple “contains” filter you can usually just type a word like `error`.
- The dashboards use the VictoriaLogs data source and LogsQL queries (not LogQL).

## Log labels and levels

- `vector.toml` extracts `service`, `app` and `environment` from Docker labels. In some environments (e.g. Coolify), labels are exposed as `.label` instead of `.container_labels`.
- `component` is derived strictly from `label_coolify_resourceName`.
- If the container name starts with `web-` or `worker-`, the component is prefixed as `web-<resource>` / `worker-<resource>`.
- For HTTP access logs without an explicit log level, Vector infers `.severity` from the HTTP status code (2xx/3xx → `info`, 4xx → `warn`, 5xx → `error`). This is sent as the `log_level` label.
- For common HTTP access log lines, Vector also extracts an optional `.user_agent` field (kept as a log field, not a label) to avoid label-cardinality explosion.
- Vector extracts the HTTP method into `.http_method` and also sends it as a low-cardinality label `http_method` so dashboards can filter without JSON parsing.

## Grafana provisioning (recommended)

To avoid Grafana pointing at the wrong VictoriaLogs URL (a common reason for “no results” in Explore), provision the VictoriaLogs datasource.

1. Create on the host:

```bash
sudo mkdir -p /data/coolify/services/grafana/provisioning/datasources
```

2. Copy the repo file:

- [grafana/provisioning/datasources/victorialogs.yaml](grafana/provisioning/datasources/victorialogs.yaml)

to:

- `/data/coolify/services/grafana/provisioning/datasources/victorialogs.yaml`

3. Ensure your Grafana service has this volume mount (see [compose.yaml](compose.yaml)):

- `/data/coolify/services/grafana/provisioning:/etc/grafana/provisioning:ro`

4. Restart Grafana.

## Protect VictoriaLogs Web UI (Basic Auth)

VictoriaLogs supports built-in HTTP Basic Auth via `-httpAuth.username` and `-httpAuth.password` flags.

This repo's [compose.yaml](compose.yaml), [vector.toml](vector.toml) and the provisioned Grafana datasource are set up so you can enable Basic Auth just by defining these environment variables in your deployment:

- `VICTORIALOGS_HTTPAUTH_USERNAME`
- `VICTORIALOGS_HTTPAUTH_PASSWORD`

If `VICTORIALOGS_HTTPAUTH_USERNAME` is empty, authentication is disabled.

Notes:

- Basic Auth will apply to the VictoriaLogs Web UI and its HTTP APIs on `:9428`.
- Vector (Loki sink) and Grafana datasource will use the same credentials.

## Protect VictoriaLogs Web UI (OIDC)

VictoriaLogs doesn't support OIDC natively. If you want OIDC (e.g. via Keycloak), put an auth proxy in front of VictoriaLogs.

This repo includes an `oauth2-proxy` service in [compose.yaml](compose.yaml) that can front VictoriaLogs using Keycloak OIDC.

Important notes:

- Expose **only** `oauth2-proxy` publicly (via `SERVICE_FQDN_OAUTH2PROXY_4180`). Do not expose VictoriaLogs directly.
- When using `oauth2-proxy`, keep VictoriaLogs Basic Auth disabled (leave `VICTORIALOGS_HTTPAUTH_USERNAME` empty), unless you also configure `oauth2-proxy` to authenticate to the upstream.

You must set these environment variables for `oauth2-proxy` (names in compose are examples; adapt to your setup):

- `OAUTH2_PROXY_OIDC_ISSUER_URL` (Keycloak realm issuer URL)
- `OAUTH2_PROXY_CLIENT_ID`
- `OAUTH2_PROXY_CLIENT_SECRET`
- `OAUTH2_PROXY_REDIRECT_URL` (must match the client config in Keycloak)
- `OAUTH2_PROXY_COOKIE_SECRET` (required; generate a strong secret)
- `OAUTH2_PROXY_COOKIE_DOMAINS`
- `OAUTH2_PROXY_EMAIL_DOMAINS`


## Alerting: errors in logs

This repository includes a provisioned Grafana alert rule that triggers when VictoriaLogs receives error-like log lines (regex matches `error|exception|traceback|fatal|panic`) over the last 5 minutes.

File:

- [grafana/provisioning/alerting/inveniordm-errors.yaml](grafana/provisioning/alerting/inveniordm-errors.yaml)

Enable it:

1. Create on the host:

```bash
sudo mkdir -p /data/coolify/services/grafana/provisioning/alerting
```

2. Copy the repo file to:

- `/data/coolify/services/grafana/provisioning/alerting/inveniordm-errors.yaml`

3. Restart Grafana.

Notes:

- You still need to configure a **Contact point** and **Notification policy** in Grafana for alerts to actually notify (SMTP env vars only enable sending; they don’t define recipients).
- The provisioned rule will show up under **Alerting** → **Alert rules**, in the **General** folder.
