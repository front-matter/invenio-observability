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

Import steps:

1. In Grafana go to **Dashboards** → **New** → **Import**
2. Upload `dashboards/inveniordm-logs.json`
3. When prompted, select your **Loki** data source

The dashboard provides variables for `app`, `environment`, `component`, `role`, `level`, `service`, `container`, and `stream`.

Notes:

- The `Search` variable is treated as a regex (LogQL `|~`). Default is `.*` (show all logs). For a simple “contains” filter you can usually just type a word like `error`.
- Loki requires at least one non-empty-compatible matcher in the selector; therefore the `app` variable uses `.+` for “All”.

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
