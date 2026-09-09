# Lightweight Observability Lab (Grafana + Loki + Prometheus)

A lightweight monitoring and logging environment built with Docker, Loki, Prometheus, and Grafana. Designed to run smoothly on machines with limited resources (e.g., 8 GB Host RAM) by aggregating metrics and system logs from Linux and Windows endpoints.

---

## Architecture Overview

```text
┌──────────────────────────┐        ┌──────────────────────────┐
│   Endpoints (VM / Node)  │        │   Central Server (Docker)│
│ ──────────────────────── │        │ ──────────────────────── │
│ • Node/Windows Exporter  │──PULL─>│ • Prometheus (:9090)     │
│ • Promtail               │──PUSH─>│ • Loki       (:3100)     │
│                          │        │ • Grafana    (:3000)     │
└──────────────────────────┘        └──────────────────────────┘

```

- **Prometheus**: Scrapes hardware and OS metrics (CPU, RAM, Disk, Network).

- **Loki**: Ingests compressed log streams indexed by label metadata.

- **Grafana**: Unified UI for visualizing metrics and querying logs via LogQL.

- **Agents**: `node_exporter` / `windows_exporter` for metrics and `promtail` for logs.

## Quick Start

### 1. Deploy the Central Docker Stack

Clone this repository and start the containers:

Bash

```bash
docker compose up -d
```

Verify services are up:

- **Grafana**: `http://localhost:3000` (Default: `admin` / `admin`)
- **Prometheus**: `http://localhost:9090`
- **Loki**: `http://localhost:3100/ready`

### 2. Configure Endpoint Agents (Linux Target)

#### Step A: Install Node Exporter (Metrics)

Bash

```bash
sudo apt update && sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
```

#### Step B: Configure Promtail (Logs)

Create `/etc/promtail-config.yaml` on the target machine:

YAML

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/log/positions.yaml

clients:
  - url: http://<CENTRAL_SERVER_IP>:3100/loki/api/v1/push

scrape_configs:
  - job_name: system-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          hostname: linux-node
          __path__: /var/log/{syslog,auth.log,messages,*.log}
```

Run Promtail:

Bash

```bash
promtail -config.file=/etc/promtail-config.yaml
```

### 3. Import Pre-built Dashboards

1. Access Grafana (`http://localhost:3000`).

2. Add Data Sources:
   - **Prometheus**: `http://prometheus:9090`
   - **Loki**: `http://loki:3100`

3. Import community dashboards via **Dashboards > Import**:
   - **Linux Metrics**: Dashboard ID `1860` (_Node Exporter Full_)
   - **Windows Metrics**: Dashboard ID `14499` (_Windows Exporter_)
   - **Linux Logs**: Dashboard ID `13639` (_Loki Syslog_)

## Network & Connectivity Notes

- **Prometheus (Metrics)** uses a **PULL** architecture: Prometheus connects to the endpoint's exporter port (`9100` for Linux, `9182` for Windows).

- **Loki (Logs)** uses a **PUSH** architecture: Promtail running on the endpoint pushes logs to Loki on port `3100`.

- **WSL2 / Windows Host**: If running Docker on WSL2, enable mirrored networking mode in `%USERPROFILE%\.wslconfig` (`networkingMode=mirrored`) or open incoming ports `3100` and `9090` via Windows Firewall to allow external VM connections.
