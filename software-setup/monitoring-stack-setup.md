# Monitoring Stack Setup (Prometheus + Grafana + DCGM)

Full observability stack for the kc-llm AI workstation using Prometheus, Grafana, Node Exporter, NVIDIA DCGM, cAdvisor, and Loki.

## Architecture

```
Node Exporter  ──→ ┐
NVIDIA DCGM    ──→ ├──→ Prometheus ──→ Grafana (:3001)
cAdvisor       ──→ ┘
Promtail       ──→ Loki ──→ Grafana
```

## What's Monitored

| Category | Metrics |
|----------|---------|
| **CPU** | Utilization, load average, per-core usage, temperature |
| **Memory** | RAM usage, swap usage, available memory |
| **GPU (2x RTX 3090)** | GPU utilization, VRAM usage, temperature, power draw, fan speed |
| **Storage** | Disk space, IOPS, read/write throughput |
| **Network** | Upload/download throughput, connection count, errors |
| **Containers** | Docker container health, CPU/memory per container |
| **Logs** | System logs, Docker container logs (searchable via Loki) |

## Access

| Service | URL | Port |
|---------|-----|------|
| **Grafana** | `http://100.112.124.101:3001` (Tailscale) | 3001 |
| Prometheus | `http://100.112.124.101:9090` | 9090 |
| Node Exporter | `http://100.112.124.101:9100` | 9100 |
| DCGM Exporter | `http://100.112.124.101:9400` | 9400 |
| cAdvisor | `http://100.112.124.101:8080` | 8080 |
| Loki | `http://100.112.124.101:3100` | 3100 |

> LAN access: replace `100.112.124.101` with `192.168.1.244`

## Grafana Login

- **Username:** `admin`
- **Password:** `kcllm2026`

## Pre-loaded Dashboards

1. **Node Exporter Full** — comprehensive Linux host overview (CPU, RAM, disk, network, filesystem)
2. **NVIDIA DCGM Exporter** — GPU utilization, VRAM, temperature, power draw for both 3090s

## Installation Details

### Prerequisites Installed

- Docker Engine v29.6.2
- Docker Compose Plugin v5.3.1
- NVIDIA Container Toolkit v1.19.1

### Stack Location

All configuration lives in `/opt/monitoring/` on kc-llm:

```
/opt/monitoring/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/datasources.yml
│   │   └── dashboards/dashboards.yml
│   └── dashboards/
│       ├── node-exporter-full.json
│       └── nvidia-dcgm.json
├── loki/
│   └── loki-config.yml
└── promtail/
    └── promtail-config.yml
```

### Managing the Stack

```bash
# SSH into kc-llm
ssh kc-llm

# View status
cd /opt/monitoring && sudo docker compose ps

# Restart everything
cd /opt/monitoring && sudo docker compose restart

# View logs
cd /opt/monitoring && sudo docker compose logs -f grafana

# Stop everything
cd /opt/monitoring && sudo docker compose down

# Start everything
cd /opt/monitoring && sudo docker compose up -d
```

### Data Retention

- **Prometheus:** 30 days of metrics
- **Loki:** 30 days of logs
- All containers set to `restart: unless-stopped` (survives reboots)

## Notes

- Port 3000 was already in use (Open WebUI / Next.js), so Grafana runs on **3001**
- Ollama v0.31.2 does not expose native Prometheus metrics — GPU inference load is still captured via DCGM. When Ollama adds `/metrics` support, add this to `prometheus.yml`:
  ```yaml
  - job_name: "ollama"
    metrics_path: /metrics
    static_configs:
      - targets: ["172.18.0.1:11434"]
  ```
- DCGM profiling metrics (SM occupancy, tensor core usage) require `--cap-add SYS_ADMIN` on the container if needed in the future

## Prometheus Scrape Targets

| Target | Status | Endpoint |
|--------|--------|----------|
| prometheus | ✅ UP | `localhost:9090` |
| node-exporter | ✅ UP | `node-exporter:9100` |
| dcgm-exporter | ✅ UP | `dcgm-exporter:9400` |
| cadvisor | ✅ UP | `cadvisor:8080` |
