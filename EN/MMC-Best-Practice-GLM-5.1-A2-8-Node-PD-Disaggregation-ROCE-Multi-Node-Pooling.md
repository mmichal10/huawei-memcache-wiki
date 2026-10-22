

### Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.

**GLM-5.1** is Zhipu's latest flagship model, with greatly enhanced coding ability and **long-horizon task** performance. It can work continuously and autonomously for up to 8 hours on a single task, completing the full cycle from planning and execution to iterative optimization and delivering engineering-grade results. Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.



### Environment

**Hardware:** 8 × A2

**Model weights:** GLM-5.1-w8a8

<span style="color:#e60000;">Note: This deployment requires the following driver and software versions.</span>

**NPU driver firmware:** 25.5.1 or later [Download](https://www.hiascend.com/hardware/firmware-drivers/old/community?product=4&model=32&cann=All&driver=Ascend+HDK+25.5.1)

**OS page size:** Run `getconf PAGESIZE` to check the system page size. MemCache can be used only if the result is 4K. If the result is 64K, consider changing the OS.


### Deployment


#### 1. Pull the official vllm-ascend image

```shell
docker pull quay.nju.edu.cn/ascend/vllm-ascend:v0.18.0
```



#### 2. Install MemCache

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

Other unreleased versions can be installed by building from source; see the [official build guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/install_run.md).


#### 3. Prepare the MemCache Configuration Files

MemCache consists of MetaService and LocalService. LocalService is integrated with the inference framework and starts with the inference process; it handles memory allocation, data reads/writes, and transfers. MetaService runs separately, connects to LocalService over TCP, and handles metadata management, data eviction, primary/standby switching, and other high-availability functions.

At startup, MetaService and LocalService load settings from separate configuration files. By default, these files are in the config directory of the MemCache wheel package.

```
/usr/local/lib/python3.11/site-packages/memcache_hybrid/config
```


##### MetaService Configuration File

Choose a node to run MetaService. Set the IP addresses in `ock.mmc.meta_service_url` and `ock.mmc.meta_service.config_store_url` to that node's IP, then save the file as `mmc-meta.conf`.

```ini
## Set the IP to the IP of the node running MetaService.
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## Set the IP to the IP of the node running MetaService.
ock.mmc.meta_service.config_store_url = tcp://127.0.0.1:6000

## HTTP service url
ock.mmc.meta_service.metrics_url = http://127.0.0.1:8000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md)
```



##### LocalService Configuration File

Prepare an `mmc-local.conf` file on each node, and adjust the IP and memory capacity for your environment.

```ini
## Set to the IP of the node running MetaService.
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## Set to the IP of the node running MetaService.
ock.mmc.local_service.config_store_url = tcp://127.0.0.1:6000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

## Card-side RoCE network transfer is typical for A2.
ock.mmc.local_service.protocol = device_rdma

## Configure 100 GB per card; adjust according to available memory in the environment.
ock.mmc.local_service.dram.size = 100GB
ock.mmc.local_service.max.dram.size = 1024GB

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md)
```



#### 4. Prepare the vLLM Launch Scripts

##### 4.1 Service Launch Helper Script

Prepare the `launch_online_dp.py` script on each node:

```python3
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



##### 4.2 vLLM Launch Script

Prepare the `run_dp_template.sh` script on all P nodes.

```shell
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
export LD_PRELOAD=/usr/lib/aarch64-linux-gnu/libjemalloc.so.2:$LD_PRELOAD
export VLLM_ASCEND_ENABLE_MLAPO=1
export VLLM_ASCEND_ENABLE_NZ=1
export HCCL_OP_EXPANSION_MODE="AIV"
nic_name="eno0"
local_ip=10.127.40.56
export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export VLLM_USE_V1=1
export HCCL_BUFFSIZE=200
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export TASK_QUEUE_ENABLE=1
export CPU_AFFINITY_CONF=1
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1
export ASCEND_AGGREGATE_ENABLE=1
export ASCEND_TRANSPORT_PRINT=1
export ACL_OP_INIT_MODE=1
export VLLM_NIXL_ABORT_REQUEST_TIMEOUT=300000

export HCCL_INTRA_ROCE_ENABLE=1
export HCCL_INTRA_PCIE_ENABLE=0

export ASCEND_RT_VISIBLE_DEVICES=$1

export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480
export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.11.14/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0
export HCCL_EXEC_TIMEOUT=3600
export HCCL_CONNECT_TIMEOUT=600

unset LOCAL_RANK

vllm serve /data02/weight/GLM-5.1-w8a8_modelers \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --enable-chunked-prefill \
    --enable-prefix-caching \
    --seed 1024 \
    --served-model-name glm5 \
    --max-model-len 202752 \
    --max-num-batched-tokens 2048 \
    --trust-remote-code \
    --max-num-seqs 24 \
    --gpu-memory-utilization 0.90 \
    --quantization ascend \
    --async-scheduling \
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
                    "kv_connector_module_path": "vllm_ascend.distributed.mooncake_connector",
                    "kv_port": "30000",
                    "kv_connector_extra_config": {
                        "use_ascend_direct": true,
                        "prefill": {"dp_size": 4, "tp_size": 8},
                        "decode": {"dp_size": 8, "tp_size": 4}
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
    }'  \
    --additional-config \
    '{
        "recompute_scheduler_enable": false,
        "multistream_overlap_shared_expert": true,
        "fuse_qknorm_rope": false,
        "fuse_muls_add": true,
        "enable_npugraph_ex": true,
	    "layer_sharding": ["q_b_proj","o_proj"]
    }' \
    --speculative-config '{"num_speculative_tokens": 3, "method": "deepseek_mtp"}' \
    --profiler-config \
    '{
        "profiler": "torch",
        "torch_profiler_dir": "/workspace/profiling",
        "torch_profiler_with_stack": false
    }' \
    2>&1 | tee glm.log

```

Prepare the `run_dp_template.sh` script on all D nodes.

```shell
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib
export LD_PRELOAD=/usr/lib/aarch64-linux-gnu/libjemalloc.so.2:$LD_PRELOAD

export VLLM_ASCEND_ENABLE_MLAPO=1
export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export VLLM_ASCEND_ENABLE_NZ=1
export HCCL_OP_EXPANSION_MODE="AIV"
nic_name="eno0"
local_ip=10.127.39.11
export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export VLLM_USE_V1=1
export HCCL_BUFFSIZE=200
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export TASK_QUEUE_ENABLE=1
export CPU_AFFINITY_CONF=1
export ASCEND_AGGREGATE_ENABLE=1
export ASCEND_TRANSPORT_PRINT=1
export ACL_OP_INIT_MODE=1
export VLLM_NIXL_ABORT_REQUEST_TIMEOUT=300000
export HCCL_INTRA_ROCE_ENABLE=1
export HCCL_INTRA_PCIE_ENABLE=0

export ASCEND_RT_VISIBLE_DEVICES=$1

export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480
export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.11.14/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0
export HCCL_EXEC_TIMEOUT=3600
export HCCL_CONNECT_TIMEOUT=600

unset LOCAL_RANK

vllm serve /data02/weight/GLM-5.1-w8a8_modelers \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --enable-chunked-prefill \
    --enable-prefix-caching \
    --seed 1024 \
    --served-model-name glm5 \
    --max-model-len 202752 \
    --max-num-batched-tokens 64 \
    --trust-remote-code \
    --max-num-seqs 48 \
    --gpu-memory-utilization 0.90 \
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
                    "kv_connector_module_path": "vllm_ascend.distributed.mooncake_connector",
                    "kv_port": "30100",
                    "kv_connector_extra_config": {
                        "use_ascend_direct": true,
                        "prefill": {"dp_size": 4, "tp_size": 8},
                        "decode": {"dp_size": 8, "tp_size": 4}
                    }
                },
                {
                    "kv_connector": "AscendStoreConnector",
                    "kv_role": "kv_consumer",
                    "kv_connector_extra_config": {
                        "backend": "memcache",
                        "lookup_rpc_port": "4"
                    }
                }
            ]
        }
    }'  \
    --compilation-config \
    '{
        "cudagraph_mode": "FULL_DECODE_ONLY"
     }' \
    --additional-config \
    '{
        "recompute_scheduler_enable": false,
        "multistream_overlap_shared_expert": true,
        "fuse_qknorm_rope": false,
        "fuse_muls_add": true,
        "enable_npugraph_ex": true
    }' \
    --speculative-config '{"num_speculative_tokens": 3,  "method": "deepseek_mtp"}' \
    --profiler-config \
    '{
        "profiler": "torch",
        "torch_profiler_dir": "/workspace/profiling",
        "torch_profiler_with_stack": false
    }' \
    2>&1 | tee glm.log

```

**Adjust the following values for your environment:**

- `local_ip`: Local node IP address.
- `nic_name`: Local network interface name.
- Model weight path.



**Compared with the baseline script, enabling MF requires only the following changes:**

1. Add environment variables.

```shell
# Set the path to the MemCache LocalService configuration file (the wheel package's default configuration path is shown here).
export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
# Set the Python hash seed so different instances generate the same key for identical requests.
export PYTHONHASHSEED=0
```

2. `kv-transfer-config` now uses both MooncakeConnectorV1 and AscendStoreConnector instead of MooncakeConnectorV1 alone.

- MooncakeConnectorV1 transfers the KV Cache for each request from P to D.
- AscendStoreConnector calls the MemCache read/write interfaces to enable prefix-cache reuse across requests.



##### 4.3 Request Forwarding Script

Get the proxy program [load_balance_proxy_server_example.py](https://github.com/vllm-project/vllm-ascend/blob/main/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py) from the vllm-ascend repository.

Start the forwarding service with the following script (update the IP addresses and ports to match the services).

```shell
unset http_proxy
unset https_proxy

python load_balance_proxy_server_example.py \
    --port 8000 \
    --host 127.0.0.1 \
    --prefiller-hosts \
        <prefill_node_0_ip> \
        <prefill_node_1_ip> \
        <prefill_node_2_ip> \
        <prefill_node_3_ip> \
    --prefiller-ports \
        6700 \
        6700 \
        6700 \
        6700 \
    --decoder-hosts \
        <decode_node_0_ip> \
        <decode_node_0_ip> \
        <decode_node_1_ip> \
        <decode_node_1_ip> \
        <decode_node_2_ip> \
        <decode_node_2_ip> \
        <decode_node_3_ip> \
        <decode_node_3_ip> \
    --decoder-ports \
        6721 6722 \
        6721 6722 \
        6721 6722 \
        6721 6722
```



#### 5. Start the Services

##### 5.1 Configure Huge-Page Memory on Each Node

Run the following command on the host to inspect the huge-page configuration, obtain Hugepagesize, and calculate the number of huge pages to configure.

```shell
grep -i huge /proc/meminfo
```

Calculate it as follows:

```
Huge pages required per machine = memory per card × cards used by the service / huge-page size
```

For example, with eight cards per server, 100 GB per card, and a 2 MB huge-page size, each machine requires `100GB * 8 / 2MB = 409600` huge pages.

Set the number of huge pages with:

```shell
echo 409600 > /proc/sys/vm/nr_hugepages
```
For a non-root user, use:
```shell
echo 409600 | sudo tee /proc/sys/vm/nr_hugepages
```



##### 5.2 Start the MetaService Metadata Management Service

On the node selected earlier to run MetaService, start the metadata management service with the following script.

```shell
export MMC_META_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```



##### 5.3 Start the Inference Service on All Nodes

###### P0~P4

```shell
python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 1 --dp-rank-start 0 --dp-address <prefill_node_0_ip> --dp-rpc-port 63521 --vllm-start-port 6700

python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 1 --dp-rank-start 1 --dp-address <prefill_node_0_ip> --dp-rpc-port 63521 --vllm-start-port 6700

python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 1 --dp-rank-start 2 --dp-address <prefill_node_0_ip> --dp-rpc-port 63521 --vllm-start-port 6700

python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 1 --dp-rank-start 3 --dp-address <prefill_node_0_ip> --dp-rpc-port 63521 --vllm-start-port 6700
```

###### D0~D4

```shell
python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 2 --dp-rank-start 0 --dp-address <decode_node_0_ip> --dp-rpc-port 63523 --vllm-start-port 6721

python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 2 --dp-rank-start 2 --dp-address <decode_node_0_ip> --dp-rpc-port 63523 --vllm-start-port 6721

python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 2 --dp-rank-start 4 --dp-address <decode_node_0_ip> --dp-rpc-port 63523 --vllm-start-port 6721

python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 2 --dp-rank-start 6 --dp-address <decode_node_0_ip> --dp-rpc-port 63523 --vllm-start-port 6721
```



##### 5.4 Start the Request Forwarding Script on Any Node

```shell
bash start_proxy.sh
```


### Test Results

SGLang multi-turn conversation benchmark tool (Zimeng Gao)

| MMC | Concurrency | TTFT (s) | Average duration (s) | Average decode duration (s) | AverageOutputLength | TPOT (ms) | Output TPS per stream | Overall hit rate |
|---------|----|----------|----------|----------------|---------------------|-----------|---------|------|
| Disable | 20 | 37.44    | 77.83    | 40.39          | 450.99              | 89.56     | 11.17   |      |
| Disable | 20 | 33.7     | 76.60    | 42.90          | 450.32              | 95.27     | 10.50   |      |
| Disable | 20 | 40.65    | 82.62    | 42.97          | 457.32              | 91.77     | 10.90   |      |
| Disable | 20 | 42.65    | 81.09    | 38.44          | 456.98              | 84.12     | 11.89   | 27%  |
| Enable  | 20 | 15.62    | 37.64    | 22.02          | 457.74              | 48.11     | 20.79   | 88%  |


90% fixed hit rate (Xidong Kou)

| Scenario | InputLen | OutputLen | Concurrency | TTFT without cache (ms) | TTFT with cache (ms) | TTFT reduction | TPOT without cache (ms) | TPOT with cache (ms) |
|------|----------|-----------|------|----------------|----------------|----------|----------------|----------------|
| 32K-20 concurrency | 32K | 1024 | 20 | 26884.4 | 11426.4 | 57.5% | 24.5 | 25.4 |
| 32K-30 concurrency | 32K | 1024 | 30 | 36535.3 | 14253.5 | 61.0% | 25.1 | 26.0 |
| 32K-50 concurrency | 32K | 1024 | 50 | 55251.4 | 16415.4 | 70.3% | 26.0 | 32.9 |
| 32K-100 concurrency | 32K | 1024 | 100 | 103109.2 | 21360.9 | 79.3% | 26.7 | 42.0 |
| 32K-200 concurrency | 32K | 1024 | 200 | 197632.2 | 42750.5 | 78.4% | 26.3 | 48.9 |
| 64K-30 concurrency | 64K | 1024 | 30 | 80304.3 | 13650.8 | 83.0% | 26.1 | 32.0 |
| 64K-50 concurrency | 64K | 1024 | 50 | 123199.1 | 18483.4 | 85.0% | 27.1 | 40.7 |
| 64K-100 concurrency | 64K | 1024 | 100 | 220035.0 | 29883.4 | 86.4% | 26.4 | 51.9 |
| 64K-200 concurrency | 64K | 1024 | 200 | 392632.1 | 72064.3 | 81.7% | 26.7 | 65.9 |
| 130K-512-20 concurrency | 130K | 512 | 20 | 132383.8 | 23612.2 | 82.2% | 24.3 | 42.0 |
| 130K-1024-30 concurrency | 130K | 1024 | 30 | 224131.7 | 31314.0 | 86.0% | 24.3 | 33.3 |

### FAQ
1. If intermittent crashes occur, apply this PR as a patch: https://github.com/vllm-project/vllm/pull/34079

2. GLM-5.1 may be memory-constrained on A2 servers. If tests show that enabling MMC degrades performance, lower `gpu-memory-utilization` to around 0.85.

3. For memory allocation details, see [DEVICE_RDMA memory allocation guide](https://gitcode.com/Ascend/memfabric_hybrid/wiki/DEVICE_RDMA%20%E5%86%85%E5%AD%98%E7%94%B3%E8%AF%B7.md).

