# Architecture

## Overview

The observability platform separates metric collection, translation, storage, and visualization into distinct roles.

Prometheus acts as the authoritative metrics backend. Grafana queries Prometheus rather than collecting metrics directly.

## Architecture

```text
                         ┌──────────────────────┐
                         │   Proxmox VE API     │
                         └──────────┬───────────┘
                                    │
                         prometheus-pve-exporter
                                    │
                                    ▼
┌─────────────────┐       ┌─────────────────────┐       ┌──────────────────┐
│ Linux / PVE     │──────▶│                     │──────▶│                  │
│ node_exporter   │       │     Prometheus      │       │     Grafana      │
└─────────────────┘       │                     │       │                  │
                          └─────────────────────┘       └──────────────────┘
                                    ▲
                                    │
                            NUT Prometheus
                               exporter
                                    │
                         ┌──────────┴───────────┐
                         │ NUT / CyberPower UPS │
                         └──────────────────────┘
```

## Design Principles

### Centralized Metrics

Prometheus provides a single backend for infrastructure telemetry. Grafana dashboards use Prometheus as their data source.

### Specialized Collectors

Different systems expose telemetry differently. Exporters and translators normalize those sources into Prometheus-compatible metrics.

### Service Separation

Monitoring roles are separated by function. Prometheus and Grafana run independently, while exporters are placed where they logically belong with the service or hardware they monitor.

### Dynamic Infrastructure

Virtual machines and containers are expected to migrate, be renamed, be created, and be removed. Monitoring components should reflect the current infrastructure state rather than assume guest metadata is static.

## Current Monitoring Paths

### Linux Hosts

```text
Linux / Proxmox Host
        ↓
   node_exporter
        ↓
    Prometheus
        ↓
      Grafana
```

### Proxmox VE

```text
Proxmox VE API
       ↓
prometheus-pve-exporter
       ↓
   Prometheus
       ↓
     Grafana
```

### UPS

```text
CyberPower UPS
       ↓
    USB HID
       ↓
Network UPS Tools
       ↓
   nut_exporter
       ↓
    Prometheus
       ↓
      Grafana
```
