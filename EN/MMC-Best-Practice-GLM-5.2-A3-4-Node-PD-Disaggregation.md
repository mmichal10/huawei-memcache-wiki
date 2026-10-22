# Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.

GLM-5.2 is a flagship model for long-horizon tasks. It supports a practical 1M context and has been shown to handle project-scale engineering contexts. It executes long-running tasks more reliably, follows engineering standards more consistently, and further improves success rates in development scenarios. A single task can complete the full development workflow from requirements to deployable artifacts across multiple platforms. Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.

# Deployment Preparation

## Prerequisites

**Hardware:** 4 × Atlas 800I A3

**Driver firmware:** >=25.5.1

【L1 1520 CC】>=1.5.0

**OS kernel version:** >=5.10

## Environment Setup

GLM-5.2-w4a8c8 model weights download:
[*https://modelscope.cn/models/Eco-Tech/GLM-5.2-w4a8c8*](https://modelscope.cn/models/Eco-Tech/GLM-5.2-w4a8c8)

Pull the vLLM image:
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

# MemCache

## Installation
The official vllm-ascend image already includes MemCache and its dependencies. Check installed versions with:
```shell
pip list | grep mem
```

The currently recommended stable version is 1.1.5. Install it with:

```shell
pip install memfabric_hybrid==1.1.5
pip install memcache_hybrid==1.1.5
```

If network restrictions prevent online installation, download the packages from [PyPI · The Python Package Index](https://pypi.org/) and install them offline.

Download links:

[memfabric](https://pypi.org/project/memfabric-hybrid/#history)

[memcache](https://pypi.org/project/memcache-hybrid/#history)

Other unreleased versions can be installed by building from source; see the [official build guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/install_run.md).

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
Assume MetaService will run on node P0 (IP denoted **P0_node_ip**). Make the following changes to the `mmc-meta.conf` and `mmc-local.conf` files on P0, then copy `mmc-local.conf` to the same location on P1, D0, and D1 (all four nodes must have identical configuration).

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

System caches may occupy memory that could be used by MemCache. Run `free -h` to check cache usage, then clear the caches with the following command to free more memory for MemCache. Note: It is normal for model startup to take longer after clearing caches because cached model weights are removed and must be loaded again.

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

**Step 1:** **Copy the `mmc-local` configuration file** for the standalone MemCache process and change its requested memory capacity.

```
mkdir -p /data/glm-5.2-start
cp /usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf /data/deploy_glm5.2_dp32tp1/mmc-local.conf
```

**Step 2:** Make the following changes to the copied `mmc-local.conf`:

```
vim /data/deploy_glm5.2_dp32tp1/mmc-local.conf
```

Update the configuration:

```
ock.mmc.local_service.dram.size = 600GB
```

Note: Set this value on each node according to the environment. The total available KV pool is the sum of the `dram.size` values across all four A3 nodes. For example, if each node is set to 600 GB, the total pooled capacity is 600 GB × 4 = 2400 GB. The values can differ by node.
**Step 3:** Create a standalone process launch script.

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

**Step 4:** Start the standalone process.

```
export MMC_LOCAL_CONFIG_PATH=/data/deploy_glm5.2_dp32tp1/mmc-local.conf
# Start the standalone process.
nohup python3 /data/deploy_glm5.2_dp32tp1/memcache.py >> /data/mmc-log/run_mmc-local.log 2>&1 &
```

**Note:** After running the standalone process launch script, do not immediately start the GLM-5.2 `vllm serve` service. Check `/data/mmc-log/run_mmc-local.log`; wait until all four nodes show the following log, indicating that their standalone processes allocated memory successfully. If memory allocation fails, lower `ock.mmc.local_service.dram.size` and restart the standalone process until all four processes start successfully.

```
Successfully initialized memcache.
```

You can also quickly check the total KV pool size at `http://<P0_node_ip>:8000/metrics/summary`.

# Deploy the GLM-5.2 High-Performance Service

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

# Deploy the GLM-5.2 Long-Sequence (1M) Service

Use the `launch_online_dp.py` script from the previous high-performance service section.

## Deployment Script Configuration for All Nodes

P0 node configuration

```
nic_name="bond0"
local_ip="<P0_node_ip>"
node_p0_ip="<P0_node_ip>"
# Prefiller node 0: data_parallel_start_rank=0, server_role_args="--api-server-count 1"
# Prefiller node 1: data_parallel_start_rank=2, server_role_args="--headless"
data_parallel_start_rank=0
server_role_args="--api-server-count 1"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

export VLLM_ASCEND_ENABLE_NZ=1
export HCCL_OP_EXPANSION_MODE="AIV"
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=20
export HCCL_BUFFSIZE=768
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_WORKER_MULTIPROC_METHOD=spawn
export TASK_QUEUE_ENABLE=1

#memcache
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480
export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf

vllm serve /data/GLM-5.2-w4a8c8 \
  --seed 1024 \
  --host 0.0.0.0 \
  --port 9081 \
  --served-model-name glm-52 \
  --max-model-len 1040000 \
  --max-num-batched-tokens 16384 \
  --gpu-memory-utilization 0.78 \
  ${server_role_args} \
  --max-num-seqs 8 \
  --data-parallel-size 4 \
  --data-parallel-size-local 2 \
  --data-parallel-start-rank $data_parallel_start_rank \
  --data-parallel-address $node_p0_ip \
  --data-parallel-rpc-port 10521 \
  --pipeline-parallel-size 1 \
  --tensor-parallel-size 8 \
  --prefill-context-parallel-size 1 \
  --decode-context-parallel-size 8 \
  --cp-kv-cache-interleave-size 128 \
  --enforce-eager \
  --additional-config '{"enable_flashcomm1": true, "enable_dsa_cp": true, "ascend_compilation_config": {"enable_npugraph_ex": true, "enable_static_kernel": false}, "fuse_muls_add": true, "multistream_overlap_shared_expert": true, "enable_mc2_hierarchy_comm": false, "enable_sparse_sfa_c8": false, "enable_sparse_li_c8": true, "enable_cpu_binding": true, "recompute_scheduler_enable": true}' \
  --speculative-config '{"num_speculative_tokens": 1, "method": "deepseek_mtp", "enforce_eager": true}' \
  --quantization ascend \
  --enable-expert-parallel \
  --safetensors-load-strategy prefetch \
  --enable-auto-tool-choice \
  --tool-call-parser glm47 \
  --reasoning-parser glm45 \
  --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_producer",
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
                        "decode": {"dp_size": 4, "tp_size": 8}
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

P1 node configuration

```
nic_name="bond0"
local_ip="<P1_node_ip>"
node_p0_ip="<P0_node_ip>"
# Prefiller node 0: data_parallel_start_rank=0, server_role_args="--api-server-count 1"
# Prefiller node 1: data_parallel_start_rank=2, server_role_args="--headless"
data_parallel_start_rank=2
server_role_args="--headless"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

export VLLM_ASCEND_ENABLE_NZ=1
export HCCL_OP_EXPANSION_MODE="AIV"
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=20
export HCCL_BUFFSIZE=768
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_WORKER_MULTIPROC_METHOD=spawn
export TASK_QUEUE_ENABLE=1

#memcache
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480
export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf

vllm serve /data/GLM-5.2-w4a8c8 \
  --seed 1024 \
  --host 0.0.0.0 \
  --port 9081 \
  --served-model-name glm-52 \
  --max-model-len 1040000 \
  --max-num-batched-tokens 16384 \
  --gpu-memory-utilization 0.78 \
  ${server_role_args} \
  --max-num-seqs 8 \
  --data-parallel-size 4 \
  --data-parallel-size-local 2 \
  --data-parallel-start-rank $data_parallel_start_rank \
  --data-parallel-address $node_p0_ip \
  --data-parallel-rpc-port 10521 \
  --pipeline-parallel-size 1 \
  --tensor-parallel-size 8 \
  --prefill-context-parallel-size 1 \
  --decode-context-parallel-size 8 \
  --cp-kv-cache-interleave-size 128 \
  --enforce-eager \
  --additional-config '{"enable_flashcomm1": true, "enable_dsa_cp": true, "ascend_compilation_config": {"enable_npugraph_ex": true, "enable_static_kernel": false}, "fuse_muls_add": true, "multistream_overlap_shared_expert": true, "enable_mc2_hierarchy_comm": false, "enable_sparse_sfa_c8": false, "enable_sparse_li_c8": true, "enable_cpu_binding": true, "recompute_scheduler_enable": true}' \
  --speculative-config '{"num_speculative_tokens": 1, "method": "deepseek_mtp", "enforce_eager": true}' \
  --quantization ascend \
  --enable-expert-parallel \
  --safetensors-load-strategy prefetch \
  --enable-auto-tool-choice \
  --tool-call-parser glm47 \
  --reasoning-parser glm45 \
  --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_producer",
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
                        "decode": {"dp_size": 4, "tp_size": 8}
                    }
                },
                {
                    "kv_connector": "AscendStoreConnector",
                    "kv_role": "kv_producer",
                    "kv_connector_extra_config": {
                        "backend": "memcache",
                        "lookup_rpc_port": "1"
                    }
                }
            ]
        }
    }'
```

D0 node configuration

```
nic_name="eth-bond4"
local_ip="<D0_node_ip>"
node_d0_ip="<D0_node_ip>"
# Decoder node 0: data_parallel_start_rank=0, server_role_args="--api-server-count 1"
# Decoder node 1: data_parallel_start_rank=2, server_role_args="--headless"
data_parallel_start_rank=0
server_role_args="--api-server-count 1"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

export VLLM_ASCEND_ENABLE_NZ=1
export HCCL_OP_EXPANSION_MODE="AIV"
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=20
export HCCL_BUFFSIZE=768
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_WORKER_MULTIPROC_METHOD=spawn

#memcache
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480
export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf
export TASK_QUEUE_ENABLE=1

vllm serve /data/GLM-5.2-w4a8c8 \
  --seed 1024 \
  --host 0.0.0.0 \
  --port 9900 \
  --served-model-name glm-52 \
  --max-model-len 1040000 \
  --max-num-batched-tokens 128 \
  --gpu-memory-utilization 0.93 \
  ${server_role_args} \
  --max-num-seqs 32 \
  --data-parallel-size 4 \
  --data-parallel-size-local 2 \
  --data-parallel-start-rank $data_parallel_start_rank \
  --data-parallel-address $node_d0_ip \
  --data-parallel-rpc-port 16600 \
  --pipeline-parallel-size 1 \
  --tensor-parallel-size 8 \
  --prefill-context-parallel-size 1 \
  --decode-context-parallel-size 8 \
  --cp-kv-cache-interleave-size 128 \
  --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY"}' \
  --additional-config '{"enable_flashcomm1": false, "enable_dsa_cp": false, "ascend_compilation_config": {"enable_npugraph_ex": true, "enable_static_kernel": false}, "fuse_muls_add": true, "multistream_overlap_shared_expert": true, "enable_mc2_hierarchy_comm": false, "enable_sparse_sfa_c8": false, "enable_sparse_li_c8": true, "enable_cpu_binding": true, "recompute_scheduler_enable": true}' \
  --speculative-config '{"num_speculative_tokens": 3, "method": "deepseek_mtp", "enforce_eager": true}' \
  --quantization ascend \
  --enable-expert-parallel \
  --safetensors-load-strategy prefetch \
  --enable-auto-tool-choice \
  --tool-call-parser glm47 \
  --reasoning-parser glm45 \
  --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_consumer",
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
                        "decode": {"dp_size": 4, "tp_size": 8}
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

D1 node configuration

```
nic_name="eth-bond4"
local_ip="<D1_node_ip>"
node_d0_ip="<D0_node_ip>"
# Decoder node 0: data_parallel_start_rank=0, server_role_args="--api-server-count 1"
# Decoder node 1: data_parallel_start_rank=2, server_role_args="--headless"
data_parallel_start_rank=2
server_role_args="--headless"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

export VLLM_ASCEND_ENABLE_NZ=1
export HCCL_OP_EXPANSION_MODE="AIV"
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=20
export HCCL_BUFFSIZE=768
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_WORKER_MULTIPROC_METHOD=spawn

#memcache
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480
export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf
export TASK_QUEUE_ENABLE=1

vllm serve /data/GLM-5.2-w4a8c8 \
  --seed 1024 \
  --host 0.0.0.0 \
  --port 9900 \
  --served-model-name glm-52 \
  --max-model-len 1040000 \
  --max-num-batched-tokens 128 \
  --gpu-memory-utilization 0.93 \
  ${server_role_args} \
  --max-num-seqs 32 \
  --data-parallel-size 4 \
  --data-parallel-size-local 2 \
  --data-parallel-start-rank $data_parallel_start_rank \
  --data-parallel-address $node_d0_ip \
  --data-parallel-rpc-port 16600 \
  --pipeline-parallel-size 1 \
  --tensor-parallel-size 8 \
  --prefill-context-parallel-size 1 \
  --decode-context-parallel-size 8 \
  --cp-kv-cache-interleave-size 128 \
  --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY"}' \
  --additional-config '{"enable_flashcomm1": false, "enable_dsa_cp": false, "ascend_compilation_config": {"enable_npugraph_ex": true, "enable_static_kernel": false}, "fuse_muls_add": true, "multistream_overlap_shared_expert": true, "enable_mc2_hierarchy_comm": false, "enable_sparse_sfa_c8": false, "enable_sparse_li_c8": true, "enable_cpu_binding": true, "recompute_scheduler_enable": true}' \
  --speculative-config '{"num_speculative_tokens": 3, "method": "deepseek_mtp", "enforce_eager": true}' \
  --quantization ascend \
  --enable-expert-parallel \
  --safetensors-load-strategy prefetch \
  --enable-auto-tool-choice \
  --tool-call-parser glm47 \
  --reasoning-parser glm45 \
  --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_consumer",
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
                        "decode": {"dp_size": 4, "tp_size": 8}
                    }
                },
                {
                    "kv_connector": "AscendStoreConnector",
                    "kv_role": "kv_consumer",
                    "kv_connector_extra_config": {
                        "backend": "memcache",
                        "lookup_rpc_port": "3"
                    }
                }
            ]
        }
    }'
```

## Start the Inference Services on Each Node in Turn

Save the four launch scripts from the previous section as the corresponding `*-run.sh` files, then run each on its respective machine:
`nohup bash p0-run.sh >> launch_dp_output.log 2>&1 &`
`nohup bash p1-run.sh >> launch_dp_output.log 2>&1 &`
`nohup bash d0-run.sh >> launch_dp_output.log 2>&1 &`
`nohup bash d1-run.sh >> launch_dp_output.log 2>&1 &`

## Expose the Service IP and Port Through the Proxy

Create a `start_proxy.sh` script to start the proxy forwarding service. Its contents are:

```
#!/bin/bash

# Clear proxy settings.
unset http_proxy
unset https_proxy

# Run the program in the background.
nohup python /vllm-workspace/vllm-ascend/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py \
  --host 0.0.0.0 \
  --port 8088 \
  --prefiller-hosts <P0_node_ip> \
  --prefiller-ports 9081 \
  --decoder-hosts <D0_node_ip> \
  --decoder-ports 9900 \
    >> start_proxy.log 2>&1 &
```

After creating the script, run the following command to start the proxy service:
`bash start_proxy.sh`

# Performance Test

Four scenarios were tested for MemCache prefix caching on a four-node A3 cluster using the quantized GLM5.2-w4a8c8 model and a PD-disaggregated architecture (two Prefill nodes and two Decode nodes), with the high-performance service configuration. For cached tests, the number of distinct prefixes was half the concurrency, prefix reuse was 90%, and prefixes were warmed with `--dp 4`, simulating AI coding workloads with multiple users and prefixes. Key results:

| Scenario | Input Len | Actual concurrency | Cache | TTFT(ms) | TTFT reduction | Output throughput (token/s) | Throughput increase |
| ------------ | --------- | --------- | ---------- | -------- | -------- | ----------------- | ------------ |
| 64K × 20CC | 65536 | 17.71 | No cache | 23,405 | / | 436.3 | / |
| 64K × 20CC | 65536 | 18.46 | **Cached** | 4,140 | 82.3% | 787.5 | 80.5% |
| 64K × 30CC | 65536 | 26.48 | No cache | 40,191 | / | 466.5 | / |
| 64K × 30CC | 65536 | 27.11 | **Cached** | 4,980 | 87.6% | 1155.3 | 147.7% |
| 128K × 10CC | 131072 | 9.16 | No cache | 35,017 | / | 173.3 | / |
| 128K × 10CC | 131072 | 9.15 | **Cached** | 5,775 | 83.5% | 367.6 | 112.1% |
| 128K × 20CC | 131072 | 17.64 | No cache | 83,090 | / | 177.9 | / |
| 128K × 20CC | 131072 | 17.97 | **Cached** | 12,235 | 85.3% | 558.8 | 214.1% |

Authors: Guinan Zhang, Can Song, and Xidong Kou

