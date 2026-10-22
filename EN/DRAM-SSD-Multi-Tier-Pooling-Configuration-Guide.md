# UBSIO Config

| Configuration                                                                                              | Type | Required | Default | Valid range | Description |
| ------------------------------------------------------------------------------------------------------- | --- | ---- | ------- | ------------------------------------- | -------------------------------------------------------------------------- |
| [`ubsio.disk.path`](#ubsio.disk.path) | String | Yes | `empty` | Up to 16 valid block-device paths in standalone mode, separated by colons | Device paths used by UBS IO as a local SSD cache; whole block devices, partitions, and loop devices are supported. Leave empty to use memory cache only. |
| [`ubsio.mem.size_in_gb`](#ubsio.mem.size_in_gb) | Integer | Yes | `50` | `0` to `3072`; must not exceed available system memory | Memory-pool capacity available to one UBSIO process, in GB. |
| [`ubsio.standalone.device_count`](#ubsio.standalone.device_count) | Integer | Yes | `1` | With disks: `1` to `16`; diskless: `0` to `16` | Number of UBS IO processes sharing this node's cache disks. It should match the number of local_service processes with storage enabled; cannot be `0` when disks are configured. |
| `ubsio.standalone.force_new_disk` | Boolean | Yes | `false` | `true`, `false` | Whether to force-initialize the SSD as a new cache disk. `false` attempts to load compatible existing disk metadata; `true` rebuilds the cache layout and invalidates the old cache. Not applicable in diskless mode. |
| [`ubsio.wcache.evict_water_level`](#ubsio.wcache.evict_water_level) | Integer | Yes | `0` | `0` to `100` | L2.5 write-cache memory eviction watermark. In diskless mode without UnderFS, a configured value of `0` is treated as `90` at runtime. |
| `ubsio.wcache.disk_evict_water_level` | Integer | No | `90` | `0` to `100` | Write-cache disk eviction watermark, as a percentage. |
| `ubsio.bdm.io_engine` | String | No | `sync` | `sync`, `io_uring` | BDM I/O engine. |
| `ubsio.bdm.io_uring.sqpoll_mode` | String | No | `auto` | `auto`, `required`, `disabled` | io_uring SQPOLL mode; `auto` falls back to regular io_uring when unsupported. |
| `ubsio.bdm.sync.worker_num` | Integer | No | `16` | `1` to `64` | Number of internal threads used by the sync engine for BDM batched blocking I/O. |
| `ubsio.bdm.batch_read.window_keys` | Integer | No | `128` | `1` to `1024` | Maximum number of keys per window when BatchGet reads from disk through BDM. |
| `ubsio.bdm.batch_read.window_bytes_mb` | Integer | No | `64` | `1` to `1024` | Maximum bytes per window when BatchGet reads from disk through BDM, in MB. |
| `ubsio.batch_read.pipeline_depth` | Integer | No | `4` | `1` to `64` | Number of windows allowed in flight while BatchGet reads from disk through BDM; also controls the number of concurrent UnderFS batch-read requests. |
| `ubsio.bdm.batch_read.temp_pool_mb` | Integer | No | `0` | `0` to `65535` | Size of the dedicated temporary buffer pool for BDM batch reads, in MB; `0` allocates on demand. |
| `ubsio.batch_read.standalone.use_scratch_pool` | Boolean | No | `true` | `true`, `false` | Whether standalone BatchGet first reads SSD or UnderFS data into a scratch buffer and then copies it to the upper-level buffer; required for A3. |
| `ubsio.log.level` | String | No | `info` | `error`, `warn`, `info`, `debug`, `trace` | Log output level. |
| `ubsio.segment.size_in_mb` | Integer | No | `4` | `1` to `16` | Cache segment size. Local memory-pool blocks, BDM chunks, and SDK data-message blocks all use this size, in MB. |
| `ubsio.sdkmem.size_in_mb` | Integer | No | `0` | `0` to `4194304` | SDK data-message memory-pool size, in MB; `0` means no preallocation. |
| `ubsio.net.tls.enable.switch` | Boolean | No | `false` | `true`, `false` | TLS network-security switch. Usually left `false` for single-machine local deployments. |
| `ubsio.trace.enable` | Boolean | No | `true` | `true` or `false` | Enables or disables HTrace collection. Single-machine mode initializes the tracer module. |
| `ubsio.cache.qos.enable` | Boolean | No | `false` | `true`, `false` | Cache QoS and overload-control switch. |
| `ubsio.work.io.timeout` | Integer | No | `60` | `60` to `300` | I/O timeout passed to the SDK through the single-machine runtime config, in seconds. |
| `ubsio.batchget.thread.num` | Integer | No | `32` | `8` to `512` | Number of threads in the MirrorServer BatchGet executor outside standalone mode; unused in standalone mode. |
| `ubsio.cli_tools.enable` | Boolean | No | `false` | `true` or `false` | Enables CLI agent diagnostics. Set to `true` when using `cli_client --attach` to connect to `bio_console`. |

# Detailed Configuration

## ubsio.disk.path

### Meaning and Examples

`ubsio.disk.path` specifies device paths UBS IO uses as a local SSD cache. Whole block devices, existing partitions, and loop devices are supported. Create loop devices on the host and map them into the container; separate multiple paths with colons (`:`), up to **16** paths. <span style="color:red">**UBS IO must have exclusive access to configured devices, and they must not have mount points.**</span> **An empty value enables diskless mode, which uses only the local memory pool configured by `ubsio.mem.size_in_gb`; cached data is not persistent across restarts.**

```ini
# Configure a whole disk, partition, and loop device together
ubsio.disk.path = /dev/nvmexn1:/dev/nvmexn2p1:/dev/loopX

# Diskless mode
ubsio.disk.path =
```
---

## ubsio.standalone.device_count

`ubsio.standalone.device_count` is the number of devices participating in local cache-path allocation. **For brevity, this document refers to this setting as `device_count`. Its value should match the number of local_service processes with DRAM enabled; “DRAM enabled” means `ock.mmc.local_service.dram.size` is nonzero.**

- `0`: Preserves the legacy behavior. Each deviceId selects a device by its index in `ubsio.disk.path`; for example, `deviceId = 1` selects the second path.
- Greater than `0`: Configure the target whole disk, existing partition, or loop device in `ubsio.disk.path`.

Examples:

- With `4` local_service processes that have DRAM enabled and two target disks:

```ini
ubsio.disk.path = /dev/nvmexn1:/dev/nvmeyn1
ubsio.standalone.device_count = 4
```

- With `1` local_service process that has DRAM enabled and one target disk:

```ini
ubsio.disk.path = /dev/nvmexn1
ubsio.standalone.device_count = 1
```
---

## ubsio.mem.size_in_gb

`ubsio.mem.size_in_gb` is the local memory-pool capacity available to one UBSIO process, in GB. Calculate it from the MemCache deployment model, the node memory remaining for UBSIO, and the number of local_service processes with DRAM enabled.

Here, “DRAM enabled” means `ock.mmc.local_service.dram.size` is greater than `0`. “Remaining memory” is node memory available to UBSIO after accounting for system reservations, inference processes, the MemCache DRAM pool, and other uses. Calculate the value as follows, rounding down:

```text
ubsio.mem.size_in_gb = min(3072, floor(node memory remaining for UBS IO (GB) / number of local_service processes with DRAM enabled))
```

<span style="color:red">**`ubsio.mem.size_in_gb` must be at least `5GB`. If less than `5GB` can be allocated to a UBSIO process, free node memory or reduce the number of local_service processes with DRAM enabled before configuring this setting.**</span>

For example, if `200GB` of node memory remains for UBSIO and there are `4` local_service processes with `ock.mmc.local_service.dram.size > 0`, the per-process configuration limit is `50GB`. For SSD deployments, an explicit setting of `10GB` is recommended:

```ini
# floor(200 / 4) = 50; the per-process configuration limit is 50GB
ubsio.mem.size_in_gb = 10
```

For a disaggregated MemCache deployment, `50` GB per process is recommended; for other deployments, `10` GB is recommended. Consider increasing this value only when L2.5 is needed. Do not exceed the calculated limit above, and adjust [`ubsio.wcache.evict_water_level`](#ubsiowcacheevict_water_level) as well.


---

## ubsio.wcache.evict_water_level

`ubsio.wcache.evict_water_level` is the memory eviction watermark for the L2.5 write cache. Its range is `0` to `100`; **it specifies the percentage of L2.5 write-cache memory capacity in use at which eviction begins**.

### Tier Definitions

- **L2.5**: UBS IO's in-memory write-cache layer, between MemCache DRAM (L2) and local SSD (L3), which caches UBS IO data in memory.
- **L3**: UBS IO's local SSD cache layer, whose capacity is provided by the local SSDs or partitions specified by `ubsio.disk.path`.

### Configuration Recommendations

#### L2.5 Scenario

Use L2.5 only, without L3. Data remains in the UBS IO memory write cache and is discarded when it reaches the eviction watermark. A high watermark such as `85` is recommended:

```ini
ubsio.disk.path =
ubsio.wcache.evict_water_level = 85
```

#### L3 Scenario

Use L3 as the primary cache layer and L2.5 only as a temporary layer during writes. Set the watermark to `0` so data written to L2.5 is evicted to L3 as soon as possible:

```ini
ubsio.disk.path = /dev/nvmexn1p1:/dev/nvmexn1p2:/dev/nvmexn1p3:/dev/nvmexn1p4
ubsio.wcache.evict_water_level = 0
```

#### L2.5 + L3 Scenario

Use L2.5 and L3 together. Set the watermark above `0` so data is preferentially retained in L2.5 and evicted to L3 after reaching the watermark. For example:

```ini
ubsio.disk.path = /dev/nvmexn1p1:/dev/nvmexn1p2:/dev/nvmexn1p3:/dev/nvmexn1p4
ubsio.wcache.evict_water_level = 85
```

For L2.5 + L3, adjust the watermark according to available memory, SSD capacity, and the desired hot-to-cold data ratio. A higher watermark retains more data in L2.5.

---

## ubsio.bdm.batch_read.standalone.use_scratch_pool

`ubsio.bdm.batch_read.standalone.use_scratch_pool` controls whether standalone BatchGet reads data into a scratch pool before copying it to the upper-level buffer when reading from disk through BDM. When set to `false`, BDM writes directly to the upper-level buffer.

The scratch pool is required for A3, which does not support direct BDM writes to the upper-level buffer. For other scenarios, disabling the scratch pool is recommended so BDM writes directly to the upper-level buffer, avoiding one memory copy:

```ini
ubsio.bdm.batch_read.standalone.use_scratch_pool = false
```

---

# Other

## Device Configuration

Set `ubsio.disk.path` to the path of the target whole disk, existing partition, or loop device.

### Scenario 1: DRAM + an Available Local SSD (Recommended)

If a node has one or more NVMe drives, configure the target drive directly. An existing partition can also be configured, but UBS IO must have exclusive access to the device and it must not have a mount point.

#### 1. Check Device Status

Use the following commands to check the target device path, type, and mount status:

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS

# Check whether the target device has a mount point; /dev/nvme0n1 is used as an example
lsblk /dev/nvme0n1
findmnt /dev/nvme0n1
```

<span style="color:red">**⚠️ Verify the target device path carefully before configuration. UBS IO must have exclusive access to the device, which must have no mount point or existing data that needs to be preserved.**</span>

#### 2. Typical Scenario

With `4` local_service processes that have DRAM enabled and two target disks, configure:

```ini
ubsio.disk.path = /dev/nvme0n1:/dev/nvme1n1
ubsio.standalone.device_count = 4
```

When using an existing partition, configure its path:

```ini
ubsio.disk.path = /dev/nvme2n1p1
ubsio.standalone.device_count = 4
```

---

### Scenario 2: DRAM + No Free Local SSD + Available Space on an SSD (Loop Device)

When no physical disk is available, use `fallocate` and `losetup --find --show` to create a loop block device that simulates a disk. The system automatically selects an unused loop device and enables direct I/O. After creating it, set `ubsio.disk.path` to the loop-device path.

> <span style="color:red">**⚠️ Before creating a loop device, check which loop devices are in use. Do not manually reuse an occupied `/dev/loopN`, as this may cause data loss.**</span>

```bash
# Create and preallocate a 1 TB image file; adjust the size as needed
sudo fallocate -l 1T /data/ubsio_disk.img

# Create a loop device and print the path assigned by the system
LOOP_DEV=$(sudo losetup --find --show --direct-io=on /data/ubsio_disk.img)
echo "${LOOP_DEV}"
# Example output: /dev/loop2
```

Write the actual loop-device path printed above to `mmc-local.conf`:

```ini
ubsio.disk.path = /dev/loop2
ubsio.standalone.device_count = 4
```
---
### Scenario 3: DRAM + Remote Storage (Loop Device)
For a DRAM deployment with a remote network filesystem mount (such as A800), create a file under the remote storage mount point and configure it as a loop block device to use remote capacity for kvcache data. For high performance, remote-storage network bandwidth of at least 200GE is recommended.
> <span style="color:red"><strong>⚠️ Before creating a loop device, check which loop devices are in use. Do not manually reuse an occupied <code>/dev/loopN</code>, as this may cause data loss.</strong></span><br>
> <span style="color:red"><strong>⚠️ If multiple A2/A3/A5 machines share one A800, create a separate file for each machine and configure each as that machine's loop device. The machines must not use the same file on remote storage.</strong></span>

Assume A800 is mounted at `/data/A800/` on two compute nodes, A and B. Create two 1 TB files and configure loop devices as follows:
```bash
# Create and preallocate 1 TB image files; adjust the size as needed
truncate -s 1T /data/A800/disk0.img
truncate -s 1T /data/A800/disk1.img

# Create a loop device and print the path assigned by the system; run on node A.
LOOP_DEV=$(sudo losetup --find --show --direct-io=on /data/A800/disk0.img)
echo "${LOOP_DEV}"
# Example output: /dev/loop2

# Create a loop device and print the path assigned by the system; run on node B.
LOOP_DEV=$(sudo losetup --find --show --direct-io=on /data/A800/disk1.img)
echo "${LOOP_DEV}"
# Example output: /dev/loop3
```
```
# Set the actual loop-device path in node A's `mmc-local.conf`.
ubsio.disk.path = /dev/loop2

# Set the actual loop-device path in node B's `mmc-local.conf`.
ubsio.disk.path = /dev/loop3
```
---

### Detach a Loop Device

> <span style="color:red">**⚠️ Proceed only after confirming the loop disk is no longer needed and related services have stopped. Cached data will be lost after detaching it.**</span>

```bash
# 1. List loop devices and their associated files
losetup -a

# 2. Detach the loop device, substituting its actual path
sudo losetup -d /dev/loop2

# 3. Delete the image file (optional)
sudo rm -f /data/ubsio_disk.img
```


### Notes

| Item | Description |
| --- | --- |
| Exclusive access | UBS IO must have exclusive access to configured whole disks, partitions, or loop devices; do not share them with other services. |
| Mount restriction | Configured devices must not have mount points. |
| Path limit | `ubsio.disk.path` accepts at most 16 paths. |
| Hot-add disks | Standalone mode does not support adding disks at runtime. |
| Device reuse | A BDM disk head is written for each deviceId; recovery is rejected if they do not match. |
| Permissions | The process needs read and write access to block devices. Running as root is recommended. |
| Disk specifications | The combined random-read bandwidth of all configured local SSDs should be at least 7GB/s; remote-storage network bandwidth should be at least 200GE. See the [disk bandwidth benchmark guide](https://gitcode.com/zhangjialee/ubs-io/wiki/%E7%A3%81%E7%9B%98%E5%B8%A6%E5%AE%BD%E5%8E%8B%E6%B5%8B%E6%8C%87%E5%8D%97.md) for benchmark instructions. |
