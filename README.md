# Prometheus Monitoring Template

A small, runnable Prometheus monitoring stack for a Linux host and its containers.
It includes Prometheus, Alertmanager, Node Exporter, cAdvisor, Blackbox Exporter, and Grafana.

## What is included

- Host metrics: CPU, memory, disk, load, and network.
- Container metrics: container availability, restarts, CPU, and memory.
- HTTP probes: availability, latency, and TLS certificate expiry.
- Prometheus alert rules with severity labels.
- A provisioned Grafana dashboard.
- Pinned image versions in `.env.example`.

## Architecture

```text
Node Exporter ─┐
cAdvisor ──────┼──> Prometheus ──> Alertmanager
Blackbox ──────┘         │
                         └──> Grafana
```

Prometheus scrapes Node Exporter, cAdvisor, itself, and Blackbox Exporter.
Blackbox Exporter probes the HTTP targets listed in `prometheus/targets/http-targets.yml`.

## Requirements

- Linux host with Docker and Docker Compose.
- Ports 3000, 8080, 9090, 9093, 9100, and 9115 available.
- cAdvisor runs privileged and mounts host paths to read container metrics.

## Quick start

```bash
cp .env.example .env
# Change GRAFANA_ADMIN_PASSWORD in .env
docker compose up -d
```

Check the containers:

```bash
docker compose ps
```

Open:

- Prometheus: <http://localhost:9090>
- Alertmanager: <http://localhost:9093>
- Grafana: <http://localhost:3000>
- cAdvisor: <http://localhost:8080>
- Node Exporter: <http://localhost:9100/metrics>
- Blackbox Exporter: <http://localhost:9115>

Grafana uses the admin user and password from `.env`. The Prometheus data source and the
`Node Exporter Overview` dashboard are provisioned automatically.

## Add HTTP targets

Edit `prometheus/targets/http-targets.yml`:

```yaml
- targets:
    - https://your-service.example.com/health
  labels:
    env: production
    service: your-service
```

Prometheus reloads file-based targets automatically. The configured Blackbox module is `http_2xx`.

## Add an alert receiver

The default Alertmanager receiver is a no-op so the stack starts without external credentials.
Configure a real receiver in `alertmanager/alertmanager.yml`, for example:

```yaml
receivers:
  - name: default
    webhook_configs:
      - url: https://your-alert-webhook.example.com
        send_resolved: true
```

Then restart Alertmanager:

```bash
docker compose restart alertmanager
```

## Tune alerts

Rules live under `prometheus/rules/`:

- `host.yml`: node availability, CPU, memory, disk, and load.
- `containers.yml`: container exits, restarts, CPU, and memory.
- `http.yml`: probe failures, slow responses, and certificate expiry.

Adjust thresholds and `for` durations to match your environment. Every alert has `severity: warning`
or `severity: critical` so Alertmanager can route them differently.

## Useful checks

Validate the Prometheus configuration and rules with the bundled `promtool`:

```bash
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
```

Check the loaded targets in Prometheus:

<http://localhost:9090/targets>

Check the current alert state:

<http://localhost:9090/alerts>

## Layout

```text
.
├── alertmanager/alertmanager.yml
├── blackbox/blackbox.yml
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   ├── rules/
│   └── targets/http-targets.yml
└── grafana/
    ├── dashboards/node-exporter-overview.json
    └── provisioning/
```

## Notes before production

- Configure an Alertmanager receiver with authentication and a real destination.
- Review the 15-day Prometheus retention and the local Docker volumes.
- Add TLS and authentication to Prometheus, Alertmanager, and Grafana.
- cAdvisor needs Linux host access and privileged mode; review its security impact.
- This is a starter template, not a complete production monitoring platform.
