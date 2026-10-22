### <span style="color:#e60000;">This document is outdated as of 2026-07-14. See [A3 × 4 + GLM5.1](https://gitcode.com/Ascend/memcache/wiki/MMC%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E2%80%94GLM-5.1+A3-4%E6%9C%BAPD%E5%88%86%E7%A6%BB.md)</span>

### Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.

**GLM-5.1** is Zhipu's latest flagship model, with greatly enhanced coding ability and **long-horizon task** performance. It can work continuously and autonomously for up to 8 hours on a single task, completing the full cycle from planning and execution to iterative optimization and delivering engineering-grade results. Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.

### Test Results Summary

MemCache prefix-cache performance was tested across four scenarios on a four-node A3 cluster (8 × 910C per node; each 910C combines two 910B dies), using the GLM5.1 W8A8-quantized model and a PD-disaggregated architecture (2 Prefill × 4DP × 8TP + 2 Decode × 8DP × 4TP). MMC v1.1.1 was used, with 500 GB of storage per machine and 2 TB of total KV cache. Key results:

| Scenario | InputLen | OutputLen | Prefixes | Requests | Concurrency | Actual concurrency | RR | Cache | Average TTFT (ms) | TTFT reduction | Average TPOT (ms) | Throughput (token/s) | Throughput increase |
|------|----------|-----------|----------|----------|------|----------|----|----------|--------------|----------|--------------|---------------|----------|
| 64K × 20CC | 65536 | 1024 | 10 | 80 | 20 | 17.67 | 0 | No | 35,124 | / | 21.6 | 316.1 | / |
| 64K × 20CC | 65536 | 1024 | 10 | 80 | 20 | 18.89 | 0 | 90% | 5,445 | 84.5% | 23.9 | 647.5 | 104.8% |
| 64K × 30CC | 65536 | 1024 | 15 | 120 | 30 | 26.70 | 0 | No | 61,898 | / | 22.1 | 323.5 | / |
| 64K × 30CC | 65536 | 1024 | 15 | 120 | 30 | 27.01 | 0 | 90% | 9,114 | 85.3% | 25.5 | 786.6 | 143.2% |
| 128K × 10CC | 131072 | 1024 | 5 | 40 | 10 | 9.10 | 0 | No | 51,366 | / | 20.7 | 128.4 | / |
| 128K × 10CC | 131072 | 1024 | 5 | 40 | 10 | 9.25 | 0 | 90% | 12,369 | 75.9% | 20.9 | 280.4 | 118.4% |
| 128K × 20CC | 131072 | 1024 | 10 | 80 | 20 | 17.82 | 0 | No | 107,496 | / | 20.3 | 142.3 | / |
| 128K × 20CC | 131072 | 1024 | 10 | 80 | 20 | 17.90 | 0 | 90% | 39,251 | 63.5% | 20.9 | 294.9 | 107.2% |

Key findings:
1. **64K context:** Higher concurrency amplifies cache benefits; throughput improvement reaches 143.20% when concurrency rises from 20 to 30.
2. **At concurrency 20:** Increasing input length from 64K to 128K reduces the TTFT improvement from 84.5% to 63.5%, noticeably weakening the cache benefit.
3. **128K long context:** Cache benefits weaken as concurrency increases. Raising concurrency from 10 to 20 reduces both TTFT improvement and throughput gain. Due to system concurrency limits, the current DP4 setup supports only four concurrent requests; queueing offsets some gains from KV cache hits.



### Deployment

This deployment is adapted from the [official guide to deploying GLM-5.1 with vllm-ascend](https://docs.vllm.ai/projects/vllm-ascend-cn/zh-cn/latest/tutorials/models/GLM5.html).

**Hardware:** Four Atlas 900 A3 SuperPoD nodes, 64 cards total

**NPU driver firmware:** 25.5.1 or later

**L1 1520:** 1.5.0 or later

**CANN software:** CANN 8.5.1 or later

**Inference framework:** vLLM

**Deployment configuration:** 1P1D (Prefill TP8 DP4, Decode TP4 DP8)

```
torch                                    2.9.0+cpu
torch_npu                                2.9.0.post1+gitee7ba04
torchaudio                               2.9.0
torchvision                              0.24.0
transformers                             5.2.0
triton-ascend                            3.2.0.dev20260322
vllm                                     0.18.0+empty           /vllm-workspace/vllm
vllm_ascend                              0.18.0rc1              /vllm-workspace/vllm-ascend
```

**Apply the following patches to vllm and vllm-ascend:**

1. Fix MTP and KV pool compatibility: https://github.com/vllm-project/vllm-ascend/pull/7874
2. KV cache transfer stability: https://github.com/vllm-project/vllm-ascend/pull/8890 https://github.com/vllm-project/vllm-ascend/pull/8959 https://github.com/vllm-project/vllm-ascend/pull/9235 https://github.com/vllm-project/vllm-ascend/pull/7874
3. Metrics and GLM5 chat template: https://github.com/vllm-project/vllm/issues/39614
4. Fix intermittent crashes: https://github.com/vllm-project/vllm/pull/34079

#### 1. Install MF

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



#### 2. Edit the Configuration Files

MemCache consists of MetaService and LocalService. LocalService is integrated with the inference framework and starts with the inference process; it handles memory allocation, data reads/writes, and transfers. MetaService runs separately, connects to LocalService over TCP, and handles metadata management, data eviction, primary/standby switching, and other high-availability functions.

At startup, MetaService and LocalService load settings from separate configuration files. By default, these files are in the config directory of the MemCache wheel package.

```
/usr/local/lib/python3.11/site-packages/memcache_hybrid/config
```



###### MetaService Configuration File

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



###### LocalService Configuration File

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

## Provide 25 GB of pooled space per inference process; adjust according to the environment.
ock.mmc.local_service.dram.size = 25GB
```

For more configuration options, see the official documentation: https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md



#### 3. Prepare the Inference Service Launch Scripts

##### 3.1 Service Launch Helper Script

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



##### 3.2 vLLM Launch Script

Prepare the `run_dp_template.sh` script on each node.

###### P0

```shell
nic_name="br-provider" # change to your own nic name
local_ip="10.246.224.2" # change to your own ip

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
export VLLM_NIXL_ABORT_REQUEST_TIMEOUT=300

export ASCEND_RT_VISIBLE_DEVICES=$1

export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export VLLM_ASCEND_ENABLE_MLAPO=1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.11.14/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=180

vllm serve /model/ \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --speculative-config '{"num_speculative_tokens": 3, "method":"deepseek_mtp"}' \
    --profiler-config \
    '{"profiler": "torch", "torch_profiler_dir": "./vllm_profile", "torch_profiler_with_stack": false}' \
    --seed 1024 \
    --served-model-name glm-5 \
    --max-model-len 200704 \
    --additional-config '{"enable_npugraph_ex": true, "fuse_muls_add": true, "multistream_overlap_shared_expert": true, "recompute_scheduler_enable": false}' \
    --max-num-batched-tokens 4096 \
    --trust-remote-code \
    --max-num-seqs 64 \
    --quantization ascend \
    --async-scheduling \
    --enable-chunked-prefill \
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
                    "kv_port": "40000",
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
    }'
```



###### P1

```shell
nic_name="br-provider" # change to your own nic name
local_ip="10.246.224.3" # change to your own ip

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
export VLLM_NIXL_ABORT_REQUEST_TIMEOUT=300

export ASCEND_RT_VISIBLE_DEVICES=$1
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True

export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export VLLM_ASCEND_ENABLE_MLAPO=1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.11.14/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=180

vllm serve /model/ \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --speculative-config '{"num_speculative_tokens": 3, "method":"deepseek_mtp"}' \
    --profiler-config \
    '{"profiler": "torch", "torch_profiler_dir": "./vllm_profile", "torch_profiler_with_stack": false}' \
    --seed 1024 \
    --served-model-name glm-5 \
    --max-model-len 200704 \
    --additional-config '{"enable_npugraph_ex": true, "fuse_muls_add": true, "multistream_overlap_shared_expert": true, "recompute_scheduler_enable": false}' \
    --max-num-batched-tokens 4096 \
    --trust-remote-code \
    --max-num-seqs 64 \
    --gpu-memory-utilization 0.95 \
    --quantization ascend \
    --async-scheduling \
    --enable-chunked-prefill \
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
    }'
```



###### D0

```shell
nic_name="br-provider" # change to your own nic name
local_ip="10.246.224.15" # change to your own ip

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
export VLLM_NIXL_ABORT_REQUEST_TIMEOUT=300

export TASK_QUEUE_ENABLE=1

export ASCEND_RT_VISIBLE_DEVICES=$1

export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export VLLM_ASCEND_ENABLE_MLAPO=1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.11.14/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=180

vllm serve /model/ \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --speculative-config '{"num_speculative_tokens": 3,  "method":"deepseek_mtp"}' \
    --profiler-config \
    '{"profiler": "torch", "torch_profiler_dir": "./vllm_profile", "torch_profiler_with_stack": false}' \
    --seed 1024 \
    --served-model-name glm-5 \
    --max-model-len 200000 \
    --max-num-batched-tokens 32 \
    --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY", "cudagraph_capture_sizes":[4, 8, 12, 16,20,24,28, 32]}' \
    --additional-config '{"enable_npugraph_ex": true, "fuse_muls_add": true, "multistream_overlap_shared_expert": true, "recompute_scheduler_enable": false}' \
    --trust-remote-code \
    --max-num-seqs 8 \
    --gpu-memory-utilization 0.92 \
    --async-scheduling \
    --enable-chunked-prefill \
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
    }'
```



###### D1

```shell
nic_name="br-provider" # change to your own nic name
local_ip="10.246.224.17" # change to your own ip

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
export VLLM_NIXL_ABORT_REQUEST_TIMEOUT=300

export TASK_QUEUE_ENABLE=1

export ASCEND_RT_VISIBLE_DEVICES=$1

export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export VLLM_ASCEND_ENABLE_MLAPO=1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib

export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.11.14/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
export PYTHONHASHSEED=0
export VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT=180

vllm serve /model/ \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --speculative-config '{"num_speculative_tokens": 3,  "method":"deepseek_mtp"}' \
    --profiler-config \
    '{"profiler": "torch",
    "torch_profiler_dir": "./vllm_profile",
    "torch_profiler_with_stack": false}' \
    --seed 1024 \
    --served-model-name glm-5 \
    --max-model-len 200000 \
    --max-num-batched-tokens 32 \
    --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY", "cudagraph_capture_sizes":[4, 8, 12, 16,20,24,28, 32]}' \
    --additional-config '{"enable_npugraph_ex": true, "fuse_muls_add":true,"multistream_overlap_shared_expert":true,"recompute_scheduler_enable" : false}' \
    --trust-remote-code \
    --max-num-seqs 8 \
    --gpu-memory-utilization 0.92 \
    --async-scheduling \
    --enable-chunked-prefill \
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
    }'
```

**Adjust the following values for your environment:**

- `local_ip`: Local node IP address.
- `nic_name`: Local network interface name.
- Model weight path.



**Compared with the baseline script, enabling MF requires only these two changes:**

1. Add an environment variable pointing to the MemCache LocalService configuration file (the wheel package's default configuration path is shown here).

```
export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
```

2. `kv-transfer-config` now uses both MooncakeConnectorV1 and AscendStoreConnector instead of MooncakeConnectorV1 alone.

- MooncakeConnectorV1 transfers the KV Cache for each request from P to D.
- AscendStoreConnector calls the MemCache read/write interfaces to enable prefix-cache reuse across requests.



##### 3.3 Request Forwarding Script

Get the proxy program [load_balance_proxy_server_example.py](https://github.com/vllm-project/vllm-ascend/blob/main/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py) from the vllm-ascend repository.

Start the forwarding service with the following script (update the IP addresses and ports to match the services).

```shell
unset http_proxy
unset https_proxy

python load_balance_proxy_server_example.py \
    --port 8000 \
    --host 10.246.224.16 \
    --prefiller-hosts \
        10.246.224.2 \
        10.246.224.2 \
        10.246.224.3 \
        10.246.224.3 \
    --prefiller-ports \
        6700  6701 \
        6700  6701 \
    --decoder-hosts \
        10.246.224.15 \
        10.246.224.15 \
        10.246.224.15 \
        10.246.224.15 \
        10.246.224.17 \
        10.246.224.17 \
        10.246.224.17 \
        10.246.224.17 \
    --decoder-ports \
        6721 6722 6723 6724 \
        6721 6722 6723 6724
```



#### 4. Start the Services

##### 4.1 Start MetaService

Start the MetaService metadata management service on any node.

```shell
export MMC_META_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```

##### 4.2 Start the Inference Service

Then start the inference service on each node with the following command.

###### P0

```shell
python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 2 --dp-rank-start 0 --dp-address 10.246.224.2 --dp-rpc-port 63521 --vllm-start-port 6700
```

###### P1

```shell
python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 2 --dp-rank-start 2 --dp-address 10.246.224.2 --dp-rpc-port 63521 --vllm-start-port 6700
```

###### D0

```shell
python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 4 --dp-rank-start 0 --dp-address 10.246.224.15 --dp-rpc-port 63523 --vllm-start-port 6721
```

###### D1

```shell
python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 4 --dp-rank-start 4 --dp-address 10.246.224.15 --dp-rpc-port 63523 --vllm-start-port 6721
```

Finally, start the request forwarding script.

```shell
bash start_proxy.sh
```


### FAQ
1. If a memory allocation error occurs (`Try HalMemCreate ret: -6`), see the [MemCache+vLLM+A3 disaggregated deployment guide](MemCache-vLLM-A3-Standalone-Deployment-Case-Study.md).

