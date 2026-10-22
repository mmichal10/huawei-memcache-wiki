# MemCache Asynchronous Flush and Prefetch Features

## 1. Overview

In MemCache's three-tier cache architecture, DRAM-to-SSD flushing and SSD-to-DRAM rewarming are the two core paths for tiering data by temperature. This update adds **asynchronous flushing** and **prefetching** to these paths:

- **Asynchronous flush:** After data is written to DRAM, a background thread flushes it to SSD in batches. Watermarks control eviction only; they do not determine whether data is flushed. Upper-level `put` and `put_batch` requests are not affected by SSD write latency.
- **Prefetch:** When `is_exist` or `batch_is_exist` detects data that exists only on SSD, it triggers SSD-to-DRAM rewarming. After rewarming, later `get` or `get_batch` calls can achieve latency close to a native DRAM hit.

Both features require SSD pooling to be enabled (`ock.mmc.local_service.storage.enabled = true`). They are transparent to applications and require no changes to application calls.

## 2. How It Works

### 2.1 Asynchronous Flush

After data is written to DRAM, a background thread flushes it to SSD in batches. `put` and `put_batch` return without waiting for the SSD write to finish. After a successful flush, the SSD copy is registered in the cache metadata. Later `get` and `get_batch` calls read from DRAM or SSD according to the data's tier.


### 2.2 Prefetch

Prefetch is triggered by `is_exist` and `batch_is_exist`. If a key exists only on SSD, with no readable copy in DRAM or HBM, the system proactively rewarms it from SSD to DRAM.



For `batch_is_exist`, all keys found on SSD are submitted together to improve rewarming efficiency.

Prefetch failure does not affect correctness. Later `get` and `get_batch` calls can still retrieve the data through the passive rewarming path.

## 3. Configuration

### 3.1 `mmc-local.conf`

| Parameter | Type | Default | Description |
|------|------|--------|------|
| `ock.mmc.local_service.storage.enabled` | bool | `false` | Master switch for SSD caching. Asynchronous flush and prefetch work only when enabled. |

### 3.2 `mmc-meta.conf`

| Parameter | Type | Default | Description |
|------|------|--------|------|
| `ock.mmc.evict_threshold_high` | int | `90` | DRAM high watermark (%); reaching it triggers eviction. |
| `ock.mmc.evict_threshold_low` | int | `80` | Target watermark (%); eviction stops when this level is reached. |

### 3.3 Example Configuration

```ini
# mmc-local.conf
ock.mmc.local_service.storage.enabled = true

# mmc-meta.conf
ock.mmc.evict_threshold_high = 90
ock.mmc.evict_threshold_low = 80
```

## 4. Metrics and Observability

The MetaService `/metrics/summary` endpoint returns a one-line `key=value` summary. Fields related to these features include:

| Field | Description |
|------|------|
| `rewarm` | Number of data blocks successfully rewarmed from SSD to DRAM. |
| `flush_bytes` | Total bytes written to SSD by asynchronous flush. |
| `get_hit_dram` | Number of keys read directly from DRAM by `get` or `get_batch`. |
| `get_hit_ssd` | Number of keys returned after an SSD hit and successful rewarming by `get` or `get_batch`. |
| `dram_used` | DRAM used / total capacity. |
| `ssd_used` | SSD used / total capacity; `0/0` when no SSD is configured. |

**Verification:**

- Prefetch is working when `rewarm` keeps increasing and `get_hit_dram` rises as a share of `get_hit_dram + get_hit_ssd`.
- Flush is working when `flush_bytes` keeps increasing.

## 5. Example Usage

The recommended approach is to enable SSD pooling in `mmc-local.conf`, configure application memory in Python, and then read and write as usual:

```ini
# mmc-local.conf
ock.mmc.local_service.storage.enabled = true
```

```python
from memcache_hybrid import DistributedObjectStore, LocalConfig

config = LocalConfig()
config.dram_size = "400GB"
store = DistributedObjectStore()
store.setup(config)

# Batch reads and writes. Data is flushed to SSD asynchronously after being written to DRAM; is_exist / batch_is_exist trigger prefetch automatically.
keys = ["key1", "key2", "key3"]
values = [b"data" * 1024 for _ in keys]
store.put_batch(keys, values)
data = store.get_batch(keys)
```

## 6. Constraints and Limitations

- Asynchronous flush and prefetch require `ock.mmc.local_service.storage.enabled = true`; they have no effect when SSD is disabled.
- Prefetch is triggered only by `is_exist` and `batch_is_exist`. `get` and `get_batch` do not trigger prefetch; they use passive rewarming.
- Prefetch failure does not affect correctness. Later `get` and `get_batch` calls can still retrieve data through passive rewarming.

## 7. FAQ

**Q1: How can I tell whether prefetch is working?**

Check whether `rewarm` keeps increasing in `/metrics/summary` and whether the share of `get_hit_dram` is rising.

**Q2: What should I do if flush latency is high?**

Adjust flush-related parameters to flush more frequently and in larger batches. Also confirm that SSD large-I/O random-read bandwidth meets the requirement (at least 7 GB/s is recommended).

**Q3: Why do `get` requests occasionally time out?**

Configure the rewarming wait timeout with `ock.mmc.storage.pending_wait.timeout` in `mmc-meta.conf`; the unit is milliseconds. Set it according to SSD read latency: increase it for slower SSDs, or decrease it for latency-sensitive workloads. After a timeout, the read falls back to SSD.

**Q4: Can prefetch fill all of DRAM?**

Before prefetch runs, the system checks available DRAM and triggers eviction if more space is needed. However, if prefetch continuously runs faster than eviction, the watermark can keep rising. Set `evict_threshold_high` appropriately and ensure the space between the high watermark and 100% can hold the total NPU HBM KV cache capacity of one node.
