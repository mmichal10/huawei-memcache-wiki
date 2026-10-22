# Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a distributed KV Cache system for AI inference, optimized for Ascend supernodes and servers. It can serve as the KV Pool backend for inference frameworks such as vLLM Ascend and enables cross-request KV Cache storage and reuse for long texts and workloads with high prefix reuse.

This guide covers a single-instance PD-disaggregated deployment on four 910C (Atlas 800I A3) nodes, using GLM-5.2-W4A8 and vLLM Ascend 0.23.0 RC1. The 200K high-performance service and performance tests use two Prefill nodes and two Decode nodes. Both test groups enable MMC and compare MMC memory pooling alone with MMC + SSD pooling.

The MMC DRAM cache pool in these tests is 300 GB per node, 1.2 TB across four nodes. The SSD pooling scheme adds 3.5 TB of SSD capacity per node, 14 TB across four nodes.

# Deployment Preparation

## Prerequisites

| Item | Requirement or configuration used here |
| --- | --- |
| Hardware | Atlas 800I A3 (910C) × 4 |
| Deployment topology | Single-instance PD disaggregation: 2 Prefill nodes + 2 Decode nodes |
| Driver firmware | >= 25.5.1 |
| L1 1520 CC | >= 1.5.0 |
| OS kernel version | >= 5.10 |
| Model | GLM-5.2-W4A8 |
| Inference framework | vLLM Ascend 0.23.0 RC1 |
| MMC DRAM capacity | Reserve 300 GB per node, 1,200 GB total across four nodes (about 1.2 TB) |
| SSD capacity | Required only for SSD pooling; provide 3.5 TB of exclusively available capacity per node, 14 TB across four nodes |

> Whole disks, partitions, or loop block devices used for SSD pooling must be dedicated exclusively to UBS IO. Before configuration, confirm that the devices have no mount points and contain no data that must be preserved. Use the device paths verified on each node.

# Performance Tests

This section compares two cache configurations for GLM-5.2-W4A8 at a 200K sequence length on a single PD-disaggregated instance across four 910C nodes. MMC is enabled in both test groups: one tests MMC memory pooling alone, the other MMC + SSD pooling. Each group tests concurrency levels of 100, 150, 200, 250, and 300 in sequence.

## Test Configuration and Capacity Basis

| Configuration | Value |
| --- | --- |
| Hardware | 910C × 4 |
| Deployment mode | Four-node PD disaggregation, single instance |
| Model | GLM-5.2-W4A8 |
| Software version | vLLM Ascend 0.23.0 RC1 |
| Sequence length | 200K |
| Concurrency | 100, 150, 200, 250, 300 |
| MMC | Enabled in both test groups |
| MMC cache pool | 300 GB per node, 1,200 GB total across four nodes (about 1.2 TB) |
| SSD pooled cache | 3.5 TB per node, 14 TB total across four nodes; enabled only for the SSD pooling scheme |

## Overview: Results at Matching Concurrency

The MMC + SSD pooling scheme has a higher success rate than MMC memory pooling alone at all five concurrency levels, with the gap generally widening as concurrency increases. At concurrency 300, the success rate rises from 84.83% to 93.27%, and failed requests fall from 455 to 202. Average response time and average TTFT are also lower than with MMC memory pooling alone at the same concurrency.

### Success Rate, Average Response Time, and Throughput

| Concurrency | MMC memory pooling success rate (%) | MMC + SSD pooling success rate (%) | Success-rate increase (%) | MMC memory pooling average response time (s) | MMC + SSD pooling average response time (s) | Average response-time improvement (%) | MMC memory pooling output throughput (token/s) | MMC + SSD pooling output throughput (token/s) | Output throughput increase (%) |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 100 | 90.60 | 93.00 | 2.65 | 69.342 | 49.216 | 29.02 | 177.07 | 230.93 | 30.42 |
| 150 | 90.27 | 94.20 | 4.35 | 78.109 | 52.416 | 32.89 | 279.23 | 356.16 | 27.55 |
| 200 | 89.60 | 95.05 | 6.08 | 85.145 | 55.844 | 34.41 | 340.49 | 445.92 | 30.96 |
| 250 | 88.92 | 94.44 | 6.21 | 85.299 | 69.103 | 18.99 | 356.96 | 544.44 | 52.52 |
| 300 | 84.83 | 93.27 | 9.95 | 97.068 | 79.558 | 18.04 | 424.11 | 651.92 | 53.71 |

### TTFT

| Concurrency | MMC memory pooling Avg TTFT (s) | MMC + SSD pooling Avg TTFT (s) | Avg TTFT improvement (%) | MMC memory pooling P90 TTFT (s) | MMC + SSD pooling P90 TTFT (s) | P90 TTFT improvement (%) | MMC memory pooling P95 TTFT (s) | MMC + SSD pooling P95 TTFT (s) | P95 TTFT improvement (%) |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 100 | 45.969 | 30.904 | 32.77 | 127.781 | 60.862 | 52.37 | 175.651 | 99.345 | 43.44 |
| 150 | 54.736 | 37.742 | 31.05 | 139.828 | 69.112 | 50.57 | 208.798 | 116.691 | 44.11 |
| 200 | 60.599 | 43.543 | 28.15 | 159.877 | 83.825 | 47.57 | 217.639 | 120.340 | 44.71 |
| 250 | 58.840 | 55.826 | 5.12 | 145.336 | 110.966 | 23.65 | 199.130 | 143.664 | 27.85 |
| 300 | 67.138 | 63.895 | 4.83 | 164.029 | 126.159 | 23.09 | 211.876 | 161.919 | 23.58 |

## MMC Memory Pooling Only: Test Results

This group enables MMC memory pooling only and does not enable SSD pooled caching. The MMC cache pool is 300 GB per node, 1.2 TB total across four nodes.

| Metric | Concurrency 100 | Concurrency 150 | Concurrency 200 | Concurrency 250 | Concurrency 300 |
| --- | ---: | ---: | ---: | ---: | ---: |
| Test time | `20260812_224311` | `20260812_234333` | `20260813_004355` | `20260813_014417` | `20260813_024438` |
| Total requests | 1,000 | 1,500 | 2,000 | 2,500 | 3,000 |
| Successful requests | 906 | 1,354 | 1,792 | 2,223 | 2,545 |
| Failed requests | 94 | 146 | 208 | 277 | 455 |
| Success rate (%) | 90.60 | 90.27 | 89.60 | 88.92 | 84.83 |
| Total duration (s) | 3,009.51 | 3,009.72 | 3,010.19 | 3,010.22 | 3,009.47 |
| Concurrency | 100 | 150 | 200 | 250 | 300 |
| QPS (requests/s) | 0.33 | 0.50 | 0.66 | 0.83 | 1.00 |
| QPM (requests/min) | 19.94 | 29.90 | 39.86 | 49.83 | 59.81 |
| Average response time (s) | 69.342 | 78.109 | 85.145 | 85.299 | 97.068 |
| Maximum response time (s) | 301.035 | 300.988 | 300.994 | 301.003 | 300.987 |
| Minimum response time (s) | 2.544 | 4.146 | 4.640 | 4.463 | 1.696 |
| Total input tokens | 46,214,017 | 68,701,358 | 90,540,928 | 111,100,229 | 127,842,078 |
| Total output tokens | 532,901 | 840,392 | 1,024,940 | 1,074,518 | 1,276,356 |
| Average input tokens | 46,214.0 | 45,800.9 | 45,270.5 | 44,440.1 | 42,614.0 |
| Average output tokens | 588.2 | 620.7 | 572.0 | 483.4 | 501.5 |
| Average TTFT (s) | 45.969 | 54.736 | 60.599 | 58.840 | 67.138 |
| P50 TTFT (s) | 20.845 | 29.429 | 33.166 | 35.488 | 43.510 |
| P90 TTFT (s) | 127.781 | 139.828 | 159.877 | 145.336 | 164.029 |
| P95 TTFT (s) | 175.651 | 208.798 | 217.639 | 199.130 | 211.876 |
| Response time P50 (s) | 24.000 | 33.608 | 39.331 | 40.942 | 50.113 |
| Response time P90 (s) | 272.511 | 281.337 | 300.061 | 300.438 | 300.644 |
| Response time P95 (s) | 300.923 | 300.876 | 300.860 | 300.818 | 300.804 |
| Response time P99 (s) | 300.996 | 300.972 | 300.975 | 300.965 | 300.945 |

## MMC + SSD Pooling: Test Results

This group enables both MMC memory pooling and SSD pooled caching. The MMC cache pool remains 300 GB per node, 1.2 TB total across four nodes; SSD pooled caching provides 3.5 TB per node, 14 TB total.

| Metric | Concurrency 100 | Concurrency 150 | Concurrency 200 | Concurrency 250 | Concurrency 300 |
| --- | ---: | ---: | ---: | ---: | ---: |
| Test time | `20260813_113213` | `20260813_123234` | `20260813_133255` | `20260813_143317` | `20260813_153339` |
| Total requests | 1,000 | 1,500 | 2,000 | 2,500 | 3,000 |
| Successful requests | 930 | 1,413 | 1,901 | 2,361 | 2,798 |
| Failed requests | 70 | 87 | 99 | 139 | 202 |
| Success rate (%) | 93.00 | 94.20 | 95.05 | 94.44 | 93.27 |
| Total duration (s) | 3,009.87 | 3,009.56 | 3,009.54 | 3,010.22 | 3,010 |
| Concurrency | 100 | 150 | 200 | 250 | 300 |
| QPS (requests/s) | 0.33 | 0.50 | 0.66 | 0.83 | 1.00 |
| QPM (requests/min) | 19.93 | 29.90 | 39.87 | 49.83 | 59.80 |
| Average response time (s) | 49.216 | 52.416 | 55.844 | 69.103 | 79.558 |
| Maximum response time (s) | 301.074 | 301.006 | 301.082 | 300.996 | 301.003 |
| Minimum response time (s) | 3.776 | 3.596 | 3.457 | 3.196 | 4.381 |
| Total input tokens | 47,870,136 | 72,334,504 | 97,730,861 | 120,439,789 | 144,153,609 |
| Total output tokens | 695,070 | 1,071,887 | 1,342,005 | 1,638,895 | 1,962,265 |
| Average input tokens | 47,870.1 | 48,223.0 | 48,865.4 | 48,175.9 | 48,051.2 |
| Average output tokens | 747.4 | 758.6 | 705.9 | 694.2 | 701.3 |
| Average TTFT (s) | 30.904 | 37.742 | 43.543 | 55.826 | 63.895 |
| P50 TTFT (s) | 20.568 | 27.172 | 31.994 | 41.817 | 49.054 |
| P90 TTFT (s) | 60.862 | 69.112 | 83.825 | 110.966 | 126.159 |
| P95 TTFT (s) | 99.345 | 116.691 | 120.340 | 143.664 | 161.919 |
| Response time P50 (s) | 21.623 | 28.475 | 33.117 | 44.223 | 53.767 |
| Response time P90 (s) | 122.113 | 123.144 | 117.018 | 145.991 | 182.307 |
| Response time P95 (s) | 300.870 | 300.763 | 280.600 | 300.654 | 300.656 |
| Response time P99 (s) | 300.984 | 300.963 | 300.920 | 300.935 | 300.928 |

## Environment Setup

This guide uses GLM-5.2-W4A8. The following ModelScope repository and example deployment directory retain the artifact name `GLM-5.2-w4a8c8`:
[*https://modelscope.cn/models/Eco-Tech/GLM-5.2-w4a8c8*](https://modelscope.cn/models/Eco-Tech/GLM-5.2-w4a8c8)

Pull the vLLM Ascend 0.23.0 RC1 image:
``docker pull quay.io/ascend/vllm-ascend:v0.23.0rc1``
Save the following as `start-docker.sh`.

```
#!/bin/bash
IMAGES_ID="$1"
NAME="$2"

# Check the number of arguments (two are required: image ID and container name).
if [ $# -ne 2 ]; then
    echo "error: two arguments are required, format: $0 <image-id> <container-name>"
    exit 1
fi

# Check whether the image exists.
if ! docker images --format "{{.ID}}" | grep -q "^${IMAGES_ID:0:12}$"; then
    echo "error: image ID $IMAGES_ID does not exist"
    exit 1
fi

# Start the container (quote variables to avoid issues with special characters).
docker run --name "${NAME}" -it -d --net=host --shm-size=500g \
    --privileged=true \
    -w /home \
    --device=/dev/davinci_manager \
    --device=/dev/hisi_hdc \
    --device=/dev/devmm_svm \
    --entrypoint=bash \
    -v /usr/local/Ascend/driver:/usr/local/Ascend/driver \
    -v /usr/local/dcmi:/usr/local/dcmi \
    -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
    -v /etc/ascend_install.info:/etc/ascend_install.info \
    -v /usr/local/sbin:/usr/local/sbin \
    -v /etc/hccn.conf:/etc/hccn.conf \
    -v /home:/home \
    -v /mnt:/mnt \
    -v /tmp:/tmp \
    -v /data:/data \
    -v /usr/share/zoneinfo/Asia/Shanghai:/etc/localtime \
    -e http_proxy="$http_proxy" \
    -e https_proxy="$https_proxy" \
    "${IMAGES_ID}"
```

Run the following command to create the container.
`bash start-docker.sh 611962961cde glm5.2-memcache`
Run the following command to enter the created container.
`docker exec -it glm5.2-memcache bash`

# MemCache Configuration and Startup

## Install the Specified Package Versions

Before editing the MemCache configuration, install the specified package versions in the containers on all four nodes:

```bash
pip install memfabric_hybrid==1.2.0 --force-reinstall --no-deps
pip install memcache_hybrid==1.2.0 --force-reinstall --no-deps
```

`--force-reinstall` overwrites packages with the same name in the current environment. `--no-deps` does not install or modify dependencies automatically. After installation, read the `VERSION` file from the installed package in each of the four containers. Verify that the full commit matches the test baseline and is identical across all nodes:

```bash
python3 - <<'PY'
from importlib.util import find_spec
from pathlib import Path

for package in ("memfabric_hybrid", "memcache_hybrid"):
    spec = find_spec(package)
    version_file = Path(spec.origin).resolve().parent / "VERSION"
    print(f"[{package}]")
    print(version_file.read_text().strip())
PY
```

PyPI-published  `1.2.0` CPython 3.12 AArch64 wheel reference values are:`memfabric_hybrid` commit prefix `999034` and `memcache_hybrid` commit prefix `4310d6`.

## Edit the MemCache Configuration Files

**Step 1: Find the configuration file paths**

At startup, MetaService and LocalService load settings from separate configuration files. By default, these files are in the config directory of the MemCache wheel package.

```
# Find the Python path.
which python
/usr/local/python3.12.13/bin/python
The configuration file paths are:
/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-meta.conf
/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf
```

**Step 2: Edit the configuration files**
Assume MetaService will run on node P0 (IP denoted **P0_node_ip**). Make the following changes to `mmc-meta.conf` and `mmc-local.conf` on P0, then copy `mmc-local.conf` to the same location on P1, D0, and D1 (all four nodes must have identical configuration).

Edit the MetaService configuration file:
`vim /usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-meta.conf`
Make the following changes:

```
## Meta service start-up url
ock.mmc.meta_service_url = tcp://<P0_node_ip>:5000
## Set the IP to the IP of the node running MetaService.

## Config store url
ock.mmc.meta_service.config_store_url = tcp://<P0_node_ip>:6000
## Set the IP to the IP of the node running MetaService.

## HTTP service url
ock.mmc.meta_service.metrics_url = http://<P0_node_ip>:8000      
## Set the IP to the IP of the node running MetaService.

## Data eviction low watermark: target space utilization after eviction completes.
ock.mmc.evict_threshold_low = 50

## Data eviction high watermark: eviction is triggered when space utilization reaches this value.
ock.mmc.evict_threshold_high = 70

## DRAM pressure watermark: when reached, eviction quickly deletes data to free memory.
ock.mmc.rewarm.dram_watermark = 90

## Log level: debug, info, warn, error
ock.mmc.log_level = info
```

Note: Replace `127.0.0.1` in all three places above with the IP of the node running MetaService.
**Edit the LocalService configuration file:**
`vim /usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf`
Make the following changes:

```
## Meta service start-up url
## Keep consistent with the same name configuration in mmc-meta.conf
ock.mmc.meta_service_url = tcp://<P0_node_ip>:5000                      
## Set to the IP of the node running MetaService.

## Config store url
## Keep consistent with the same name configuration in mmc-meta.conf
ock.mmc.local_service.config_store_url = tcp://<P0_node_ip>:6000         
## Set to the IP of the node running MetaService.

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

## Data transfer protocol options
ock.mmc.local_service.protocol = device_sdma                          
## SDMA transfer within the supernode is typical for A3; use device_rdma on A2.

## DRAM space usage
ock.mmc.local_service.dram.size = 0GB                                
## Memory is provided by the standalone process; set this to 0 GB.

ock.mmc.local_service.max.dram.size = 1024GB                                
## New configuration
```

Note: Replace `127.0.0.1` in both places with the IP of the node running MetaService.

## Clear Cached Memory on All Nodes

**If the nodes have already been rebooted before deploying the service, you can skip this step.**

System caches may occupy memory that could be used by MemCache. Run `free -h` to check cache usage, then clear the caches with the following command to free more memory for MemCache. Note: Model startup may take longer afterward because cached model weights are removed and must be loaded again.

echo 3 > /proc/sys/vm/drop_caches

After cleanup, run `free -h` to check whether the buff/cache value decreased.

Alternatively, scan available memory with a script; see the following link.
[memfabric-hybrid - Wiki - AtomGit](https://gitcode.com/Ascend/memfabric_hybrid/wiki/FAQ.md#2-%E8%B0%83%E7%94%A8halmemcreate%E8%BF%94%E5%9B%9E%E9%94%99%E8%AF%AF%E7%A0%81-6)

## Start the Main Process

Note: Run this only on the node running MetaService. Start one MetaService service; the command can be run from any directory.

```
export MMC_META_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-meta.conf
# Start mmc-meta.
nohup python3 -c "from memcache_hybrid import MetaService; MetaService.main()" >> /data/mmc-log/run_meta_service.log 2>&1 &
```

## Start a Standalone Process on Each Node to Reserve Memory for MemCache

**Note: Perform the following steps on all four nodes, starting one standalone process on each.**

**Step 1:** **Copy the `mmc-local` configuration file**​，for the standalone MemCache process and change its requested memory capacity.

```
mkdir -p /data/deploy_glm5.2_dp32tp1
cp /usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf /data/deploy_glm5.2_dp32tp1/mmc-local.conf
```

**Step 2:** Make the following changes to the copied `mmc-local.conf`:

```
vim /data/deploy_glm5.2_dp32tp1/mmc-local.conf
```

Update the configuration:

```
ock.mmc.local_service.dram.size = 300GB
```

Note: Set this value on each node according to the environment. Total available KV pool capacity is the sum of `dram.size` across the four A3 nodes. For example, if each node is set to 300 GB, total pooled capacity is 300 GB × 4 = 1200 GB. Values may differ by node; this test used 300 GB per node.

The SSD pooling scheme uses two loop devices per node, each backed by a different physical disk. Each loop device is 1.75 TB, for 3.5 TB per node and 14 TB across four nodes. SSD pooling is disabled for the MMC-memory-pooling-only scheme.

**Step 3: Prepare two loop devices on each host.** The example uses two different SSD backing directories; `1792G` corresponds to about 1.75 TB. Replace both directory placeholders with actual local paths and confirm that each has enough free space:

```bash
fallocate -l 1792G <SSD0_BACKING_DIR>/ubsio-loop0.img
fallocate -l 1792G <SSD1_BACKING_DIR>/ubsio-loop1.img

LOOP0=$(losetup --find --show --direct-io=on <SSD0_BACKING_DIR>/ubsio-loop0.img)
LOOP1=$(losetup --find --show --direct-io=on <SSD1_BACKING_DIR>/ubsio-loop1.img)
echo "$LOOP0"
echo "$LOOP1"
```

The system assigns loop numbers dynamically; do not assume they will be `/dev/loop0`、`/dev/loop1`。Before adding them to the configuration, verify the actual mappings, capacity, Direct I/O, mount status, and signatures on the host:

```bash
losetup --list --output NAME,BACK-FILE,SIZE,DIO
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS "$LOOP0" "$LOOP1"
findmnt -S "$LOOP0"
findmnt -S "$LOOP1"
```

Both loop devices should have no mount points or data that needs to be preserved, and `BACK-FILE` should point to two local 1.75 TB image files, each stored on its intended physical SSD. The combined random 4 MB read bandwidth of the two local SSDs should be at least 7 GB/s. Create loop devices on the host. If the container was started earlier, restart the privileged Docker container with `docker restart <container-name>`, then confirm that both loop devices are visible inside it.

**Step 4: Edit only the configuration used by the standalone cache process: `/data/deploy_glm5.2_dp32tp1/mmc-local.conf`** In A3 disaggregated mode, only this standalone process on each node enables SSD pooling, so `device_count` is fixed at `1`:

```ini
ock.mmc.local_service.storage.enabled = true
ubsio.disk.path = <actual-local-loop0-path>:<actual-local-loop1-path>
ubsio.mem.size_in_gb = 80
ubsio.standalone.device_count = 1
ubsio.standalone.force_new_disk = true
```

Here, `80GB` is a typical value for A3 disaggregated mode; confirm that sufficient memory remains on the node before using it.`force_new_disk = true` initializes the two dedicated loop devices as new disks and does not restore old UBS IO metadata. Use it only with dedicated image files confirmed safe to overwrite.

This test used `ubsio.mem.size_in_gb = 80`; Meta watermarks were `50/70/90`. For the feature overview and complete parameter descriptions, see [DRAM-SSD-Multi-Tier-Pooling-Feature-Overview](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E7%89%B9%E6%80%A7%E4%BB%8B%E7%BB%8D.md) and [DRAM-SSD-Multi-Tier-Pooling-Configuration-Guide](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md)。

**Step 5:** Create a standalone process launch script.

`vim /data/deploy_glm5.2_dp32tp1/memcache.py`

Script contents:

```
import time

from memcache_hybrid import DistributedObjectStore


if __name__ == "__main__":
    store = DistributedObjectStore()
    res = store.init(0)
    if res != 0:
        print(f"Failed to initialize memcache, res = {res}", flush=True)
        exit(1)
    print("Successfully initialized memcache. ", flush=True)

    while True:
        time.sleep(1)
```

**Step 6:** Start the standalone process.

```
export MMC_LOCAL_CONFIG_PATH=/data/deploy_glm5.2_dp32tp1/mmc-local.conf
# Start the standalone process.
nohup python3 /data/deploy_glm5.2_dp32tp1/memcache.py >> /data/mmc-log/run_mmc-local.log 2>&1 &
```

**Note:** After running the standalone process launch script, do not immediately start the GLM-5.2 `vllm serve` service. Check `/data/mmc-log/run_mmc-local.log` and wait until all four nodes show the following log, indicating successful memory allocation. If allocation fails, lower `ock.mmc.local_service.dram.size` and restart the standalone process until all four processes start successfully.

```
Successfully initialized memcache.
```

For SSD pooling, also check the current startup log:

```bash
grep -i "boostio" /data/mmc-log/run_mmc-local.log
```

The standalone process has actually started SSD pooling only if matching `boostio` logs appear. If there are no matches or errors, check `storage.enabled`, loop-device visibility, the `device_count` value, and the detailed log at `/var/log/ubsio/bio${PID}.log`.

You can also quickly check the total KV pool size at `http://<P0_node_ip>:8000/metrics/summary`.

# Deploy the GLM-5.2 200K High-Performance Service

This section describes the 200K service configuration used in the performance tests at the end of this document. The script sets the server maximum model length with `--max-model-len 210000`; the test sequence length is 200K.

## Prepare the launch_online_dp.py Script in Containers on All Nodes

```
import argparse
import multiprocessing
import os
import subprocess
import sys

def parse_args():
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--dp-size",
        type=int,
        required=True,
        help="Data parallel size."
    )
    parser.add_argument(
        "--tp-size",
        type=int,
        default=1,
        help="Tensor parallel size."
    )
    parser.add_argument(
        "--dp-size-local",
        type=int,
        default=-1,
        help="Local data parallel size."
    )
    parser.add_argument(
        "--dp-rank-start",
        type=int,
        default=0,
        help="Starting rank for data parallel."
    )
    parser.add_argument(
        "--dp-address",
        type=str,
        required=True,
        help="IP address for data parallel master node."
    )
    parser.add_argument(
        "--dp-rpc-port",
        type=str,
        default=12345,
        help="Port for data parallel master node."
    )
    parser.add_argument(
        "--vllm-start-port",
        type=int,
        default=9000,
        help="Starting port for the engine."
    )
    return parser.parse_args()

args = parse_args()
dp_size = args.dp_size
tp_size = args.tp_size
dp_size_local = args.dp_size_local
if dp_size_local == -1:
    dp_size_local = dp_size
dp_rank_start = args.dp_rank_start
dp_address = args.dp_address
dp_rpc_port = args.dp_rpc_port
vllm_start_port = args.vllm_start_port

def run_command(visible_devices, dp_rank, vllm_engine_port):
    command = [
        "bash",
        "./run_dp_template.sh",
        visible_devices,
        str(vllm_engine_port),
        str(dp_size),
        str(dp_rank),
        dp_address,
        dp_rpc_port,
        str(tp_size),
    ]
    subprocess.run(command, check=True)

if __name__ == "__main__":
    template_path = "./run_dp_template.sh"
    if not os.path.exists(template_path):
        print(f"Template file {template_path} does not exist.")
        sys.exit(1)

    processes = []
    num_cards = dp_size_local * tp_size
    for i in range(dp_size_local):
        dp_rank = dp_rank_start + i
        vllm_engine_port = vllm_start_port + i
        visible_devices = ",".join(str(x) for x in range(i * tp_size, (i + 1) * tp_size))
        process = multiprocessing.Process(target=run_command,
                                        args=(visible_devices, dp_rank,
                                                vllm_engine_port))
        processes.append(process)
        process.start()

    for process in processes:
        process.join()
```

## Prepare the run_dp_template.sh Deployment Script in Containers on All Nodes

P node

```
nic_name="bond0" # change to your own nic name
local_ip="100.xx.xx.xx" # change to your own ip

export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export HCCL_OP_EXPANSION_MODE="AIV"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

export OMP_PROC_BIND=false
export OMP_NUM_THREADS=1
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export HCCL_BUFFSIZE=400

export ACL_OP_INIT_MODE=1
export ASCEND_A3_ENABLE=1

export ASCEND_RT_VISIBLE_DEVICES=$1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

#memcache
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480
export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf
export VLLM_EXECUTE_MODEL_TIMEOUT_SECONDS=1800

vllm serve /data/GLM-5.2-w4a8c8 \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --speculative-config '{"num_speculative_tokens":1, "method":"deepseek_mtp","enforce_eager":true}' \
    --seed 1024 \
    --served-model-name glm-52 \
    --max-model-len 210000 \
    --additional-config '{"recompute_scheduler_enable" : false,"multistream_overlap_shared_expert": true, "enable_dsa_cp":true,"enable_sparse_sfa_c8": false, "enable_sparse_li_c8": true,"c8_enable_reshape_optim":false}' \
    --max-num-batched-tokens 8192 \
    --trust-remote-code \
    --max-num-seqs 64 \
    --quantization ascend \
    --gpu-memory-utilization 0.92 \
    --enforce-eager \
    --enable-auto-tool-choice \
    --tool-call-parser glm47 \
    --reasoning-parser glm45 \
    --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_producer",
        "kv_load_failure_policy": "recompute",
        "engine_id": "0",
        "kv_connector_extra_config": {
            "connectors": [
                {
                    "kv_connector": "MooncakeConnectorV1",
                    "kv_role": "kv_producer",
                    "kv_port": "40000",
                    "kv_connector_extra_config": {
                        "use_ascend_direct": true,
                        "prefill": {"dp_size": 4, "tp_size": 8},
                        "decode": {"dp_size": 32, "tp_size": 1}
                    }
                },
                {
                    "kv_connector": "AscendStoreConnector",
                    "kv_role": "kv_producer",
                    "kv_connector_extra_config": {
                        "backend": "memcache",
                        "lookup_rpc_port": "0"
                    }
                }
            ]
        }
    }'
```

D node

```
nic_name="eth-bond4" # change to your own nic name
local_ip="100.xx.xx.xx" # change to your own ip

export HCCL_OP_EXPANSION_MODE="AIV"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

#Mooncake
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=1

export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export HCCL_BUFFSIZE=256
export TASK_QUEUE_ENABLE=1
export ASCEND_RT_VISIBLE_DEVICES=$1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
export VLLM_ASCEND_ENABLE_FUSED_MC2=1

#memcache
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480
export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf
export VLLM_EXECUTE_MODEL_TIMEOUT_SECONDS=1800

vllm serve /data/GLM-5.2-w4a8c8 \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --seed 1024 \
    --served-model-name glm-52 \
    --max-model-len 210000 \
    --max-num-batched-tokens 164 \
    --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY"}' \
    --speculative-config '{"num_speculative_tokens": 5,  "method":"deepseek_mtp","enforce_eager":true}' \
    --additional-config '{"recompute_scheduler_enable":true,"multistream_overlap_shared_expert":true,"enable_sparse_sfa_c8": false, "enable_sparse_li_c8": true}' \
    --trust-remote-code \
    --max-num-seqs 32 \
    --gpu-memory-utilization 0.92 \
    --async-scheduling \
    --quantization ascend \
    --enable-auto-tool-choice \
    --tool-call-parser glm47 \
    --reasoning-parser glm45 \
    --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_consumer",
        "kv_load_failure_policy": "recompute",
        "engine_id": "1",
        "kv_connector_extra_config": {
            "connectors": [
                {
                    "kv_connector": "MooncakeConnectorV1",
                    "kv_role": "kv_consumer",
                    "kv_port": "40100",
                    "kv_connector_extra_config": {
                        "use_ascend_direct": true,
                        "prefill": {"dp_size": 4, "tp_size": 8},
                        "decode": {"dp_size": 32, "tp_size": 1}
                    }
                },
                {
                    "kv_connector": "AscendStoreConnector",
                    "kv_role": "kv_consumer",
                    "kv_connector_extra_config": {
                        "backend": "memcache",
                        "lookup_rpc_port": "2"
                    }
                }
            ]
        }
    }'
```

## Start the Inference Services on Each Node in Turn

Start Prefill1 Node0 by creating and running `p0-run.sh`. Replace the highlighted values with the actual IP address and port:

```
nohup python launch_online_dp.py \
  --dp-size 4 --tp-size 8 --dp-size-local 2 \
  --dp-rank-start 0 --dp-address <P0_node_ip> \
  --dp-rpc-port 10521 --vllm-start-port 6700 \
  >> launch_dp_output.log 2>&1 &
```

Start Prefill1 Node1 by creating and running `p1-run.sh`. Replace the highlighted values:

```
nohup python launch_online_dp.py \
  --dp-size 4 --tp-size 8 --dp-size-local 2 \
  --dp-rank-start 2 --dp-address <P0_node_ip> \
  --dp-rpc-port 10521 --vllm-start-port 6700 \
  >> launch_dp_output.log 2>&1 &
```

Start Decode1 Node0 by creating and running `d0-run.sh`. Replace the highlighted values:

```
nohup python launch_online_dp.py \
  --dp-size 32 --tp-size 1 --dp-size-local 16 \
  --dp-rank-start 0 --dp-address <D0_node_ip> \
  --dp-rpc-port 10521 --vllm-start-port 6700 \
  >> launch_dp_output.log 2>&1 &
```

Start Decode1 Node1 by creating and running `d1-run.sh`. Replace the highlighted values:

```
nohup python launch_online_dp.py \
  --dp-size 32 --tp-size 1 --dp-size-local 16 \
  --dp-rank-start 16 --dp-address <D0_node_ip> \
  --dp-rpc-port 10521 --vllm-start-port 6700 \
  >> launch_dp_output.log 2>&1 &
```

## Expose the Service IP and Port Through the Proxy

The script reads the version from the file under `/vllm-workspace` in the container:

```
root@localhost:/vllm-workspace# find -name load_balance_proxy_server_example.py
./vllm-ascend/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py
```

Create a `start_proxy.sh` script to start the proxy forwarding service. Its contents are:

```
#!/bin/bash

# Clear proxy settings.
unset http_proxy
unset https_proxy

# Run the program in the background.
nohup python load_balance_proxy_server_example.py \
    --port 8088 \
    --host <P0_node_ip> \
    --prefiller-hosts \
       "<P0_node_ip>" \
       "<P0_node_ip>" \
       "<P1_node_ip>" \
       "<P1_node_ip>" \
    --prefiller-ports \
       6700 6701\
       6700 6701\
    --decoder-hosts \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D0_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
      "<D1_node_ip>" \
    --decoder-ports \
      6700 6701 6702 6703 6704 6705 6706 6707 6708 6709 6710 6711 6712 6713 6714 6715 \
      6700 6701 6702 6703 6704 6705 6706 6707 6708 6709 6710 6711 6712 6713 6714 6715 \
    >> start_proxy.log 2>&1 &
```

After creating the script, run the following command to start the proxy service:
`bash start_proxy.sh`

