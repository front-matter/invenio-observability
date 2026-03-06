# invenio-observability

Self-hosted observability stack for InvenioRDM. Key components include:

- Vector for log collection and forwarding
- Loki for log aggregation
- Grafana for visualization and notification

## Linux host: volume permissions

This stack uses bind mounts under `/data/coolify/services/...`. Make sure the host directories exist and are writable by the user IDs used inside the containers.

Container default users:

- Loki (`grafana/loki:2.9.6`): UID/GID `10001:10001`
- Grafana (`grafana/grafana:11.0.0`): UID/GID `472:0`
- Vector (`timberio/vector:0.38.0-alpine`): runs as `root` by default (needed for `/var/run/docker.sock` in this setup)

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
