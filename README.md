# VPS Monitoring Stack

A production-ready monitoring stack for VPS deployments running Docker containers. This setup provides comprehensive observability through metrics collection, alerting, and visualization.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              VPS HOST                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │   App 1     │    │   App 2     │    │   App 3     │    │   App N     │  │
│  │  (Next.js)  │    │  (Next.js)  │    │  (Next.js)  │    │    ...      │  │
│  │  :3000      │    │  :3000      │    │  :3000      │    │             │  │
│  │ prom-client │    │ prom-client │    │ prom-client │    │             │  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └─────────────┘  │
│         │                  │                  │                             │
│         └──────────────────┼──────────────────┘                             │
│                            │ /api/metrics                                   │
│                            ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     MONITORING STACK                                │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐              │   │
│  │  │ Prometheus  │◄───│  cAdvisor   │    │    Node     │              │   │
│  │  │   :9090     │    │   :8080     │    │  Exporter   │              │   │
│  │  │             │    │  Container  │    │   :9100     │              │   │
│  │  │  - Scrape   │    │   Metrics   │    │   Host      │              │   │
│  │  │  - Store    │    │             │    │   Metrics   │              │   │
│  │  │  - Alert    │    └─────────────┘    └──────┬──────┘              │   │
│  │  └──────┬──────┘                              │                     │   │
│  │         │                                     │                     │   │
│  │         ▼                                     │                     │   │
│  │  ┌─────────────┐    ┌─────────────┐          │                     │   │
│  │  │Alertmanager │    │   Grafana   │◄─────────┘                     │   │
│  │  │   :9093     │    │   :3001     │                                │   │
│  │  │             │    │             │                                │   │
│  │  │  - Route    │    │  - Visualize│                                │   │
│  │  │  - Notify   │    │  - Dashboard│                                │   │
│  │  └──────┬──────┘    └─────────────┘                                │   │
│  │         │                                                           │   │
│  └─────────┼───────────────────────────────────────────────────────────┘   │
│            │                                                                │
│            ▼                                                                │
│       ┌─────────┐                                                           │
│       │  Email  │                                                           │
│       │  Alert  │                                                           │
│       └─────────┘                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Components

| Component | Purpose | Port |
|-----------|---------|------|
| **Prometheus** | Time-series database, scrapes and stores metrics | 9090 |
| **Alertmanager** | Handles alerts, routes to email/Slack | 9093 |
| **Grafana** | Visualization and dashboards | 3001 |
| **cAdvisor** | Container metrics (CPU, memory, network, disk) | 8080 |
| **Node Exporter** | Host-level metrics (CPU, memory, disk, network) | 9100 |

## Metrics Layers

### 1. Host Metrics (Node Exporter)
- CPU utilization
- Memory usage
- Disk space and I/O
- Network traffic
- System load

### 2. Container Metrics (cAdvisor)
- Per-container CPU usage
- Per-container memory consumption
- Container network I/O
- Container filesystem usage
- Container restart counts

### 3. Application Metrics (prom-client)
- HTTP request rate, latency, errors (RED method)
- Custom business metrics (quiz completions, payments, etc.)
- Database query performance
- Event loop lag (Node.js)

## Quick Start

### Prerequisites
- Docker and Docker Compose installed
- A VPS with at least 2GB RAM
- Docker network for your applications already created

### 1. Clone and Configure

```bash
git clone https://github.com/YOUR_USERNAME/vps-monitoring-guide.git
cd vps-monitoring-guide

# Copy example configs
cp alertmanager/alertmanager.example.yml alertmanager/alertmanager.yml

# Edit alertmanager.yml with your email settings
nano alertmanager/alertmanager.yml
```

### 2. Create External Network (if not exists)

```bash
docker network create monitoring
```

### 3. Start the Stack

```bash
docker compose up -d
```

### 4. Verify Everything is Running

```bash
# Check container status
docker compose ps

# Check Prometheus targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

### 5. Access Dashboards

- **Grafana**: http://your-vps-ip:3001 (admin / changeme)
- **Prometheus**: http://your-vps-ip:9090
- **Alertmanager**: http://your-vps-ip:9093

## Connecting Your Applications

### Option 1: Add Application Network to Prometheus

Edit `docker-compose.yml` and add your app's network:

```yaml
services:
  prometheus:
    networks:
      - monitoring
      - your-app-network  # Add this

networks:
  your-app-network:
    external: true
```

### Option 2: Expose Metrics Port (Not Recommended for Production)

Expose your app's metrics port to the host and configure Prometheus to scrape `host.docker.internal:PORT`.

### Adding Scrape Targets

Edit `prometheus/prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'your-app'
    metrics_path: '/api/metrics'
    static_configs:
      - targets: ['your-app-container:3000']
        labels:
          environment: 'production'
          service: 'your-app'
```

Reload Prometheus:
```bash
curl -X POST http://localhost:9090/-/reload
```

## Alerting

### Configured Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| HostHighCpuUsage | CPU > 80% for 5m | warning |
| HostHighMemoryUsage | Memory > 85% for 5m | warning |
| HostDiskSpaceLow | Disk > 85% for 5m | warning |
| HostDown | Node exporter unreachable for 1m | critical |
| ContainerHighCpu | Container CPU > 80% for 5m | warning |
| ContainerHighMemory | Container memory > 90% for 5m | warning |
| ContainerRestarting | > 2 restarts in 15m | warning |
| ApplicationDown | App endpoint unreachable for 1m | critical |

### Email Configuration

Edit `alertmanager/alertmanager.yml`:

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@yourdomain.com'
  smtp_auth_username: 'your-email@gmail.com'
  smtp_auth_password: 'your-app-password'

receivers:
  - name: 'email'
    email_configs:
      - to: 'your-email@gmail.com'
```

## cAdvisor Notes (cgroups v2)

Modern Linux systems use cgroups v2 with systemd driver. Container paths appear as:
- `/system.slice/docker-<container-id>.scope`

NOT the legacy format:
- `/docker/<container-id>`

The cAdvisor configuration includes flags for cgroups v2 compatibility:
```yaml
command:
  - '--docker_only=true'
  - '--store_container_labels=true'
```

## Grafana Dashboards

Import these dashboards for immediate visibility:

| Dashboard ID | Name | Purpose |
|--------------|------|---------|
| 1860 | Node Exporter Full | Host metrics |
| 14282 | cAdvisor + Node Exporter | Container + host metrics |
| 11159 | cAdvisor Compute Resources | Container resources |

To import: Grafana → Dashboards → Import → Enter ID

## File Structure

```
vps-monitoring-guide/
├── docker-compose.yml          # Main stack definition
├── prometheus/
│   ├── prometheus.yml          # Prometheus configuration
│   └── alerts.yml              # Alert rules
├── alertmanager/
│   ├── alertmanager.yml        # Alertmanager config (gitignored)
│   └── alertmanager.example.yml# Example config
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasources.yml # Auto-configure Prometheus datasource
└── README.md
```

## Security Considerations

1. **Bind to localhost**: All ports are bound to `127.0.0.1` to prevent external access
2. **Use a reverse proxy**: Put Nginx/Caddy in front with authentication
3. **Change default passwords**: Update Grafana admin password immediately
4. **Gitignore secrets**: Never commit `alertmanager.yml` with real SMTP credentials

## Useful Commands

```bash
# Check Prometheus config
docker exec buergerquiz-prometheus promtool check config /etc/prometheus/prometheus.yml

# Reload Prometheus config
curl -X POST http://localhost:9090/-/reload

# View active alerts
curl -s http://localhost:9093/api/v2/alerts | jq

# Query Prometheus
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq

# Check cAdvisor container metrics
curl -s 'http://localhost:9090/api/v1/query?query=container_memory_usage_bytes{name!=""}' | jq '.data.result[] | {name: .metric.name, memory_mb: (.value[1] | tonumber / 1024 / 1024)}'
```

## Troubleshooting

### Prometheus can't scrape app containers
- Check network connectivity: `docker network inspect monitoring`
- Verify app exposes `/api/metrics` endpoint
- Check Prometheus targets page: http://localhost:9090/targets

### cAdvisor shows empty container names
- Ensure `--docker_only=true` flag is set
- Check cgroups version: `cat /sys/fs/cgroup/cgroup.controllers`
- Container names are in `name` label, not `container` label

### Alerts not sending emails
- Check Alertmanager logs: `docker logs buergerquiz-alertmanager`
- Verify SMTP credentials in alertmanager.yml
- Check spam folder

## License

MIT
