# Exporters and Collectors

## Overview

The platform uses specialized exporters and collectors to translate infrastructure telemetry into Prometheus-compatible metrics.

## node_exporter

`node_exporter` provides operating-system metrics from Linux and Proxmox hosts.

It is used for host-level telemetry such as:

- CPU
- Memory
- Filesystems
- Network interfaces
- System availability

## Proxmox Metrics Translator

`pve-metrics-translator-01` runs the Python-based `prometheus-pve-exporter`.

Its role is to translate Proxmox VE API data into metrics that Prometheus can scrape.

```text
Proxmox VE API
       ↓
prometheus-pve-exporter
       ↓
Prometheus metrics
       ↓
Prometheus
```

The exporter is installed in:

```text
/pve-exporter
```

It replaced the previously used `cv4pve-metrics-exporter` after problems were encountered with stale guest metadata during normal Proxmox changes.

## NUT Exporter

`ups-monitor-01` runs DRuggeri `nut_exporter` v3.3.0.

The exporter is installed in:

```text
/nut-exporter
```

It connects to the local NUT server and exposes UPS telemetry on TCP port `9199`.

The UPS metrics endpoint is:

```text
/ups_metrics
```

## Why Translators Matter

Not every infrastructure component natively exposes Prometheus metrics.

The collection layer therefore acts as a translation boundary:

```text
Native API / telemetry
        ↓
Exporter / translator
        ↓
Prometheus exposition format
        ↓
Prometheus
```

This keeps Prometheus independent of vendor-specific APIs while allowing Grafana to query a consistent metrics backend.

## Operational Considerations

Exporters should:

- Reflect current infrastructure state
- Avoid retaining stale identity metadata
- Remain independently testable with `curl`
- Expose predictable Prometheus metric names
- Survive service restarts and infrastructure changes
