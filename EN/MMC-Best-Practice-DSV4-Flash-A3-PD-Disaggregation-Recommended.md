### Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.



DeepSeek-V4 supports million-character long contexts and leads domestic and open-source models in agent capabilities, world knowledge, and reasoning performance. Compared with DeepSeek-V4-Pro, DeepSeek-V4-Flash has somewhat less world knowledge but delivers similar reasoning capability. Its smaller model and activation sizes allow V4-Flash to provide faster, more economical API services. Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.


### Test Results


Validation with the aisbench prefix testing tool:
Tool documentation: `https://github.com/rayn-zzz/aisbench_auto_tools_prefix/tree/main`

Case 1: 90% prefix; 128K input tokens; 1 output token; concurrency 60; 240 requests total, with 15 distinct prefixes shared by every 16 requests.

`python3 aisbench_test.py --input_len 131072 --output_len 1 --data_num 240 --concurrency 60 --request_rate 0 --dataset_type prefix_cache --prefix_num 15 --repeat_rate 0.9 --prefix_test --dp 4`

Results:

| MemCache enabled | mean ttft(ms) | mean tpot(ms) | HBM hit rate | DRAM hit rate |
|--|--|--|--|--|
| No | 51837.1 | / | 59.06% | / |
| Yes | 32121.5 | / | 57.24% | 70.76% |


### Environment

**Hardware:** 2 × Atlas 800I A3

**Model weights:** [DeepSeek-V4-Flash-w8a8-mtp](https://www.modelscope.cn/models/Eco-Tech/DeepSeek-V4-Flash-w8a8-mtp)

<span style="color:#e60000;">Note: This deployment requires the following driver and software versions.</span>

**NPU driver firmware:** >=25.5.1

【L1 1520 CC】>=1.5.0

**OS kernel version:** >=5.10


### Deployment


#### 1. Pull the official vllm-ascend image

```shell
docker pull quay.io/ascend/vllm-ascend:v0.23.0rc1
```

For the container launch command, see the [official guide for deploying DeepSeek-V4-Flash with vllm-ascend](https://docs.vllm.ai/projects/ascend/zh-cn/latest/tutorials/models/DeepSeek-V4-Flash.html).

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


#### 3. Edit the MemCache Configuration Files

MemCache consists of MetaService and LocalService. LocalService handles memory allocation, data reads/writes, and transfers. MetaService runs separately, connects to LocalService over TCP, and handles metadata management, data eviction, primary/standby switching, and other high-availability functions.

For MemCache on A3, `device_sdma` with [disaggregated deployment](https://gitcode.com/Ascend/memcache/wiki/MemCache+vLLM+A3%E5%88%86%E7%A6%BB%E9%83%A8%E7%BD%B2%E6%A1%88%E4%BE%8B.md) is recommended for optimal performance.

At startup, MetaService and LocalService load settings from separate configuration files. By default, these files are in the config directory of the MemCache wheel package.

```
/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/
```
Here, `mmc-meta.conf` is the MetaService configuration file and `mmc-local.conf` is the LocalService configuration file.



##### MetaService Configuration File
Prepare the MetaService configuration file on any node.

Replace `<ip_or_domain>` below with the IP address or domain name used in your environment. The port can be changed as needed; if changed, update the corresponding IP/domain and port in the LocalService configuration file as well.

```ini
## Meta service start-up url
ock.mmc.meta_service_url = tcp://<ip_or_domain>:5000

## Config store url
ock.mmc.meta_service.config_store_url = tcp://<ip_or_domain>:6000

## HTTP service url
ock.mmc.meta_service.metrics_url = http://127.0.0.1:8000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md)
```



##### LocalService Configuration File
Prepare a LocalService configuration file on all nodes.

```ini
## Meta service start-up url
## Keep consistent with the same name configuration in mmc-meta.conf
ock.mmc.meta_service_url = tcp://<ip_or_domain>:5000

## Config store url
## Keep consistent with the same name configuration in mmc-meta.conf
ock.mmc.local_service.config_store_url = tcp://<ip_or_domain>:6000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

## Data transfer protocol options
ock.mmc.local_service.protocol = device_sdma

## DRAM space usage
## Adjust the DRAM size for your environment.
ock.mmc.local_service.dram.size = 100GB

## max DRAM space usage
ock.mmc.local_service.max.dram.size = 1024GB

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md)
```

In disaggregated mode, prepare two configuration files on each node: one for the standalone LocalService process and one for the client integrated with the vLLM inference process. The standalone LocalService process contributes memory; the vLLM-integrated client handles data reads, writes, and transfers. The two files differ as follows; all other fields must remain the same:

###### Standalone LocalService Process Configuration

```ini
## DRAM space usage
## Adjust the DRAM size for your environment.
ock.mmc.local_service.dram.size = 100GB
```


###### Client Configuration for the vLLM Inference Process

```ini
## DRAM space usage
## Set the DRAM size to 0; this process does not contribute memory.
ock.mmc.local_service.dram.size = 0
```

Save the files as `mmc-local-standalone.conf` and `mmc-local.conf`, respectively.

##### Prepare the Standalone LocalService Process Launch Script

Prepare the following script on each node; the example name is `memcache_standalone.py`.

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

For more configuration options, see the [official configuration guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md).

#### 4. Prepare the Inference Service Launch Scripts

This section describes deploying DeepSeek-V4-Flash in a 1P1D configuration on two Atlas 800 A3 nodes for improved performance.

##### 4.1 Prepare the `launch_online_dp.py` Script on Each Node

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

##### 4.2 P Node Inference Service Launch Script: `run_dp_template.sh`

```bash
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

##### 4.3 D Node Inference Service Launch Script: `run_dp_template.sh`

```bash
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

##### 4.4 Request Forwarding Proxy Script
Prepare the following file on any node:

Get the proxy program [load_balance_proxy_server_example.py](https://github.com/vllm-project/vllm-ascend/blob/main/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py) from the vllm-ascend repository.

Then prepare a Bash script to start it, named `start_proxy.sh` here. Replace `<proxy_ip>`, `<p_ip>`, and `<d_ip>` with the actual service IP addresses: `<proxy_ip>` is the node running the proxy, while `<p_ip>` and `<d_ip>` are the P and D node addresses, respectively.

```bash
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

Start the MetaService metadata management service on any node, using the actual configuration file path.
```bash
export MMC_META_CONFIG_PATH=/xxx/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```

##### 5.2 Start the Standalone LocalService Process

Start the standalone LocalService process on all nodes, using the actual configuration file path.
```bash
export MMC_LOCAL_CONFIG_PATH="/xxx/mmc-local-standalone.conf"
python3 memcache_standalone.py
```
Wait until all nodes start successfully (the log shows `Successfully initialized memcache.`) before proceeding.

##### 5.3 Start the vLLM Inference Service

Replace `<p_ip>` and `<d_ip>` with the P and D node IP addresses. If you change ports, update the request forwarding proxy script `start_proxy.sh` as well.

P node:
```bash
python launch_online_dp.py --dp-size 4 --tp-size 4 --dp-size-local 4 --dp-rank-start 0 --dp-address <p_ip> --dp-rpc-port 12321 --vllm-start-port 7100
```

D node:
```bash
python launch_online_dp.py --dp-size 16 --tp-size 1 --dp-size-local 16 --dp-rank-start 0 --dp-address <d_ip> --dp-rpc-port 12321 --vllm-start-port 7100
```

##### 5.4 Start the Request Forwarding Proxy Script
Start it on any node.
```bash
bash start_proxy.sh
```





