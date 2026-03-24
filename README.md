# Observability as Code - Multi-Market Fintech Infrastructure

[![Validate Configs](https://github.com/example-com/observability-as-code/workflows/validate_configs/badge.svg)](https://github.com/example-com/observability-as-code/actions)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
![Platforms](https://img.shields.io/badge/Markets-6-blue)
![Status](https://img.shields.io/badge/Status-Production-brightgreen)

## Problem Statement

Operating fintech infrastructure across 6 African markets presents unique observability challenges:

- **Geographic Dispersion**: Servers distributed across 6 markets.
- **VPN Dependency**: Mission-critical applications depend on inter-market VPN tunnels that are prone to latency and packet loss
- **Compliance Requirements**: Financial regulations require comprehensive audit trails and incident response documentation
- **No Slack**: 99.9% uptime SLA requires rapid incident detection and response (< 5 minutes for P1 alerts)
- **Manual Sprawl**: Without infrastructure-as-code, monitoring configs drift and become inconsistent across markets
- **Silent Failures**: Traditional monitoring misses pre-failure degradation until customer impact

**This project** implements a comprehensive, production-grade observability platform that detects issues before they impact customers and operationalizes incident response.

## What's Included

### 📊 Grafana Dashboards (3 production dashboards)
1. **VPN Tunnel Health** - Multi-market tunnel monitoring (status, latency, packet loss, bandwidth)
2. **Infrastructure Overview** - Host health across all markets (CPU, memory, disk, availability)
3. **Security Events** - Real-time security monitoring (failed auths, firewall blocks, anomalies)

### 🚨 Prometheus Alert Rules (50+ alerts)
- **Infrastructure**: CPU, memory, disk, network, load average, swap usage
- **VPN**: Tunnel status, latency, packet loss, bandwidth, jitter, flapping, degradation
- **Security**: Failed authentication, brute force, firewall blocks, port scanning, anomalies, certificate expiration

### ⚙️ AlertManager Configuration
- **Hierarchical routing**: P1→PagerDuty, P2→Slack, P3→Email digest, P4→Low-priority digest
- **Alert inhibition**: P1 alerts automatically suppress P2/P3/P4 cascading alerts
- **Smart grouping**: Related alerts grouped to reduce noise while maintaining context

### 📖 Comprehensive Documentation
- [Architecture Overview](docs/architecture.md) - System design, multi-market structure, compliance considerations
- [Alert Runbook](docs/alert_runbook.md) - First-response actions for every alert with examples
- GitHub Actions CI/CD for automated configuration validation

## Alert Severity Matrix

| Severity | Response Time | Channel | Examples | SLA Impact |
|----------|---------------|---------|----------|-----------|
| **P1 - CRITICAL** | < 5 min | PagerDuty + Slack #ops-critical | VPN down, host unreachable, cert expired, critical resource usage | Direct: Service unavailable |
| **P2 - HIGH** | 15-30 min | Slack #ops-alerts | High CPU/memory/disk, network errors, brute force, tunnel degradation | Potential: Escalation risk |
| **P3 - MEDIUM** | 1-4 hours | Email digest (5min batch) | Cert expiring soon, medium resource usage, port scanning | Minimal: Preventive |
| **P4 - LOW** | 1-2 days | Email digest (daily) | Cert expiring far future, routine maintenance | None: Informational |

## Quick Start

### Prerequisites
- Docker & Docker Compose
- Git
- Make (optional but recommended)

### Installation

```bash
# Clone repository
git clone https://github.com/example-com/observability-as-code.git
cd observability-as-code

# Copy environment template and fill in values
cp .env.example .env
# Edit .env with your settings:
# - SLACK_WEBHOOK_URL
# - PAGERDUTY_SERVICE_KEY
# - SMTP credentials
# - Database passwords
vi .env

# Deploy infrastructure
docker-compose up -d

# Verify deployment
docker-compose logs -f prometheus | head -20
curl http://localhost:9093  # AlertManager
curl http://localhost:3000  # Grafana (admin/admin)
curl http://localhost:9090  # Prometheus
```

### Accessing Components

- **Grafana**: http://localhost:3000 (login: admin/admin)
- **Prometheus**: http://localhost:9090
- **AlertManager**: http://localhost:9093
- **InfluxDB**: http://localhost:8086

### First Steps

1. **Configure data sources** in Grafana:
   - Go to Configuration → Data Sources
   - Verify Prometheus, InfluxDB, Loki, AlertManager are connected

2. **Import dashboards**:
   - Dashboards are auto-provisioned from `grafana/dashboards/*.json`
   - Check Dashboards menu to verify all 3 are present

3. **Configure notifications**:
   - Edit `.env` with your Slack/PagerDuty/Email credentials
   - Restart AlertManager: `docker-compose restart alertmanager`
   - Test by triggering sample alert

4. **Add data sources**:
   - Point exporters to Prometheus: `http://prometheus:9090`
   - Configure host exporters: `node_exporter`, `telegraf`
   - Verify metrics appear in Prometheus UI

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│             Multi-Market Fintech Infrastructure             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Market-KE  Market-TZ  Market-GH  Market-DRC  Market-LS    │
│  (Kenya)    (Tanzania) (Ghana)    (DR Congo)  (Lesotho)    │
│    Applications / Databases / Caches                        │
│         via VPN Tunnels (Inter-market connectivity)         │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                      Observability Layer                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Prometheus  │  │   InfluxDB   │  │     Loki     │     │
│  │   Metrics    │  │   Storage    │  │     Logs     │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Grafana Dashboards (VPN Health, Infrastructure,    │  │
│  │  Security Events) with Market Filtering             │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  AlertManager (Hierarchical Routing)                │  │
│  │  P1→PagerDuty  P2→Slack  P3/P4→Email               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Multi-Market Deployment

Each market has independent infrastructure but unified observability:

### VPN Tunnel Naming Convention
- `market-ke.vpn.example.com` - Kenya
- `market-tz.vpn.example.com` - Tanzania
- `market-gh.vpn.example.com` - Ghana
- `market-drc.vpn.example.com` - DR Congo
- `market-ls.vpn.example.com` - Lesotho
- `market-mz.vpn.example.com` - Mozambique

### Adding a New Market

1. Update PrometheusAlertManager rules with new market regex
2. Configure VPN exporter with new tunnel metrics
3. Update Grafana dashboard filters (auto-populated from metrics)
4. Deploy: `git push origin new-market-config`
5. GitHub Actions validates automated
6. Metrics flow to central hub within 30 seconds

## Technologies

| Component | Technology | Purpose | Version |
|-----------|-----------|---------|---------|
| Metrics Collection | Prometheus | Time-series metrics, alerting | v2.40+ |
| Metrics Storage | InfluxDB | Long-term time-series | v2.0+ |
| Visualization | Grafana | Dashboard & alerting UI | v9.0+ |
| Alert Routing | AlertManager | Alert deduplication, routing | v0.24+ |
| Log Storage | Loki | Log aggregation & search | v2.6+ |
| Exporters | Node Exporter, Custom | Infrastructure metrics | node_exporter v1.5+ |
| CI/CD Validation | GitHub Actions | Configuration validation | Built-in |
| Infrastructure | Docker Compose | Local/testing deployment | v20.0+ |
| IaC Format | YAML/JSON | Infrastructure as code | - |
| Language | Python, Bash | Validation scripts | v3.11, v5.0+ |

## Project Structure

```
observability-as-code/
├── grafana/
│   ├── dashboards/                    # Grafana dashboards (JSON)
│   │   ├── vpn_tunnel_health.json     # VPN monitoring
│   │   ├── infrastructure_overview.json   # Host health
│   │   └── security_events.json       # Security dashboard
│   └── provisioning/                  # Auto-provisioning configs
│       ├── dashboards.yaml
│       └── datasources.yaml
├── alertmanager/
│   ├── alertmanager.yml               # Alert routing rules
│   └── templates/
│       └── notification.tmpl          # Email/Slack templates
├── prometheus/
│   └── rules/                         # Prometheus alert rules
│       ├── infrastructure_alerts.yaml  # Host/system alerts
│       ├── vpn_alerts.yaml            # VPN tunnel alerts
│       └── security_alerts.yaml       # Security alerts
├── .github/
│   └── workflows/
│       └── validate_configs.yml       # CI/CD validation
├── docs/
│   ├── architecture.md                # System design
│   ├── alert_runbook.md              # Alert response guide
│   └── README.md                      # This file
├── docker-compose.yml                 # Local test environment
├── .env.example                       # Environment template
├── Makefile                          # Make tasks (optional)
└── README.md                         # This file
```

## Configuration

### Environment Variables (.env)

```bash
# Slack notifications
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
SLACK_WEBHOOK_URL_OPS=https://hooks.slack.com/services/...
SLACK_WEBHOOK_URL_SECURITY=https://hooks.slack.com/services/...
SLACK_WEBHOOK_URL_CRITICAL=https://hooks.slack.com/services/...

# PagerDuty
PAGERDUTY_SERVICE_KEY=your_pagerduty_key_here

# Email
EMAIL_FROM_ADDRESS=alerts@example.com
EMAIL_OPS_TEAM=ops-team@example.com
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USERNAME=your_email@example.com
SMTP_PASSWORD=your_smtp_password

# Data sources
PROMETHEUS_URL=http://prometheus:9090
INFLUXDB_URL=http://influxdb:8086
INFLUXDB_DATABASE=observability
INFLUXDB_API_KEY=your_influxdb_api_key
LOKI_URL=http://loki:3100
ALERTMANAGER_URL=http://alertmanager:9093

# Grafana
GRAFANA_ADMIN_PASSWORD=your_secure_password
```

### Alert Customization

Edit alert thresholds in `prometheus/rules/*.yaml`:

```yaml
- alert: HighCPUUsage
  expr: cpu_usage > 85    # Change threshold from 85% to desired value
  for: 5m                 # Change duration from 5m to x(m/h)
  labels:
    severity: P2          # Change P2 to P1/P3/P4
```

### Adding Custom Metrics

1. Create exporter that outputs Prometheus metrics
2. Add scrape config to `prometheus.yml`
3. Create alert rule in appropriate `prometheus/rules/*.yaml`
4. Add dashboard panel in Grafana
5. Commit and push (GitHub Actions validates)

## Operational Tasks

### Manual Validation

```bash
# Validate YAML
yamllint alertmanager/ prometheus/

# Validate JSON dashboards
for file in grafana/dashboards/*.json; do
  python -m json.tool "$file" > /dev/null && echo "✓ $file" || echo "✗ $file"
done

# Validate Prometheus alert rules
promtool check rules prometheus/rules/*.yaml

# Full config validation
docker-compose config
```

### Testing Alerts

```bash
# Trigger test alert
docker exec prometheus promtool alert test prometheus/rules/infrastructure_alerts.yaml <<EOF
groups:
- name: test
  interval: 1m
  rules:
  - alert: TestAlert
    expr: up{job="non-existent"} == 0
    for: 1m
    labels:
      severity: P1
    annotations:
      summary: "Test alert"
EOF

# View AlertManager status
curl http://localhost:9093/api/v1/alerts
```

### Disaster Recovery

```bash
# Backup all configs
tar -czf observability-backup-$(date +%Y%m%d).tar.gz \
  grafana/ alertmanager/ prometheus/ .env

# Restore from Git
git status  # See what changed
git checkout -- .  # Reset to committed state
docker-compose restart  # Redeploy
```

## Monitoring the Monitors

It's critical to monitor the observability stack itself:

- **Prometheus Health**: Alert if Prometheus scraping target fails
- **AlertManager Health**: Monitor AlertManager uptime
- **Grafana Health**: Check Grafana connectivity
- **InfluxDB/Loki**: Monitor storage disk usage

These are implemented as self-referential alerts - the system monitors itself to catch failures early.

## Security Considerations

This project handles **sensitive infrastructure data**:

1. **Credentials**: All stored in `.env`, NEVER committed to Git (.gitignore protects this)
2. **API Keys**: Use separate keys for each market/service
3. **Network Access**: Run behind VPN or firewall, restrict to ops team
4. **Audit Logs**: AlertManager logs all notifications for compliance
5. **RBAC**: Implement Grafana role-based access per market team

## Contributing

This is a portfolio project demonstrating professional observability practices. To adapt for your infrastructure:

1. Fork this repository
2. Update market names, hostnames, VPN tunnel names
3. Customize thresholds based on your SLAs
4. Add your exporters/metrics
5. Configure notification channels
6. Test in staging before production

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## Troubleshooting

### Alerts not firing?
1. Check Prometheus targets: http://localhost:9090/targets
2. Verify alert rule syntax: `promtool check rules prometheus/rules/*.yaml`
3. Check AlertManager logs: `docker-compose logs alertmanager`

### Dashboards not loading?
1. Verify datasources in Grafana UI (Configuration → Data Sources)
2. Check metrics available: http://localhost:9090/graph
3. Try manual query in Prometheus Graph tab

### Notifications not working?
1. Verify credentials in `.env`
2. Test manually: `curl -X POST SLACK_WEBHOOK_URL -d '{...}'`
3. Check AlertManager metrics for notification errors
4. Review AlertManager logs for errors

## Performance & Scaling

- **Prometheus**: 1 instance handles 10M+ time-series for 6 markets
- **Grafana**: 100+ concurrent users per instance
- **AlertManager**: Single instance handles 10K+ alerts/min
- **For vertical scaling**: Increase resource limits in docker-compose.yml
- **For horizontal scaling**: See docs/architecture.md

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) file for details.

### Portfolio Context

This project demonstrates production-grade infrastructure observability expertise applied to real-world constraints of African fintech operations. It showcases:

- **Zero Touch Operations**: Infrastructure fully defined as code
- **Multi-tenancy**: Scalable to any number of markets
- **Production Readiness**: 99.9% uptime SLA alignment
- **Compliance Focus**: Audit trails, incident response runbooks
- **Operational Excellence**: Automated validation, clear runbooks, monitoring the monitors

Based on real experience managing infrastructure across 6 markets with strict financial SLAs and zero tolerance for silent failures.

## Contact & Support

- **Author**: DevOps Infrastructure Engineer
- **Portfolio**: github.com/example-com
- **Demo Issues**: Sanitized examples - no actual credentials included
- **Questions**: Open an issue on GitHub

---

**⭐ If this demonstrates useful patterns for your team, please give it a star!**

Last Updated: 2024 | Production Ready
