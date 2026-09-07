# UPS Monitoring

## Overview

UPS telemetry is collected from a CyberPower CP1500PFCRM2U using Network UPS Tools (NUT) and exposed to Prometheus through a dedicated NUT exporter.

The monitoring services run in `ups-monitor-01`.

## Hardware

| Item | Value |
|---|---|
| UPS | CyberPower CP1500PFCRM2U |
| Rated Real Power | 1000 W |
| Connection | USB HID |
| NUT Driver | `usbhid-ups` |
| Vendor ID | `0764` |
| Product ID | `0601` |

The UPS reports a placeholder serial value, so the configuration does not depend on serial-number matching.

USB bus and device numbers are treated as dynamic troubleshooting information rather than fixed configuration values.

## NUT

NUT communicates directly with the UPS through the `usbhid-ups` driver.

Communication can be verified with:

```bash
upsc cyberpower
```

Available telemetry includes:

- Battery charge
- Battery runtime
- Battery voltage
- Input voltage
- Output voltage
- UPS load
- Online/on-battery state
- Battery thresholds

## Prometheus Exporter

DRuggeri `nut_exporter` v3.3.0 is installed under:

```text
/nut-exporter
```

The exporter connects to the local NUT server:

```text
127.0.0.1:3493
```

and listens on:

```text
10.10.10.61:9199
```

UPS metrics are exposed at:

```text
/ups_metrics
```

## Exported Variables

The default exporter variable set was expanded to include:

```text
battery.charge
battery.charge.low
battery.charge.warning
battery.runtime
battery.runtime.low
battery.voltage
battery.voltage.nominal
input.voltage
input.voltage.nominal
output.voltage
ups.load
ups.realpower.nominal
ups.status
```

## Prometheus Integration

Prometheus uses the following scrape job:

```yaml
  - job_name: 'ups'
    static_configs:

      - targets: ['10.10.10.61:9199']
        labels:
          hostname: ups-monitor-01
          role: ups
```

## Monitoring Flow

```text
CyberPower CP1500PFCRM2U
          ↓
        USB HID
          ↓
     usbhid-ups
          ↓
     NUT / upsd
          ↓
     nut_exporter
          ↓
      Prometheus
          ↓
        Grafana
```

## Future Integration

UPS telemetry can also support automated power management, including graceful shutdown sequencing during extended outages and controlled recovery after utility power returns.
