# Observability as Code - Architecture Overview

## Executive Summary

This "Observability as Code" platform provides comprehensive monitoring, alerting, and observability for multi-market fintech infrastructure spanning 6 African markets: **market-ke** (Kenya), **market-tz** (Tanzania), **market-gh** (Ghana), **market-drc** (DR Congo), **market-ls** (Lesotho), and **market-mz** (Mozambique).

The platform implements **Zero Touch operations** principles - all infrastructure is defined as code, versioned in Git, and deployed consistently across all markets with automated validation and consistency checking.

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OBSERVABILITY AS CODE PLATFORM                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │   Prometheus │  │  InfluxDB    │  │    Loki      │  │  AlertMgr    │   │
│  │  Time-Series │  │   Metrics    │  │     Logs     │  │  Routing     │   │
│  │   Metrics    │  │   Storage    │  │   Storage    │  │              │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│         │                 │                 │                 │           │
│         └─────────────────┼─────────────────┼─────────────────┘           │
│                           │                 │                             │
│                     ┌─────▼─────────────────▼─────┐                       │
│                     │     Grafana Dashboard       │                       │
│                     │   Visualization Layer       │                       │
│                     └─────┬─────────────────────────┘                       │
│                           │                                               │
│                  ┌────────┼────────┐                                     │
│                  │        │        │                                     │
│        ┌─────────▼──┐ ┌──▼───────┐ │                                     │
│        │  Slack     │ │PagerDuty │ │  ┌───────────────┐                  │
│        │ Webhooks   │ │ Incidents│ │  │ Email Digest  │                  │
│        └────────────┘ └──────────┘ │  └───────────────┘                  │
│                                    │                                     │
│─────────────────────────────────────────────────────────────────────────  │
│                    DATA COLLECTION LAYER                                  │
│                                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │   VPN      │  │ Firewalls  │  │  Hosts     │  │ Services   │        │
│  │  Tunnels   │  │   (FW)     │  │ (Nodes)    │  │  (Apps)    │        │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘        │
│        │               │               │              │                │
│        └───────────────┼───────────────┼──────────────┘                │
│                        │               │                              │
│          ┌─────────────▼───────────────▼────────────┐               │
│          │  Exporters / Collectors / Agents         │               │
│          │  - Node Exporter (hosts)                 │               │
│          │  - Custom VPN exporter                   │               │
│          │  - Firewall syslog collector             │               │
│          │  - Application instrumentation           │               │
│          └──────────────────────────────────────────┘               │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

## Component Descriptions

### Metrics (Prometheus)
- **Purpose**: Collect time-series metrics from infrastructure
- **What it collects**:
  - Host metrics (CPU, memory, disk, network)
  - VPN tunnel health (latency, packet loss, bandwidth)
  - Application performance metrics
  - Security events (authentication, firewall blocks)
- **Retention**: Configured for 30 days of hot storage
- **Scrape interval**: 30 seconds

### Time-Series Storage (InfluxDB)
- **Purpose**: Long-term storage for VPN and custom metrics
- **Retention**: Configurable, typically 1 year
- **Use case**: Historical trend analysis and compliance

### Log Storage (Loki)
- **Purpose**: Centralized logging for all market infrastructure
- **Log types**: VPN connection logs, firewall logs, application logs
- **Integration**: Queryable from Grafana for correlation with metrics

### Alerting (Prometheus AlertManager)
- **Purpose**: Intelligent alert routing and deduplication
- **Features**:
  - Hierarchical alert routing by severity (P1, P2, P3, P4)
  - Alert grouping to prevent alert fatigue
  - Inhibition rules to suppress cascading alerts
  - Integration with multiple notification channels

### Visualization (Grafana)
- **Dashboards**:
  1. **VPN Tunnel Health** - Multi-market VPN tunnel monitoring
  2. **Infrastructure Overview** - Host health across markets
  3. **Security Events** - Security and compliance monitoring
- **Data Sources**: Prometheus, InfluxDB, Loki, AlertManager
- **Auto-provisioning**: All dashboards deployed via code

## Multi-Market Architecture

### Market Structure
Each market has independent infrastructure but unified observability:

```
┌─────────────────────────────────────────────────────────────┐
│                    MARKET INFRASTRUCTURE                     │
│                    (e.g., market-ke)                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Application Cluster                         │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│  │  │ App-KE-1 │ │ App-KE-2 │ │ App-KE-N │           │   │
│  │  └──────────┘ └──────────┘ └──────────┘           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │        VPN Tunnel to Central Hub (HQ)              │   │
│  │  Tunnels: market-ke.vpn.example.com               │   │
│  │  IPs: 192.168.1.0/24 (local), 10.0.0.0/8 (remote) │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Firewall & Security                         │   │
│  │  FW-KE-Primary: 192.168.1.1                        │   │
│  │  FW-KE-Secondary: 192.168.1.2 (HA pair)           │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Monitoring (Node Exporter, Telegraf, etc.)        │   │
│  │  Sends metrics to Central Prometheus               │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### VPN Tunnel Architecture

All markets are connected via managed VPN tunnels to a central hub:

- **Tunnel naming**: `{market}-to-hub.vpn.example.com`
- **Markets**:
  - `market-ke.vpn.example.com` (Kenya)
  - `market-tz.vpn.example.com` (Tanzania)
  - `market-gh.vpn.example.com` (Ghana)
  - `market-drc.vpn.example.com` (DR Congo)
  - `market-ls.vpn.example.com` (Lesotho)
  - `market-mz.vpn.example.com` (Mozambique)
  - `hub.vpn.example.com` (Central Hub)

- **Monitoring**: Each tunnel has dedicated metrics including:
  - Tunnel status (up/down)
  - Latency (ms)
  - Packet loss (%)
  - Bandwidth utilization (Mbps)
  - Jitter (ms)

## Alert Severity Tiers

The system implements a 4-tier severity model aligned with SLA requirements (99.9% uptime target):

### P1 - CRITICAL (Immediate)
- **Response Time**: < 5 minutes
- **Notification**: PagerDuty + Slack #ops-critical
- **Examples**:
  - VPN tunnel down
  - Host/service unreachable
  - Database corruption/failure
  - Security breach detected
  - Certificate expired
  - CPU/Memory/Disk critical (95%+)
- **SLA Impact**: Direct - system is unavailable or severely degraded
- **Action**: Manual incident response, potential customer impact

### P2 - HIGH (Urgent)
- **Response Time**: 15-30 minutes
- **Notification**: Slack #ops-alerts
- **Examples**:
  - High CPU/Memory/Disk (75-95%)
  - Network errors or high packet loss
  - Failed authentication attempts (brute force)
  - Firewall blocks increasing
  - VPN latency degradation
- **SLA Impact**: Potential - risk of escalation to P1
- **Action**: Investigation and remediation during business hours

### P3 - MEDIUM (Important)
- **Response Time**: 1-4 hours
- **Notification**: Email digest (5min batched)
- **Examples**:
  - Certificate expiring (7-15 days)
  - Disk usage high (75-85%)
  - Port scanning detected
  - Suspicious process execution
  - Memory growing steadily
- **SLA Impact**: Minimal - preventive/informational
- **Action**: Plan remediation, schedule maintenance

### P4 - LOW (Informational)
- **Response Time**: 1-2 days
- **Notification**: Email digest (once per day)
- **Examples**:
  - Certificate expiring (>15 days)
  - Disk cleanup recommended
  - Unusual traffic patterns
  - Routine maintenance alerts
- **SLA Impact**: None - informational only
- **Action**: Backlog planning, capacity optimization

## Design Decisions

### 1. Hierarchical Alert Routing
**Decision**: Use AlertManager's route tree instead of flat receiver lists.

**Rationale**:
- Severity-based routing prevents alert fatigue
- Market-specific alerts can be routed to regional teams
- Cascading alerts are inhibited automatically
- Easier to manage as system grows

### 2. Prometheus as Primary Metrics Store
**Decision**: Use Prometheus + InfluxDB hybrid approach.

**Rationale**:
- Prometheus: Hot metrics (30 days), scrape-based, excellent for alerting
- InfluxDB: Long-term storage, excellent for time-series analysis
- Separation of concerns: operational (Prometheus) vs analytical (InfluxDB)

### 3. Custom VPN Metrics Exporter
**Decision**: Develop custom exporter instead of using generic tunnel metrics.

**Rationale**:
- VPN health is critical for multi-market fintech operations
- Standard exporters lack fintech-specific metrics
- Enables correlation between tunnel quality and transaction latency
- Can integrate with ISP API for comprehensive monitoring

### 4. Multi-Market Labels Strategy
**Decision**: Use consistent market labels across all metrics.

**Rationale**:
- Enables dashboards to filter by market
- Supports multi-tenancy alerting rules
- Simplifies compliance and regional reporting

### 5. Infrastructure as Code (IaC)
**Decision**: All configs in Git, deployed consistently.

**Rationale**:
- Zero Touch operations - no manual configuration
- Automated validation on every push
- Version control for audit/compliance
- Disaster recovery - rebuild from Git
- Easy promotion across environments

## Compliance & Security

### Data Residency
- Metrics can be stored per-market region
- Logs are aggregated but tagged with market
- Backup retention aligns with fintech compliance requirements (7 years)

### Security Events Monitoring
- All failed authentication attempts logged
- Port scanning detected and alerted
- Certificate expiration monitored
- Suspicious process execution recorded
- Data access anomalies detected

### Audit Trail
- All alert configurations in Git with commit history
- Grafana dashboards provisioned from code
- Alert routing logic versioned
- Automated validation prevents misconfigurations

## Scaling Considerations

### Horizontal Scaling
- Prometheus federation for multiple data centers
- Alertmanager clustering for HA
- Grafana with shared database backend
- InfluxDB replication for durability

### Adding New Markets
1. Update VPN alert rules with new market regex
2. Add new datasource tags in exporters
3. Update dashboard drilldown filters
4. Deploy via GitHub Actions CI/CD
5. Grafana dashboards automatically reflect new metrics

### Growing Alert Volume
- Alert inhibition rules prevent cascading (P1 → P2/P3/P4)
- Alert grouping combines related alerts
- Repeat intervals prevent notification spam

## References

- [Prometheus Documentation](https://prometheus.io/docs/)
- [AlertManager Routing](https://prometheus.io/docs/alerting/latest/configuration/)
- [Grafana Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [Best Practices for Observability](https://github.com/example-com/observability-standards)
