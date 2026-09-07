# Dashboards and Metrics

## Overview

The Grafana dashboards focus on metrics that provide useful operational visibility into the homelab.

## Host Metrics

Useful host panels include:

- Host availability
- CPU utilization
- CPU utilization over time
- Memory utilization
- Memory utilization over time

## Proxmox Guest Metrics

Guest dashboards track both virtual machines and LXC containers.

### Inventory

Useful summary values include:

- Total guests
- Total LXC containers
- Total QEMU virtual machines
- Running LXC containers
- Running QEMU virtual machines

### CPU

Guest CPU panels display current utilization and historical utilization.

Legends use:

```text
{{name}} ({{type}} {{vmid}})
```

### Memory

Guest memory panels display utilization as a percentage of configured guest memory.

The same legend format is used for consistency.

## UPS Metrics

The NUT exporter exposes metrics including:

```text
network_ups_tools_battery_charge
network_ups_tools_battery_charge_low
network_ups_tools_battery_charge_warning
network_ups_tools_battery_runtime
network_ups_tools_battery_runtime_low
network_ups_tools_battery_voltage
network_ups_tools_input_voltage
network_ups_tools_output_voltage
network_ups_tools_ups_load
network_ups_tools_ups_realpower_nominal
network_ups_tools_ups_status
```

### Useful UPS Panels

- Battery charge (%)
- Runtime remaining
- UPS load (%)
- Input voltage
- Output voltage
- UPS online status
- On-battery status
- Low-battery status

Status metrics are exposed as flags, for example:

```promql
network_ups_tools_ups_status{flag="OL"}
```

A value of `1` indicates that the status is active.

## Dashboard Direction

The platform is intended to grow toward a consolidated infrastructure overview suitable for both interactive troubleshooting and a dedicated status display.
