### Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.

DeepSeek-V4 supports million-character long contexts and leads domestic and open-source models in agent capabilities, world knowledge, and reasoning performance. Compared with DeepSeek-V4-Pro, DeepSeek-V4-Flash has somewhat less world knowledge but delivers similar reasoning capability. Its smaller model and activation sizes allow V4-Flash to provide faster, more economical API services.
Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.



### Environment

**Model weights:** [DeepSeek-V4-Flash-w8a8-mtp](https://www.modelscope.cn/models/Eco-Tech/DeepSeek-V4-Flash-w8a8-mtp)

**Note: MemCache's preferred A3 solution (multi-node pooling over HCCS) requires specific switch CC packages, NPU driver firmware, and OS kernels, which may be difficult to upgrade at some customer sites. This example uses the alternative MemCache solution (single-node L2 pooling over PCIe) to avoid low-level software compatibility issues.**

**OS kernel version:** No special requirements

**NPU driver firmware:** No special requirements

**L1 1520 CC:** No special requirements



### Test Results

**Hardware:** Four Atlas 900 A3 SuperPoD nodes, 64 cards

**Test tool:** aisbench

![memcache_ttft_compare.png](https://raw.gitcode.com/user-images/assets/7672915/919887de-21f6-43e2-8e46-ea8f34f2687f/memcache_ttft_compare.png 'memcache_ttft_compare.png')



### Deployment

#### 1. Pull the official vllm-ascend image

Pull the official vllm-ascend image.

```shell
docker pull quay.io/ascend/vllm-ascend:v0.23.0rc1
```

<span style="color:#e60000;">**Note:** The current v0.23.0rc1 release of vllm-ascend has a known compatibility issue when MemCache is enabled as the memory-pooling backend on A3.</span> Modify the following code in the image or apply the corresponding Git patch to work around the issue. It has been fixed in the vllm-ascend repository and is expected to be included in the next release.

```diff
--- a/vllm_ascend/distributed/kv_transfer/kv_pool/ascend_store/pool_worker.py
+++ b/vllm_ascend/distributed/kv_transfer/kv_pool/ascend_store/pool_worker.py
@@ -737,8 +737,6 @@ class KVPoolWorker:
         )
         # Initialize store, register buffers, and start transfer threads
         # directly here (like main) — no separate init_backend handshake.
-        if hasattr(self.m_store, "init_store"):
-            self.m_store.init_store()
         self.m_store.register_buffer(ptrs, lengths)
         self._start_kv_transfer_threads()
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

Download links:

[memfabric](https://pypi.org/project/memfabric-hybrid/#history)

[memcache](https://pypi.org/project/memcache-hybrid/#history)

Other unreleased versions can be installed by building from source; see the [official build guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/install_run.md).



#### 3. Prepare the MemCache Configuration Files

MemCache consists of MetaService and LocalService. LocalService handles memory allocation, data reads/writes, and transfers. MetaService runs separately, connects to LocalService over TCP, and handles metadata management, data eviction, primary/standby switching, and other high-availability functions.

At startup, MetaService and LocalService load settings from separate configuration files. By default, these files are in the config directory of the MemCache wheel package (run `pip show memcache_hybrid` to find the path).

```
/usr/local/lib/python3.11/site-packages/memcache_hybrid/config
```

**This example uses a separate memory pool on each node, so MetaService must be started on every node.**



###### Prepare the MetaService Configuration File

Prepare `mmc-meta.conf` on all nodes.

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



##### LocalService configuration file

Prepare an `mmc-local.conf` file on each node and adjust memory capacity for your environment.

```ini
## Set to the IP of the node running MetaService.
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## Set to the IP of the node running MetaService.
ock.mmc.local_service.config_store_url = tcp://127.0.0.1:6000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

## Use the shared-memory pooling scheme.
ock.mmc.local_service.protocol = host_shm

## Configure 24 GB per card; adjust according to the actual shared-memory capacity in the environment (check with `df -h`).
ock.mmc.local_service.dram.size = 24GB

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md)
```

#### 4. Prepare the Inference Service Launch Scripts
##### 4.1 Service Launch Helper Script

Prepare the `launch_online_dp.py` script on each node.

```python
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

###### P node
```shell
unset ftp_proxy
unset https_proxy
unset http_proxy

nic_name="xxxx" # Replace with the actual network interface name.
local_ip=xx.xx.xx.1 # Replace with the actual network interface IP.

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

export VLLM_RPC_TIMEOUT=3600000
export VLLM_EXECUTE_MODEL_TIMEOUT_SECONDS=30000
export HCCL_EXEC_TIMEOUT=204
export HCCL_CONNECT_TIMEOUT=120

export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export HCCL_BUFFSIZE=2560
export TASK_QUEUE_ENABLE=1

export VLLM_ASCEND_ENABLE_FLASHCOMM1=1
export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export HCCL_OP_EXPANSION_MODE="AIV"
export LD_PRELOAD=/usr/lib/aarch64-linux-gnu/libjemalloc.so.2:$LD_PRELOAD
export ASCEND_RT_VISIBLE_DEVICES=$1

export PYTHONHASHSEED=0 
# Replace with the actual configuration file path; this is an example.
export MMC_LOCAL_CONFIG_PATH=/xxx/mmc-local.conf
# This environment variable can reduce some KV Cache reads and writes and may improve performance in certain scenarios, but it can also affect prefix hit rate. Benchmark and set it as needed.
# Leaving it unset is the default behavior. Setting it to 0 is the most aggressive policy. Other values can be multiples of the minimum number of tokens that can hit; for DSV4 with block_size=128, use a multiple of 16384.
# For vLLM environment variables, see https://docs.vllm.com.cn/en/latest/configuration/env_vars/?h=environment
export VLLM_PREFIX_CACHE_RETENTION_INTERVAL=0

# Replace with the actual model weight path; this is an example.
vllm serve /root/.cache/modelscope/hub/models/vllm-ascend/DeepSeek-V4-Flash-w8a8-mtp \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --seed 1024 \
    --served-model-name dsv4 \
    --max-model-len 1048576 \
    --max-num-batched-tokens 8192 \
    --max-num-seqs 16 \
    --no-disable-hybrid-kv-cache-manager \
    --model-loader-extra-config='{"enable_multithread_load": "true", "num_threads": 128}' \
    --safetensors-load-strategy 'prefetch' \
    --speculative-config '{"num_speculative_tokens": 1,"method": "mtp","enforce_eager": true}' \
    --trust-remote-code \
    --block-size 128 \
    --tokenizer-mode deepseek_v4 \
    --tool-call-parser deepseek_v4 \
    --enable-auto-tool-choice \
    --reasoning-parser deepseek_v4 \
    --gpu-memory-utilization 0.85 \
    --quantization ascend \
    --enforce-eager \
    --additional-config '{"enable_cpu_binding": true, "enable_shared_expert_dp": true,  "enable_dsa_cp": true}' \
    --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_producer",
        "kv_connector_extra_config": {
            "connectors": [
                {
                    "kv_connector": "MooncakeHybridConnector",
                    "kv_role": "kv_producer",
                    "kv_port": "30000",
                    "kv_connector_extra_config": {
                        "prefill": {
                            "dp_size": 4,
                            "tp_size": 4
                        },
                        "decode": {
                            "dp_size": 16,
                            "tp_size": 1
                        }
                    }
                },
                {
                    "kv_connector": "AscendStoreConnector",
                    "kv_role": "kv_producer",
                    "kv_connector_extra_config": {
                        "lookup_rpc_port":"0",
                        "backend": "memcache"
                    }
                }
            ]
        }
    }' 2>&1 | tee pd_p1.log
```

###### D node
```shell
unset ftp_proxy
unset https_proxy
unset http_proxy

nic_name="xxxx" # Replace with the actual network interface name.
local_ip=xx.xx.xx.2 # Replace with the actual network interface IP.

export LD_PRELOAD=/usr/lib/aarch64-linux-gnu/libjemalloc.so.2:$LD_PRELOAD
export HCCL_OP_EXPANSION_MODE="AIV"
export TASK_QUEUE_ENABLE=1

export VLLM_RPC_TIMEOUT=3600000
export VLLM_EXECUTE_MODEL_TIMEOUT_SECONDS=30000
export HCCL_EXEC_TIMEOUT=204
export HCCL_CONNECT_TIMEOUT=1200

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name

export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export HCCL_BUFFSIZE=1024

export ASCEND_RT_VISIBLE_DEVICES=$1
export PYTHONHASHSEED=0 
# Replace with the actual configuration file path; this is an example.
export MMC_LOCAL_CONFIG_PATH=/xxx/mmc-local.conf
# This environment variable can reduce some KV Cache reads and writes and may improve performance in certain scenarios, but it can also affect prefix hit rate. Benchmark and set it as needed.
# Leaving it unset is the default behavior. Setting it to 0 is the most aggressive policy. Other values can be multiples of the minimum number of tokens that can hit; for DSV4 with block_size=128, use a multiple of 16384.
# For vLLM environment variables, see https://docs.vllm.com.cn/en/latest/configuration/env_vars/?h=environment
export VLLM_PREFIX_CACHE_RETENTION_INTERVAL=0

# Replace with the actual model weight path; this is an example.
vllm serve /root/.cache/modelscope/hub/models/vllm-ascend/DeepSeek-V4-Flash-w8a8-mtp \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --seed 1024 \
    --served-model-name dsv4 \
    --max-model-len 1048576 \
    --max-num-batched-tokens 120 \
    --max-num-seqs 60 \
    --async-scheduling \
    --block-size 128 \
    --no-disable-hybrid-kv-cache-manager \
    --safetensors-load-strategy 'prefetch' \
    --trust-remote-code \
    --tokenizer-mode deepseek_v4 \
    --model-loader-extra-config='{"enable_multithread_load": "true", "num_threads": 128}' \
    --tool-call-parser deepseek_v4 \
    --enable-auto-tool-choice \
    --reasoning-parser deepseek_v4 \
    --gpu-memory-utilization 0.9 \
    --quantization ascend \
    --speculative-config '{"num_speculative_tokens": 1,"method": "mtp","enforce_eager": true}' \
    --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY"}' \
    --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_consumer",
        "kv_connector_extra_config": {
            "connectors": [
                {
                    "kv_connector": "MooncakeHybridConnector",
                    "kv_role": "kv_consumer",
                    "kv_port": "30100",
                    "kv_connector_extra_config": {
                        "prefill": {
                            "dp_size": 4,
                            "tp_size": 4
                        },
                        "decode": {
                            "dp_size": 16,
                            "tp_size": 1
                        }
                    }
                },
                {
                    "kv_connector": "AscendStoreConnector",
                    "kv_role": "kv_consumer",
                    "kv_connector_extra_config": {
                        "lookup_rpc_port":"0",
                        "backend": "memcache"
                    }
                }
            ]
        }
    }' \
    --additional-config '{
        "ascend_compilation_config":{
            "enable_npugraph_ex":true,
            "enable_static_kernel":false
        },
        "enable_cpu_binding":true,
        "multistream_overlap_shared_expert":true,
        "recompute_scheduler_enable":true
    }' 2>&1 | tee pd_d.log
```

##### 4.3 Request Forwarding Script
Prepare the following file on any node:

Get the proxy program [load_balance_proxy_server_example.py](https://github.com/vllm-project/vllm-ascend/blob/main/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py) from the vllm-ascend repository.

Then prepare a shell script to start this Python script. The example uses the name `start_proxy.sh`; configure IP addresses and ports according to the number of instances and parallelism in your environment.

```shell
unset http_proxy 
unset https_proxy 
python load_balance_proxy_server_example.py \
  --port 8025 \
  --host <proxy_ip> \
  --prefiller-hosts \
    <p_ip> \
    <p_ip> \
    <p_ip> \
    <p_ip> \
  --prefiller-ports \
    7100 7101 7102 7103 \
  --decoder-hosts \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
    <d_ip> \
  --decoder-ports \
    7100 7101 7102 7103 7104 7105 7106 7107 7108 7109 7110 7111 7112 7113 7114 7115 \
```

#### 5. Start the Services
##### 5.1 Start MetaService
Start the MetaService metadata management service on **all** nodes, using the actual configuration file path.
```shell
export MMC_META_CONFIG_PATH=/xxx/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```

##### 5.2 Start the vLLM Inference Service
Start the service on each node with commands similar to the following.

P node:
```shell
python launch_online_dp.py --dp-size 4 --tp-size 4 --dp-size-local 4 --dp-rank-start 0 --dp-address <p_ip> --dp-rpc-port 12321 --vllm-start-port 7100
```

D node:
```shell
python launch_online_dp.py --dp-size 16 --tp-size 1 --dp-size-local 16 --dp-rank-start 0 --dp-address <d_ip> --dp-rpc-port 12321 --vllm-start-port 7100
```

##### 5.3 Start the Request Forwarding Script
```shell
bash start_proxy.sh
```


### Detailed Results

Baseline (pooling disabled)

| Input | Output | Concurrency | Total requests | Prefixes | Prefix reuse rate | TTFT(ms) | TPOT(ms) |
| ---- | ---- | ---- | -------- | ------ | ---------- | -------- | -------- |
| 64K  | 1024 | 80   | 320      | 80     | 30%        | 16835.4  | 25.4     |
| 64K  | 1024 | 80   | 320      | 80     | 50%        | 15627.7  | 25.8     |
| 64K  | 1024 | 100  | 400      | 100    | 30%        | 24489.6  | 25.1     |
| 64K  | 1024 | 100  | 400      | 100    | 50%        | 24474.3  | 25.8     |
|      |      |      |          |        |            |          |          |
| 128K | 1024 | 80   | 320      | 80     | 30%        | 57755.3  | 24.0     |
| 128K | 1024 | 80   | 320      | 80     | 50%        | 56368.8  | 23.9     |
| 128K | 1024 | 100  | 400      | 100    | 30%        | 75295.0  | 23.9     |
| 128K | 1024 | 100  | 400      | 100    | 50%        | 75377.0  | 23.8     |



MemCache enabled

| Input | Output | Concurrency | Total requests | Prefixes | Prefix reuse rate | TTFT(ms) | TPOT(ms) |
| ---- | ---- | ---- | -------- | ------ | ---------- | -------- | -------- |
| 64K  | 1024 | 80   | 320      | 80     | 30%        | 15672.2  | 25.5     |
| 64K  | 1024 | 80   | 320      | 80     | 50%        | 9988.6   | 26.6     |
| 64K  | 1024 | 80   | 320      | 80     | 70%        | 9625.7   | 26.3     |
| 64K  | 1024 | 80   | 320      | 80     | 90%        | 6499.8   | 26.8     |
| 64K  | 1024 | 100  | 400      | 100    | 30%        | 23302.1  | 25.8     |
| 64K  | 1024 | 100  | 400      | 100    | 50%        | 11167.7  | 27.2     |
| 64K  | 1024 | 100  | 400      | 100    | 70%        | 11055.9  | 27.5     |
| 64K  | 1024 | 100  | 400      | 100    | 90%        | 7008.4   | 27.5     |
| 64K  | 1024 | 120  | 480      | 120    | 30%        | 30834.6  | 25.6     |
| 64K  | 1024 | 120  | 480      | 120    | 50%        | 16037.9  | 27.3     |
| 64K  | 1024 | 120  | 480      | 120    | 70%        | 15837.2  | 27.3     |
| 64K  | 1024 | 120  | 480      | 120    | 90%        | 7739.3   | 27.8     |
| 64K  | 1024 | 140  | 560      | 140    | 30%        | 39402.4  | 25.5     |
| 64K  | 1024 | 140  | 560      | 140    | 50%        | 21184.3  | 27.5     |
| 64K  | 1024 | 140  | 560      | 140    | 70%        | 27952.3  | 26.9     |
| 64K  | 1024 | 140  | 560      | 140    | 90%        | 8419.5   | 28.2     |
|      |      |      |          |        |            |          |          |
| 128K | 1024 | 80   | 320      | 80     | 30%        | 63597.40 | 23.70    |
| 128K | 1024 | 80   | 320      | 80     | 50%        | 39609.40 | 24.90    |
| 128K | 1024 | 80   | 320      | 80     | 70%        | 26776.40 | 25.90    |
| 128K | 1024 | 80   | 320      | 80     | 90%        | 9041.0   | 27.20    |
| 128K | 1024 | 100  | 400      | 100    | 30%        | 84653.60 | 23.70    |
| 128K | 1024 | 100  | 400      | 100    | 50%        | 52333.30 | 25.20    |
| 128K | 1024 | 100  | 400      | 100    | 70%        | 37833.30 | 26.10    |
| 128K | 1024 | 100  | 400      | 100    | 90%        | 9955.10  | 28.10    |
| 128K | 1024 | 120  | 480      | 120    | 30%        | 111618.6 | 23.6     |
| 128K | 1024 | 120  | 480      | 120    | 50%        | 68280.6  | 25.0     |
| 128K | 1024 | 120  | 480      | 120    | 70%        | 49319.1  | 26.1     |
| 128K | 1024 | 120  | 480      | 120    | 90%        | 19934.5  | 28.1     |
| 128K | 1024 | 140  | 560      | 140    | 30%        | 131692.9 | 23.7     |
| 128K | 1024 | 140  | 560      | 140    | 50%        | 84138.8  | 24.8     |
| 128K | 1024 | 140  | 560      | 140    | 70%        | 60551.2  | 26.0     |
| 128K | 1024 | 140  | 560      | 140    | 90%        | 24689.0  | 28.1     |

