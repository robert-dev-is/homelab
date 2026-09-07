# Lessons Learned

## Dynamic Infrastructure Requires Dynamic Monitoring

Virtual machines and containers are not static objects. They can be migrated between Proxmox nodes, renamed, created, and removed.

Monitoring software must handle these changes without retaining stale identity information.

## Exporter Metadata Can Break PromQL

The original `cv4pve-metrics-exporter` deployment exposed stale `guest_info` series after guest migrations and renames.

Examples included the same guest ID appearing with:

- An old node and a new node
- A current guest name and a blank name
- A current guest name and a generated name such as `CT109`

This caused Prometheus binary joins to fail with many-to-many matching errors because the right-hand metadata series was no longer unique for a guest ID.

## Query-Side Deduplication Is Useful but Not a Complete Fix

PromQL aggregation could temporarily make metadata unique:

```promql
max by (id, name, vmid, type) (
  cv4pve_guest_info{name!="", name!~"CT[0-9]+"}
)
```

This allowed Grafana panels to continue working in some cases.

However, query-side filtering does not fix an exporter that continues producing stale metadata.

## Restarting an Exporter Does Not Remove Historical Prometheus Data

Restarting the original exporter cleared its current stale in-memory metrics, but Grafana queries covering an earlier time range could still encounter historical duplicate series already stored by Prometheus.

This reinforced the difference between:

- Current exporter state
- Prometheus historical time-series data
- Grafana query behavior over a selected time range

## Replace Components That Do Not Fit the Environment

The original exporter became increasingly troublesome as normal Proxmox changes occurred.

Rather than continually compensating for exporter behavior in Grafana queries, the Proxmox collection layer was replaced with the Python-based `prometheus-pve-exporter`.

The installation itself was lightweight, and the new exporter better matched the requirement that monitoring remain reliable as the Proxmox environment changes.

## Test Each Layer Independently

A useful troubleshooting pattern emerged:

```text
Source telemetry
      ↓
Exporter endpoint
      ↓
Prometheus target
      ↓
PromQL result
      ↓
Grafana panel
```

Testing each layer independently makes it much easier to determine whether a problem is caused by:

- The monitored service
- The exporter
- Prometheus scraping
- PromQL
- Grafana configuration

Direct `curl` requests to exporter and Prometheus endpoints were especially useful.

## Export Only Useful Telemetry

The NUT exporter initially exposed a limited default set of UPS variables.

Reviewing the UPS data available through `upsc` showed that additional useful values were available, including runtime, output voltage, battery thresholds, and nominal real power.

The exporter variable list was expanded rather than accepting the defaults unchanged.

## Prefer Understandable Infrastructure

The monitoring platform is intentionally built from components whose roles can be inspected and tested independently.

That makes failures easier to reason about and allows individual components to be replaced without redesigning the entire observability stack.
