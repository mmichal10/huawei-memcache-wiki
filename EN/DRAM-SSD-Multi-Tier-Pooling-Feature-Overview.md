# MemCache SSD Caching Overview
## 1. Background and Motivation

MemCache is a high-performance distributed KV cache engine for LLM inference, built on the MemFabric pooled-memory architecture. In long-context inference, a single request can produce tens of gigabytes of KV cache, making an HBM + DRAM two-tier cache very expensive.

SSD caching adds a third storage tier to MemCache. It integrates with the external UBS IO key-value storage engine and uses the high capacity and low cost of local NVMe drives to automatically move infrequently accessed, cold data to SSD, keeping only hot data in memory. This provides a near all-memory cache experience at a much lower cost per unit of capacity.


**Use cases:**

- **Long-context LLM inference:** A single request can have tens of gigabytes of KV cache, and total cache demand can exceed memory capacity. SSD can hold large amounts of cold context data.
- **Multi-turn conversations:** As KV cache from earlier turns is accessed less frequently, it can move to SSD and free memory for new requests.
- **Batch or offline inference:** Suitable when latency is not extremely sensitive and throughput and cache capacity are the priorities.
- **Cost-sensitive deployments:** Expanding cache capacity with NVMe SSDs is more cost-effective than adding more memory.

## 2. Three-Tier Cache Architecture

The system organizes the cache into three tiers by performance and capacity, transparently to users:

```
┌──────────────┐  Medium             Capacity      Key advantage
│   HBM (L1)   │  NPU high-bandwidth memory  ~tens of GB  Extreme bandwidth, near-memory compute
├──────────────┤
│  DRAM (L2)   │  Host DDR memory     ~hundreds of GB  Balance of capacity and bandwidth
├──────────────┤
│   SSD (L3)   │  Local NVMe drive   ~several TB  Low-cost, high-capacity persistence
└──────────────┘
```

Users do not need to know which tier holds the data when calling the `Put` or `Get` APIs.

## 3. Deployment

### 3.1 Installation

```bash
# Install memfabric_hybrid
pip install memfabric_hybrid==1.2.0

# Install memcache_hybrid (UBSIO is integrated into MemCache and does not need a separate installation)
pip install memcache_hybrid==1.2.0
```

The MemCache installation path depends on the Python version, virtual environment, and installation method. Use the following command to locate the configuration directory:

```bash
MMC_CONFIG_DIR="$(python3 -c 'from pathlib import Path; import memcache_hybrid; print(Path(memcache_hybrid.__file__).resolve().parent / "config")')"
echo "$MMC_CONFIG_DIR"
```

The configuration file paths are:

| Item | Path |
| ------------------------- | --------------------------------- |
| Configuration file (`mmc-meta.conf`) | `$MMC_CONFIG_DIR/mmc-meta.conf` |
| Configuration file (`mmc-local.conf`) | `$MMC_CONFIG_DIR/mmc-local.conf` |

### 3.2 Edit the Configuration Files

#### `mmc-local.conf`

```ini
# Enable SSD pooling
ock.mmc.local_service.storage.enabled = true

# Block device paths for UBS IO local SSD caching. Whole disks, partitions, and loop devices are supported. Separate multiple paths with colons. UBS IO must have exclusive use of these devices, which must not be mounted.
ubsio.disk.path = /dev/nvmexn1:/dev/nvmexn2p1:/dev/loopX

# UBS IO memory-pool size per process, in GB. Typical setting: 80 GB for A3 separated deployment, 10 GB otherwise.
ubsio.mem.size_in_gb = 10

# Number of MemCache LocalService processes with SSD pooling enabled in one vLLM startup. Use 1 for A3 separated deployment; otherwise use the number of vLLM processes.
ubsio.standalone.device_count = 8

# Force the configured SSD devices to be initialized as new disks
ubsio.standalone.force_new_disk = true
```

| Parameter | Meaning | Default | Description |
| --------------------------------------- | -------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ock.mmc.local_service.storage.enabled` | Enable SSD caching | `false` | Disabled by default. Set to `true` to enable SSD multi-tier pooling. |
| `ubsio.disk.path` | UBS IO local SSD cache device paths | `empty` | Specify devices for UBS IO local SSD caching. Whole disks, partitions, and loop devices are supported; separate multiple paths with colons. <span style="color:red">**UBS IO must have exclusive use of the configured devices, and they must not have mount points.**</span> |
| `ubsio.mem.size_in_gb` | UBS IO memory pool size per process | `50` | Integer in the range `0`–`3072`. Configure at least `5` GB for SSD use; `10` GB is recommended explicitly. Total UBS IO memory usage on a node equals the per-process pool size multiplied by the number of processes with SSD pooling enabled. It must be **less than** the node memory remaining after OS-reserved memory, active inference-service memory, and MemCache DRAM are deducted. Typical values: 80 GB for A3 disaggregated deployment and 10 GB otherwise. |
| `ubsio.standalone.device_count` | Number of UBS IO processes | `0` | Number of MemCache LocalService processes with SSD pooling enabled. In A3 disaggregated deployment, only the standalone process enables SSD pooling, so set this to 1. Otherwise, all vLLM processes should enable it, so set this to the number of vLLM processes. |
| `ubsio.standalone.force_new_disk` | Force initialization as new disks | `false` | When set to `true`, UBS IO initializes configured SSD devices as new disks instead of restoring existing metadata. The current version does not support recovery, so this must be set to `true`. |

`ubsio.mem.size_in_gb` is the size of the UBS IO memory pool created by each `local_service` process with SSD pooling enabled. Calculate the per-process configuration limit as follows:

```text
Per-process limit = min(3072, floor(node memory available to UBS IO (GB) / number of local_service processes with DRAM enabled))
```

For example, if `200` GB remains available to UBS IO on a node and four `local_service` processes have SSD pooling enabled, the per-process limit is `50` GB, so the recommended `10` GB is within range. If the calculated limit is below `5` GB, free more memory or reduce the process count.

For disaggregated MemCache deployments, `50` GB per process is recommended; `10` GB is recommended for other scenarios. To use L2.5 capabilities, increase this value as needed and also adjust [`ubsio.wcache.evict_water_level`](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md#ubsiowcacheevict_water_level).

For detailed UBS IO configuration options and parameters, see the [DRAM-SSD-Multi-Tier-Pooling-Configuration-Guide](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md).

#### `mmc-meta.conf` configuration file (latest version 1.2.1 example):

```ini
# Put operations trigger data eviction when space utilization reaches the high watermark.
ock.mmc.evict_threshold_high = 70

# Target space utilization after eviction completes, i.e. the low watermark.
ock.mmc.evict_threshold_low = 60
```

| Parameter | Meaning | Default | Description |
| ------------------------------- | -------------------------- | -------- | ----------------------------------------------------------------------------------------------------- |
| `ock.mmc.evict_threshold_high` | Space-utilization high watermark that triggers eviction | `90` | Integer from `1` to `99`. A `Put` triggers eviction when utilization reaches this watermark. Eviction cannot be triggered if a single `Put` exceeds 1% of capacity. |
| `ock.mmc.evict_threshold_low` | Target space utilization after eviction | `80` | Integer from `1` to `98`. Eviction stops once utilization reaches this watermark. |

> MemCache only controls whether SSD is enabled and its warm-up policy. SSD capacity, eviction watermarks, and related settings are controlled by UBS IO parameters.


#### Usage

**Deployment examples:**

A2: [MMC Best Practice — GLM-5.1+A2 8-node PD disaggregation](https://gitcode.com/Ascend/memcache/wiki/MMC%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E2%80%94GLM-5.1+A2-8%E6%9C%BAPD%E5%88%86%E7%A6%BB.md)

A3: [MemCache+vLLM+A3 disaggregated deployment example](https://gitcode.com/Ascend/memcache/wiki/MemCache+vLLM+A3%E5%88%86%E7%A6%BB%E9%83%A8%E7%BD%B2%E6%A1%88%E4%BE%8B.md)

#### Uninstall

```bash
pip uninstall memcache_hybrid
```


## 4. Limitations and Notes

- SSD caching depends on local NVMe drives; ensure an SSD device is available on the node.
- SSD latency (milliseconds) is significantly higher than memory. For real-time inference with strict latency requirements, expand DRAM first.
- Enabling SSD caching requires the ubs-io runtime library to be installed on the system.
- SSD random-read bandwidth for large IO should be at least 7 GB/s. See the [disk bandwidth benchmark guide](https://gitcode.com/zhangjialee/ubs-io/wiki/%E7%A3%81%E7%9B%98%E5%B8%A6%E5%AE%BD%E5%8E%8B%E6%B5%8B%E6%8C%87%E5%8D%97.md) for testing instructions.

## 5. FAQ
#### 1. How do I handle UBS IO errors during startup initialization?
UBS IO logs are located at `/var/log/ubsio/bio${PID}.log` by default. Common issues include: <br>
1. The `/dev/loop` device is missing or the logs show a permission error: <br>
vLLM usually runs in a container. Create the `/dev/loop` device on the host, then restart or recreate the container so it can detect the device. <br>
2. An incorrect `device_count` causes a timeout during the gather communication stage: <br>
In A3 disaggregated MemCache deployment, where MemCache and vLLM start separately, set `device_count` to 1. In other scenarios, set it to the number of vLLM processes.
[MemCache+vLLM+A3 disaggregated deployment example](https://gitcode.com/Ascend/memcache/wiki/MemCache+vLLM+A3%E5%88%86%E7%A6%BB%E9%83%A8%E7%BD%B2%E6%A1%88%E4%BE%8B.md)

#### 2. How can a host loop block device be mapped into a container?
##### With Docker:
1. For a privileged container (created with `--privileged=true`), create the loop device on the host and restart the container: `docker restart <container-name>`. <br>
2. For an unprivileged container, recreate the container and add the device when creating it: `--device=/dev/loop0:/dev/loop1:rwm`.
##### With nerdctl:
Unlike Docker, nerdctl cannot update the OCI spec on restart, so the container must be recreated. Add the `--device` argument when creating the new container, for example: <br>
```
nerdctl -n default run -d \
  --name <container> \
  --privileged \
  --device /dev/loop0:/dev/loop1:rwm
```

#### 3. Multi-tier pooling increases memory pressure on MemCache. How should memory watermarks be set?
After enabling multi-tier pooling, reserve enough space in MemCache DRAM to absorb concentrated writes from the local NPU HBM KV Cache. The remaining space between `ock.mmc.evict_threshold_high` and 100% should be sufficient to hold the total physical local NPU HBM KV Cache capacity on a single A2/A3/A5 node. Calculate it as follows: <br>
```
M × (100 - H) / 100 ≥ K 
M: Total MemCache DRAM capacity on one node;
K: Total physical NPU HBM KV Cache capacity on one node;
H: `ock.mmc.evict_threshold_high`, expressed as a percentage.
```
For example, if the NPU KV cache pool is 120 GB and total MemCache memory per node is 400 GB, set the high watermark to 70% and the low watermark to 50%. <br>

How can I find the local NPU HBM KV Cache capacity? <br>
It is usually printed at vLLM-Ascend startup. This value is the HBM KV Cache capacity of a single NPU rank (one card): <br>
`(Worker_TP0 ...) Available KV cache memory: 26.22 GiB`<br>
The NPU HBM KV Cache pool capacity per node is: <br>
`number of cards per node × HBM KV Cache capacity per card`

#### 4. How can I tell whether a process started with SSD pooling enabled?

Search the process startup logs for the keyword `boostio`:

```bash
grep -i "boostio" <process-log-file>
```

If matching `boostio` logs appear, SSD pooling is enabled for that process. If no matching logs appear, SSD pooling was not enabled.
