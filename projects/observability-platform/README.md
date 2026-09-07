# Observability Platform

## Overview

Homelab observability platform built around Prometheus and Grafana for centralized infrastructure telemetry, historical performance data, and operational visibility.

The platform collects metrics from physical hosts, Proxmox VE, virtual machines and containers, UPS infrastructure, and other services through purpose-built exporters and translators.

## Goals

- Centralize infrastructure metrics in Prometheus
- Provide readable Grafana dashboards for at-a-glance operational awareness
- Track historical CPU, memory, power, availability, and service metrics
- Keep monitoring components modular and understandable
- Support infrastructure changes without requiring static dashboard definitions
- Provide a foundation for alerting and automated power management

## Core Components

| Component | Role |
|---|---|
| Prometheus | Metrics collection and time-series storage |
| Grafana | Visualization and dashboards |
| node_exporter | Linux host metrics |
| prometheus-pve-exporter | Proxmox VE API metrics |
| NUT | UPS communication and status |
| nut_exporter | Prometheus-compatible UPS metrics |

## High-Level Flow

```text
Infrastructure
├── Linux / Proxmox hosts
├── Proxmox VE API
├── UPS / NUT
└── Other monitored services
          ↓
Exporters / Translators
          ↓
      Prometheus
          ↓
        Grafana
          ↓
Infrastructure Dashboards
```

## Project Documentation

- [architecture.md](architecture.md) - platform design and telemetry flow
- [prometheus.md](prometheus.md) - Prometheus backend and scrape organization
- [grafana.md](grafana.md) - visualization and dashboard design
- [exporters-and-collectors.md](exporters-and-collectors.md) - metric collection components
- [dashboards-and-metrics.md](dashboards-and-metrics.md) - dashboards and useful metrics
- [ups-monitoring.md](ups-monitoring.md) - NUT and CyberPower UPS telemetry
- [lessons-learned.md](lessons-learned.md) - engineering findings and design changes
