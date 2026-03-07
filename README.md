# invenio-observability

Self-hosted observability stack for InvenioRDM. Key components include:

- Vector for log collection and forwarding
- Loki for log aggregation
- Grafana for visualization and notification

## Linux host: volume permissions

This stack uses bind mounts under `/data/coolify/services/...`. Make sure the host directories exist and are writable by the user IDs used inside the containers.

Container default users:

- Loki (`grafana/loki:2.9.17`): UID/GID `10001:10001`
- Grafana (`grafana/grafana:12.4.0`): UID/GID `472:0`
- Vector (`timberio/vector:0.53.0-alpine`): runs as `root` by default (needed for `/var/run/docker.sock` in this setup)

Create directories and apply ownership/permissions:

```bash
sudo mkdir -p \
	/data/coolify/services/loki \
	/data/coolify/services/loki/loki-data \
	/data/coolify/services/grafana/grafana-data \
	/data/coolify/services/vector

# Loki data directory (read/write)
sudo chown -R 10001:10001 /data/coolify/services/loki/loki-data
sudo chmod -R u+rwX /data/coolify/services/loki/loki-data

# Loki config (read-only inside the container)
sudo chmod 644 /data/coolify/services/loki/loki-config.yaml

# Grafana data directory (read/write)
sudo chown -R 472:0 /data/coolify/services/grafana/grafana-data
sudo chmod -R u+rwX /data/coolify/services/grafana/grafana-data

# Vector config (read-only inside the container)
sudo chmod 644 /data/coolify/services/vector/vector.toml
```

## Grafana dashboard (logs)

This repository includes an importable dashboard JSON for exploring logs in Loki using the labels produced by `vector.toml`.

File:

- `dashboards/inveniordm-logs.json`
- `dashboards/inveniordm-requests.json`

Import steps:

1. In Grafana go to **Dashboards** → **New** → **Import**
2. Upload `dashboards/inveniordm-logs.json` (Services) or `dashboards/inveniordm-requests.json` (Requests)
3. When prompted, select your **Loki** data source

The dashboards provide variables for `app`, `environment`, `component`, `level`, `container`, and `stream`.

Note: the dashboard variable is named `level`, but the underlying Loki stream label is `log_level` (Vector computes `.severity` and maps it to the `log_level` label when sending to Loki).

Notes:

- The `Search` variable is treated as a regex (LogQL `|~`). Default is empty (no extra filter). For a simple “contains” filter you can usually just type a word like `error`.
- Loki requires at least one non-empty-compatible matcher in the selector; therefore the `app` variable uses `.+` for “All”.

## Log labels and levels

- `vector.toml` extracts `service`, `app` and `environment` from Docker labels. In some environments (e.g. Coolify), labels are exposed as `.label` instead of `.container_labels`.
- `component` is derived strictly from `label_coolify_resourceName`.
- If the container name starts with `web-` or `worker-`, the component is prefixed as `web-<resource>` / `worker-<resource>`.
- For HTTP access logs without an explicit log level, Vector infers `.severity` from the HTTP status code (2xx/3xx → `info`, 4xx → `warn`, 5xx → `error`). This is sent to Loki as the `log_level` label.
- For common HTTP access log lines, Vector also extracts an optional `.user_agent` field (kept as a log field, not a Loki label). In LogQL you can filter it like: `| json | user_agent=~"SentryUptimeBot.*"`.
- Vector extracts the HTTP method into `.http_method` (e.g. `get`, `post`). Filter example: `| json | http_method="post"`.

## Grafana provisioning (recommended)

To avoid Grafana pointing at the wrong Loki URL (a common reason for “no results” in Explore), provision the Loki datasource.

1. Create on the host:

```bash
sudo mkdir -p /data/coolify/services/grafana/provisioning/datasources
```

2. Copy the repo file:

- [grafana/provisioning/datasources/loki.yaml](grafana/provisioning/datasources/loki.yaml)

to:

- `/data/coolify/services/grafana/provisioning/datasources/loki.yaml`

3. Ensure your Grafana service has this volume mount (see [compose.yaml](compose.yaml)):

- `/data/coolify/services/grafana/provisioning:/etc/grafana/provisioning:ro`

4. Restart Grafana.

## Alerting: errors in logs

This repository includes a provisioned Grafana alert rule that triggers when Loki receives error-like log lines (regex matches `error|exception|traceback|fatal|panic`) over the last 5 minutes.

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
