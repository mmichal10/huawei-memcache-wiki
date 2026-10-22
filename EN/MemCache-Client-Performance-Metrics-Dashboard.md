## 1. Introduction

MemCache provides an out-of-the-box **client metrics dashboard** covering key operational metrics such as BATCH_PUT / BATCH_GET latency by rank, data transfer bandwidth, and UBS IO storage resources. Metrics come from the built-in client metrics collection framework, are pushed to Meta Service every 30 seconds, and are exposed in Prometheus format through the `/metrics` endpoint.

> **Prerequisites:** For Prometheus + Grafana deployment and basic integration, see [the MemCache Grafana + Prometheus integration tutorial](https://gitcode.com/Ascend/memcache/wiki/memcache%E5%AF%B9%E6%8E%A5Grafana+Prometheus%E4%BD%BF%E7%94%A8%E6%95%99%E7%A8%8B.md).


## 2. Import the Dashboard

### 2.1 Configure Prometheus

Add a scrape configuration targeting the `/metrics` endpoint in `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'memcache-client'
    scrape_interval: 15s
    static_configs:
      - targets: ['<meta_host>:<port>']
    metrics_path: '/metrics'
```

### 2.2 Import into Grafana

1. Add the Prometheus instance above as a data source.
2. Import the prebuilt dashboard: [example/metrics/memcache_dashboard_client_metrics.json](https://gitcode.com/Ascend/memcache/blob/master/example/metrics/memcache_dashboard_client_metrics.json)


## 3. Dashboard Overview

### 3.1 PUT / GET Performance（30s Window）

Use this section to monitor PUT/GET latency and instantaneous bandwidth over a 30-second window for each rank's client.

| Panel | Operation | Description |
|------|----------|------|
| **Get Latency** | BatchGet | BatchGet latency quantiles (P50 / P90 / P99 / Avg) over the client's 30-second window, in seconds |
| **Get Bytes** | BatchGet | Bytes transferred (totalBytes) by BatchGet over the client's 30-second window |
| **Get Bandwidth** | BatchGet | Instantaneous BatchGet bandwidth over the client's 30-second window (B/s) |
| **Put Latency** | BatchPut | BatchPut latency quantiles (P50 / P90 / P99 / Avg) over the client's 30-second window, in seconds |
| **Put Bytes** | BatchPut | Bytes transferred (totalBytes) by BatchPut over the client's 30-second window |
| **Put Bandwidth** | BatchPut | Instantaneous BatchPut bandwidth over the client's 30-second window (B/s) |

Each latency panel shows multiple curves by `rank` × `quantile`: **P50 / P90 / P99 / Avg**. Bandwidth panels show curves by `rank`. Use the Rank filter to focus on a node and the Quantile filter to select distribution curves.

### 3.2 PUT / GET Performance（Cumulative）

Use this section to monitor each rank's cumulative client transfer volume and average throughput over the process lifetime.

| Panel | Content | Description |
|------|------|------|
| **Cumulative Latency** | Cumulative IO time | Total IO time since process startup (seconds) |
| **Cumulative Bytes** | Cumulative bytes transferred | Total bytes transferred since process startup |
| **Cumulative Bandwidth** | Average bandwidth over process lifetime | Average throughput over the process lifetime (B/s) |

The cumulative panels are grouped by `rank` × `operation` (batch_put / batch_get) and support the `$rank` filter.

### 3.3 UBS IO Storage

Use this section to monitor UBS IO WriteCache storage resource utilization, disk health, and per-disk bandwidth for each rank's client.

| Panel | Metric | Description |
|------|----------|------|
| **SSD Capacity & Usage** | SSD total capacity vs. used capacity | Shows `disk_capacity_bytes` and `disk_used_bytes` to monitor SSD space utilization |
| **Memory Capacity & Usage** | Total memory vs. used memory | Shows `mem_capacity_bytes` and `mem_used_bytes` to monitor memory utilization |
| **Per-Disk Read Bandwidth** | Read bandwidth per disk (B/s) | Shows read throughput per disk, labeled by `disk_path`, to quickly identify slow readers |
| **Per-Disk Write Bandwidth** | Write bandwidth per disk (B/s) | Shows write throughput per disk, labeled by `disk_path`, to quickly identify slow writers |
| **Per-Disk Total Bandwidth** | Total bandwidth per disk (B/s) | Shows total throughput (combined reads and writes) per disk, labeled by `disk_path` |
| **Disk Count & Faults** | Total disks vs. faulty disks | Monitors disk health and triggers an alert when faulty disk count > 0 |

All panels support the `$rank` filter. Per-disk bandwidth panels also support the `$disk_path` filter.

### 3.4 Dashboard Screenshot

![image.png](https://raw.gitcode.com/user-images/assets/10261984/fa5e1ff3-0e1e-496f-82c4-f79b502e199d/image.png 'image.png')

![image.png](https://raw.gitcode.com/user-images/assets/7672915/1e883311-f43e-4c2d-81b6-92b6afbe443b/image.png 'image.png')


## 4. Using the Dashboard

### 4.1 Variable Filters

The top of the dashboard provides four dropdown filters:

| Variable | Description | Usage |
|------|------|------|
| **Datasource** | Selects the Prometheus data source | Matched automatically during import; no manual changes needed |
| **Rank (Client)** | Filters by client rank; supports multi-select / select all | Affects all panels below (window bandwidth/latency, cumulative, and UBS IO). Select ranks individually to locate node-specific issues |
| **Quantile** | Filters by latency quantile; supports multi-select / select all | Affects only the 30s Window latency panels (Get Latency / Put Latency); values are `P50`, `P90`, `P99`, and `Avg` |
| **Disk Path** | Filters by block device path; supports multi-select / select all | Affects only per-disk bandwidth panels (read/write/total bandwidth) |

### 4.2 Legend

Legends for all panels appear at the bottom in Table mode and automatically calculate **mean** (window average) and **lastNotNull** (latest non-null value). Hover in multi mode to display the current values for all series simultaneously.

- Latency panels: `{{rank}} @ {{quantile}}`
- Bandwidth panels: `{{rank}} @ {{operation}}`
- Capacity panels: compare the `{{rank}} @ capacity` and `{{rank}} @ used` curves
- Per-disk bandwidth panels: `{{rank}} @ {{disk_path}}`
- Disk health panels: compare the `{{rank}} @ total` and `{{rank}} @ fault` curves

### 4.3 Refresh Interval

The dashboard refreshes automatically every 10 seconds by default. Adjust the interval from the dropdown in the upper-right corner.



## 5. Performance Troubleshooting Guide

### 5.1 Troubleshooting High Latency

**Scenario 1: High client transfer latency**

1. Open the **Get Latency** / **Put Latency** panels (30s window).
2. In the Quantile dropdown, select only `P99` and `Avg` and check whether tail latency continues to rise.
3. Select ranks one at a time to distinguish system-wide degradation from a single-node issue.
4. If P99 is much higher than P50, occasional operations on large keys or network jitter may be responsible.
5. For a single-node issue, check that node's network link and load.

**Scenario 2: Latency rises across all ranks**

1. Possible causes:
   - A Meta-side processing bottleneck is making clients wait.
   - Network congestion (confirm using the bandwidth panels).
2. Cross-check whether latency is also rising in the Meta-side panels.

### 5.2 Troubleshooting Bandwidth Anomalies

**Scenario 1: Sudden bandwidth drop**

1. Open the **Get Bandwidth** / **Put Bandwidth** panels (30s window).
2. Check whether bandwidth curves fall at the same time across ranks.
3. Possible causes of a simultaneous drop:
   - Overall workload traffic has decreased.
   - Network congestion or a link failure.
   - A Meta-side processing bottleneck is making clients wait.
4. For a drop on one rank, check whether its client process is running and its network link is healthy.

**Scenario 2: Uneven bandwidth across ranks**

1. Open the **Get Bandwidth** / **Put Bandwidth** panels.
2. Compare bandwidth by rank to locate the hot rank.
3. Cross-check whether latency is also rising for that rank.

### 5.3 Troubleshooting Cumulative Metrics

1. Open the **Cumulative Latency** / **Cumulative Bytes** panels.
2. Curves should increase monotonically. A drop or reset to zero indicates that the client restarted.
3. Open the **Cumulative Bandwidth** panel.
4. A sustained decline in lifetime average bandwidth indicates high traffic early in the process lifetime followed by lower traffic.
5. When cumulative transfer volume and window bandwidth follow consistent trends, the system is in a steady state.

### 5.4 Troubleshooting UBS IO Storage Issues

**Scenario 1: SSD utilization keeps increasing**

1. Open the **SSD Capacity & Usage** panel.
2. Check whether the `used` curve is increasing abnormally.
3. Possible causes:
   - A sudden increase in write traffic with expired data not being cleaned up in time.
   - An SSD failure reducing available capacity (cross-check using the **Disk Count & Faults** panel).
4. Check whether the corresponding rank's PUT bandwidth has also risen, and log in to the node to inspect actual SSD mount-point utilization.

**Scenario 2: Memory utilization is unusually high**

1. Open the **Memory Capacity & Usage** panel.
2. Possible causes include an undersized WriteCache memory pool or a memory leak (check whether the `used` curve increases monotonically without resetting).
3. Check the client process's memory usage.

**Scenario 3: One disk's read/write bandwidth is much lower than the others**

1. Open the **Per-Disk Read/Write/Total Bandwidth** panel.
2. Compare disks one by one using the `$disk_path` dropdown.
3. A disk with far lower bandwidth than the others may have degraded hardware (a slow disk).


**Scenario 4: Disk failure**

1. Open the **Disk Count & Faults** panel.
2. A `fault` curve > 0 indicates a faulty disk.
3. To locate the fault:
   - Use the `$rank` filter to identify the affected node.
   - Check the `memcache_client_ubs_io_disk_status` metric to identify the disk.
4. Log in to the node and inspect the corresponding block device under `/dev/`; replace the hardware if necessary.

## 6. Notes

1. **Build requirement:** Client metrics collection is enabled by default and requires no additional configuration.

2. **Metric freshness:**
   - The client collects and reports a metrics snapshot to Meta every 30 seconds.
   - Client metrics not updated on the Meta side for more than 90 seconds are automatically marked stale and cleared.
   - Dashboard values are aggregates from the latest reporting period, not instantaneous real-time values.
   - Per-disk bandwidth is sampled by the UBS IO library every 200 ms; the client retrieves the latest sample during collection.

3. **Failed reports:** Window counters are reset only after a successful RPC SyncCall. On failure, values accumulate into the next period, so data is not lost.

4. **Dashboard compatibility:** The dashboard JSON uses Grafana schema v39. Grafana 9.x or later is recommended.

