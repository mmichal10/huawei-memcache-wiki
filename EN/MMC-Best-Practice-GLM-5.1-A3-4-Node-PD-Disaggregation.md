### Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.

**GLM-5.1** is Zhipu's latest flagship model, with greatly enhanced coding ability and **long-horizon task** performance. It can work continuously and autonomously for up to 8 hours on a single task, completing the full cycle from planning and execution to iterative optimization and delivering engineering-grade results. Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.


### Test Results Summary
**Multi-turn conversation dataset test**

| | MF | Turns | Input per turn | Output per turn | Concurrency | Total sessions | mean ttft (ms) | mean tpot (ms) | Total time (s) | HBM hit rate | DRAM hit rate |
|-----|---------|----|------|------|----|------|-----------------|----------------|----------|--------|---------|
| Group 1 | disable | 20 | 9216 | 1024 | 16 | 16 | 48166.97 | 8.61 | 594.513 | 2.8% | / |
| Group 1 | enable | 20 | 9216 | 1024 | 16 | 16 | 27450.59 (↓43%) | 9.92 | 377.931 | 1.6% | 68.6% |
| Group 2 | disable | 40 | 4096 | 1024 | 16 | 16 | 37498.54 | 15.53 | 1070.425 | 4.0% | / |
| Group 2 | enable | 40 | 4096 | 1024 | 16 | 16 | 17744.16 (↓52%) | 17.33 | 710.006 | 4.2% | 82.3% |
| Group 3 | disable | 20 | 9216 | 1024 | 30 | 30 | 90407.57 | 7.56 | 1048.888 | 0.1% | / |
| Group 3 | enable | 20 | 9216 | 1024 | 30 | 30 | 41756.93 (↓54%) | 8.91 | 519.047 | 0.2% | 69.2% |

**90% fixed prefix reuse test**

MemCache prefix-cache performance was tested across four scenarios on a four-node A3 cluster (8 × 910C per node; each 910C combines two 910B dies), using the GLM5.1 W8A8-quantized model and a PD-disaggregated architecture (2 Prefill × 4DP × 8TP + 2 Decode × 8DP × 4TP).

| Scenario | Input | Concurrency (actual CC) | TTFT without cache (ms) | TTFT with cache (ms) | TTFT reduction |
|----|-------|----------|-------------|-------------|--------|
| 1  | 64k   | 16       | 71723       | 14407       | 79.91% |
| 2  | 128k  | 13       | 83286       | 27883       | 66.52% |
| 3  | 64k   | 24       | 77366       | 8931        | 88.45% |
| 4  | 128k  | 21       | 120392      | 38678       | 67.87% |

**Conclusion**
1. **TTFT drops substantially:** TTFT falls by ~80.6% in all scenarios, reducing first-token response time from minutes to seconds.
2. **Actual concurrency increases significantly:** After cache hits, actual model-side concurrency rises from 6 to 24, doubling service capacity.


### Environment

**Hardware:** Four Atlas 900 A3 SuperPoD nodes, 64 cards total

**Deployment configuration:** 2P2D (Prefill TP8 DP4, Decode TP4 DP8)

<span style="color:#e60000;">Note: This deployment requires the following driver and software versions.</span>

**OS kernel version:** >=5.10.0

**NPU driver firmware:** >=25.5.1

【L1 1520 CC】>=1.5.0


### Deployment

This deployment is adapted from the [official guide to deploying GLM-5.1 with vllm-ascend](https://docs.vllm.ai/projects/vllm-ascend-cn/zh-cn/latest/tutorials/models/GLM5.html).



#### 1. Pull the official vllm-ascend image
```shell
docker pull quay.io/ascend/vllm-ascend:v0.18.0-a3
```
<span style="color:#e60000;">**A later vllm-ascend version can be used. The version shown was current when this example was released and is not mandatory; choose an appropriate release for testing.**</span>

For the container launch command, see the [official guide to deploying GLM-5.1 with vllm-ascend](https://docs.vllm.ai/projects/vllm-ascend-cn/zh-cn/latest/tutorials/models/GLM5.html).


#### 2. Install MF
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

Other unreleased versions can be installed by building from source; see the official documentation.

[memcache/docs/zh/install_run.md](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/install_run.md)



#### 3. Edit the Configuration Files

MemCache consists of MetaService and LocalService. LocalService handles memory allocation, data reads/writes, and transfers. MetaService runs separately, connects to LocalService over TCP, and handles metadata management, data eviction, primary/standby switching, and other high-availability functions.

At startup, MetaService and LocalService load settings from separate configuration files. By default, these files are in the config directory of the MemCache wheel package (run `pip show memcache_hybrid` to find the path).

```
/usr/local/lib/python3.11/site-packages/memcache_hybrid/config
```

**This document uses a typical A3 configuration with disaggregated deployment. Each node starts a standalone LocalService to allocate memory, while the LocalService started with the inference process only reads and writes data and does not provide memory.
For details, see the [MemCache disaggregated deployment guide](MemCache-vLLM-A3-Standalone-Deployment-Case-Study.md).**


###### Prepare the MetaService Configuration File

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
```


###### Prepare the LocalService Configuration File That Does Not Provide Memory

Replace the IP in the following configuration with the MetaService node IP, then save it as `mmc-local.conf` on each node.

```ini
## Set to the IP of the node running MetaService.
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## Set to the IP of the node running MetaService.
ock.mmc.local_service.config_store_url = tcp://127.0.0.1:6000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

## SDMA transfer within the supernode is typical for A3.
ock.mmc.local_service.protocol = device_sdma

## The LocalService started with the inference process does not allocate memory.
ock.mmc.local_service.dram.size = 0GB

ock.mmc.local_service.max.dram.size = 1024GB
```


###### Prepare the LocalService Configuration File That Provides Memory

Run the [mem_scan.py](https://gitcode.com/Ascend/memfabric_hybrid/blob/master/src/smem/python/memfabric_hybrid/memfabric_hybrid/mem_scan.py) script to scan available physical memory:

`python3 mem_scan.py`

![image.png](https://raw.gitcode.com/user-images/assets/7672916/348c0854-4efb-4803-8222-f6bf19589fa1/image.png 'image.png')

An A3 server with 2 TB of memory typically has more than 600 GB available for MemCache. If the scan reports less, clear the file cache with the following command.

<span style="color:#e60000;">!!! Warning !!! This command clears the cache. Weights and other cached files must be reloaded from the filesystem, so restarting the vLLM framework will take longer. Understand the implications before running it.</span>
```shell
echo 3 > /proc/sys/vm/drop_caches
```

If the scan still reports little available memory, fragmentation may be the cause. Run the following command to trigger memory compaction:

```shell
echo 1 /proc/sys/vm/compact_memory
```

If the scan still reports little available memory, restart the environment.

Assuming the previous scan found at least 600 GB available, replace the IP in the following configuration with the MetaService node IP and save it as `mmc-local-standalone.conf` on each node.
```ini
## Set to the IP of the node running MetaService.
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## Set to the IP of the node running MetaService.
ock.mmc.local_service.config_store_url = tcp://127.0.0.1:6000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

## SDMA transfer within the supernode is typical for A3.
ock.mmc.local_service.protocol = device_sdma

## The standalone LocalService provides 600 GB of memory.
ock.mmc.local_service.dram.size = 600GB

ock.mmc.local_service.max.dram.size = 1024GB
```

For more configuration options, see the official documentation: https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md


#### 4. Prepare the Inference Service Launch Scripts

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

Prepare the `run_dp_template.sh` script on each node.

###### P0

```shell
nic_name="enp194s0f0" # change to your own nic name
local_ip="141.61.39.233" # change to your own ip

export HCCL_OP_EXPANSION_MODE="AIV"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

export OMP_PROC_BIND=false
export OMP_NUM_THREADS=1
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export HCCL_BUFFSIZE=256

export ASCEND_AGGREGATE_ENABLE=1
export ASCEND_TRANSPORT_PRINT=1
export ACL_OP_INIT_MODE=1
export ASCEND_A3_ENABLE=1
# Timeout (in seconds) for automatically releasing the prefiller’s KV cache for a particular request.
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480

export ASCEND_RT_VISIBLE_DEVICES=$1
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0

vllm serve /home/weights/GLM5.1-W8A8 \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --speculative-config '{"num_speculative_tokens": 3, "method":"deepseek_mtp"}' \
    --profiler-config '{"profiler": "torch", "torch_profiler_dir": "./vllm_profile", "torch_profiler_with_stack": false}' \
    --seed 1024 \
    --served-model-name glm-5 \
    --max-model-len 200704 \
    --additional-config '{"fuse_muls_add": true, "multistream_overlap_shared_expert": true, "ascend_compilation_config": {"enable_npugraph_ex": true}, "layer_sharding": ["q_b_proj", "o_proj"]}' \
    --max-num-batched-tokens 4096 \
    --trust-remote-code \
    --max-num-seqs 64 \
    --async-scheduling \
    --enable-chunked-prefill \
    --quantization ascend \
    --gpu-memory-utilization 0.95 \
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
    }' 2>&1 | tee output.log
```



###### P1

```shell
nic_name="enp48s3u1u1" # change to your own nic name
local_ip="141.61.39.235" # change to your own ip

export HCCL_OP_EXPANSION_MODE="AIV"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

export OMP_PROC_BIND=false
export OMP_NUM_THREADS=1
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export HCCL_BUFFSIZE=256

export ASCEND_AGGREGATE_ENABLE=1
export ASCEND_TRANSPORT_PRINT=1
export ACL_OP_INIT_MODE=1
export ASCEND_A3_ENABLE=1
# Timeout (in seconds) for automatically releasing the prefiller’s KV cache for a particular request.
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480

export ASCEND_RT_VISIBLE_DEVICES=$1
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True

export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0

vllm serve /home/weights/GLM5.1-W8A8 \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --speculative-config '{"num_speculative_tokens": 3, "method":"deepseek_mtp"}' \
    --profiler-config '{"profiler": "torch", "torch_profiler_dir": "./vllm_profile", "torch_profiler_with_stack": false}' \
    --seed 1024 \
    --served-model-name glm-5 \
    --max-model-len 200704 \
    --additional-config '{"fuse_muls_add": true, "multistream_overlap_shared_expert": true, "ascend_compilation_config": {"enable_npugraph_ex": true}, "layer_sharding": ["q_b_proj", "o_proj"]}' \
    --max-num-batched-tokens 4096 \
    --trust-remote-code \
    --max-num-seqs 64 \
    --async-scheduling \
    --enable-chunked-prefill \
    --gpu-memory-utilization 0.95 \
    --quantization ascend \
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
                        "lookup_rpc_port": "1"
                    }
                }
            ]
        }
    }' 2>&1 | tee output.log
```



###### D0

```shell
nic_name="enx9c69d302197d" # change to your own nic name
local_ip="141.61.39.237" # change to your own ip

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


export ASCEND_AGGREGATE_ENABLE=1
export ASCEND_TRANSPORT_PRINT=1
export ACL_OP_INIT_MODE=1
export ASCEND_A3_ENABLE=1
# Timeout (in seconds) for automatically releasing the prefiller’s KV cache for a particular request.
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480

export TASK_QUEUE_ENABLE=1

export ASCEND_RT_VISIBLE_DEVICES=$1

export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export VLLM_ASCEND_ENABLE_MLAPO=1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0

vllm serve /home/weights/GLM5.1-W8A8 \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --speculative-config '{"num_speculative_tokens": 3,  "method":"deepseek_mtp"}' \
    --profiler-config '{"profiler": "torch", "torch_profiler_dir": "./vllm_profile", "torch_profiler_with_stack": false}' \
    --seed 1024 \
    --served-model-name glm-5 \
    --max-model-len 200704 \
    --max-num-batched-tokens 64 \
    --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY", "cudagraph_capture_sizes": [4,8,12,16,20,24,28,32]}' \
    --additional-config '{"fuse_muls_add": true, "multistream_overlap_shared_expert": true, "ascend_compilation_config": {"enable_npugraph_ex": true}}' \
    --trust-remote-code \
    --max-num-seqs 16 \
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
                        "lookup_rpc_port": "2"
                    }
                }
            ]
        }
    }' 2>&1 | tee output.log
```



###### D1

```shell
nic_name="enx9c69d3020bab" # change to your own nic name
local_ip="141.61.39.238" # change to your own ip

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

export ASCEND_AGGREGATE_ENABLE=1
export ASCEND_TRANSPORT_PRINT=1
export ACL_OP_INIT_MODE=1
export ASCEND_A3_ENABLE=1
# Timeout (in seconds) for automatically releasing the prefiller’s KV cache for a particular request.
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=480

export TASK_QUEUE_ENABLE=1

export ASCEND_RT_VISIBLE_DEVICES=$1

export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export VLLM_ASCEND_ENABLE_MLAPO=1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0

vllm serve /home/weights/GLM5.1-W8A8 \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --speculative-config '{"num_speculative_tokens": 3,  "method":"deepseek_mtp"}' \
    --profiler-config '{"profiler": "torch", "torch_profiler_dir": "./vllm_profile", "torch_profiler_with_stack": false}' \
    --seed 1024 \
    --served-model-name glm-5 \
    --max-model-len 200704 \
    --max-num-batched-tokens 64 \
    --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY", "cudagraph_capture_sizes": [4,8,12,16,20,24,28,32]}' \
    --additional-config '{"fuse_muls_add": true, "multistream_overlap_shared_expert": true, "ascend_compilation_config": {"enable_npugraph_ex": true}}' \
    --trust-remote-code \
    --max-num-seqs 16 \
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
                        "lookup_rpc_port": "3"
                    }
                }
            ]
        }
    }' 2>&1 | tee output.log
```

**Adjust the following values for your environment:**

- `local_ip`: Local node IP address.
- `nic_name`: Local network interface name.
- Model weight path.



**Compared with the baseline script, enabling MF requires only these two changes:**

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

node_p0_ip=141.61.39.233
node_p1_ip=141.61.39.235
node_d0_ip=141.61.39.237
node_d1_ip=141.61.39.238

python load_balance_proxy_server_example.py \
    --port 8000 \
    --host 0.0.0.0 \
    --prefiller-hosts \
        $node_p0_ip \
        $node_p1_ip \
    --prefiller-ports \
        6700 \
        6700 \
    --decoder-hosts \
        $node_d0_ip \
        $node_d0_ip \
        $node_d0_ip \
        $node_d0_ip \
        $node_d1_ip \
        $node_d1_ip \
        $node_d1_ip \
        $node_d1_ip \
    --decoder-ports \
        6721 6722 6723 6724 \
        6721 6722 6723 6724
```



#### 5. Start the Services

##### 5.1 Start MetaService

Start the MetaService metadata management service on the node selected earlier.

```shell
export MMC_META_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```

##### 5.2 Start the Standalone MemCache Process
Prepare the following script on each node, named `memcache_standalone.py`.

```python
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

Start the standalone MemCache process on each node with:
```shell
export MMC_LOCAL_CONFIG_PATH="mmc-local-standalone.conf"
python3 memcache_standalone.py
```
**Wait until all nodes start successfully (the log shows `Successfully initialized memcache.`) before proceeding.**

##### 5.3 Start the Inference Service

Start the inference service on each node with the following command.

###### P0

```shell
python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 2 --dp-rank-start 0 --dp-address 141.61.39.233 --dp-rpc-port 10521 --vllm-start-port 6700
```

###### P1

```shell
python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 2 --dp-rank-start 2 --dp-address 141.61.39.233 --dp-rpc-port 10521 --vllm-start-port 6700
```

###### D0

```shell
python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 4 --dp-rank-start 0 --dp-address 141.61.39.237 --dp-rpc-port 10523 --vllm-start-port 6721
```

###### D1

```shell
python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 4 --dp-rank-start 4 --dp-address 141.61.39.237 --dp-rpc-port 10523 --vllm-start-port 6721
```

Finally, start the request forwarding script.

```shell
bash start_proxy.sh
```



### Testing

The recommended tool is the official vLLM multi-turn conversation benchmark: [vllm/benchmarks/multi_turn/README.md at main · vllm-project/vllm](https://github.com/vllm-project/vllm/blob/main/benchmarks/multi_turn/README.md)

In `generate_multi_turn.json`, configure key parameters such as the number of conversations, number of turns, and input/output tokens per turn. The first data set below is used as an example.

```json
{
    "filetype": "generate_conversations",
    "num_conversations": 16,
    "text_files": ["pg1184.txt"],
    "print_stats": false,
    "prompt_input": {
        "num_turns": {
            "distribution": "constant",
            "value": 20
        },
        "prefix_num_tokens": {
            "distribution": "constant",
            "value": 10240
        },
        "num_tokens": {
            "distribution": "constant",
            "value": 9216
        }
    },
    "prompt_output": {
        "num_tokens": {
            "distribution": "constant",
            "value": 1024
        }
    }
}

```

Start the test with the following command:

```shell
python benchmark_serving_multi_turn.py \
    --model /home/weights/GLM5.1-W8A8 \
    --served-model-name glm-5 \
    --input-file generate_multi_turn.json \
    --num-clients 16 \                       # Concurrency
    --max-active-conversations 16 \
    --no-early-stop \
    --request-timeout-sec 600 \
    -u http://<ip>:<port>                    # IP and port of the proxy script
```



**Test results**

| | MF | Turns | Input per turn | Output per turn | Concurrency | Total sessions | mean ttft (ms) | mean tpot (ms) | Total time (s) | HBM hit rate | DRAM hit rate |
| ------ | ------- | ---- | -------- | -------- | ---- | -------- | --------------- | -------------- | ---------- | --------- | ---------- |
| Group 1 | disable | 20 | 9216 | 1024 | 16 | 16 | 48166.97 | 8.61 | 594.513 | 2.8% | / |
| Group 1 | enable | 20 | 9216 | 1024 | 16 | 16 | 27450.59 (↓43%) | 9.92 | 377.931 | 1.6% | 68.6% |
| Group 2 | disable | 40 | 4096 | 1024 | 16 | 16 | 37498.54 | 15.53 | 1070.425 | 4.0% | / |
| Group 2 | enable | 40 | 4096 | 1024 | 16 | 16 | 17744.16 (↓52%) | 17.33 | 710.006 | 4.2% | 82.3% |
| Group 3 | disable | 20 | 9216 | 1024 | 30 | 30 | 90407.57 | 7.56 | 1048.888 | 0.1% | / |
| Group 3 | enable | 20 | 9216 | 1024 | 30 | 30 | 41756.93 (↓54%) | 8.91 | 519.047 | 0.2% | 69.2% |
