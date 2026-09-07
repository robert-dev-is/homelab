# Prometheus

## Role

Prometheus is the centralized metrics collection and time-series backend for the observability platform.

It runs in `lab-monitor-01` and provides the authoritative metrics source used by Grafana.

## Responsibilities

- Scrape infrastructure exporters
- Store historical time-series data
- Provide PromQL queries for Grafana
- Track exporter and target availability
- Apply consistent labels to monitored targets

## Target Organization

Scrape jobs are organized by monitoring function.

Example UPS job:

```yaml
  - job_name: 'ups'
    static_configs:

      - targets: ['10.10.10.61:9199']
        labels:
          hostname: ups-monitor-01
          role: ups
```

The platform uses labels such as `hostname` and `role` to make targets easier to identify in Prometheus and Grafana.

## Proxmox Metrics

Proxmox metrics are collected through `prometheus-pve-exporter`, which queries the Proxmox VE API and exposes the results in Prometheus format.

This avoids requiring a Proxmox-specific exporter installation on every cluster node.

## Host Metrics

Linux and Proxmox host operating-system metrics are collected with `node_exporter`.

Typical host metrics include:

- CPU utilization
- Memory utilization
- Filesystem usage
- Network activity
- Host availability

## Configuration Validation

Before restarting Prometheus after configuration changes:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Prometheus can then be restarted with:

```bash
systemctl restart prometheus
```

## Verification

The Prometheus Targets page is used to confirm that configured exporters are reachable and successfully scraping.

Direct PromQL queries are also useful when separating collection problems from Grafana query or panel problems.
