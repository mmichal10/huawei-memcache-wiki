### Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.

DeepSeek-V4 supports million-character long contexts and leads domestic and open-source models in agent capabilities, world knowledge, and reasoning performance. Compared with DeepSeek-V4-Pro, DeepSeek-V4-Flash has somewhat less world knowledge but delivers similar reasoning capability. Its smaller model and activation sizes allow V4-Flash to provide faster, more economical API services. Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.



### Environment

**Hardware:** 4 × Atlas 800 A2 (64 GB × 8)

**Deployment configuration:** 2 × 1P + 1 × 2D

**Model weights:** [DeepSeek-V4-Flash-w8a8-mtp](https://www.modelscope.cn/models/Eco-Tech/DeepSeek-V4-Flash-w8a8-mtp)

<span style="color:#e60000;">Note: This deployment requires the following driver and software versions.</span>

**NPU driver firmware:** 25.5.1 or later

**OS page size:** Run `getconf PAGESIZE` to check the system page size. MemCache can be used only if the result is 4K. If the result is 64K, consider changing the OS.





### Test Results

![ttft_compare.png](https://raw.gitcode.com/user-images/assets/7672915/f57f146e-c0be-4bda-ab7a-fefb6571d1cc/ttft_compare.png 'ttft_compare.png')

Detailed results:

| Pooling | Total requests | Concurrency | Prefixes | Input length | Output length | Prefix reuse | Theoretical hit rate | Actual hit rate | TTFT(ms) | TPOT(ms) |
| ---- | -------- | ---- | ------ | -------- | -------- | ---------- | ---------- | ---------- | -------- | -------- |
| Disabled | 200      | 40   | 40     | 64K      | 1K       | 80%        | 64%        | 12.15%     | 52787.2  | 24.5     |
| Disabled | 250      | 50   | 50     | 64K      | 1K       | 80%        | 64%        | 13.80%     | 70080.6  | 24.3     |
| Disabled | 300      | 60   | 60     | 64K      | 1K       | 80%        | 64%        | 8.25%      | 101942.1 | 24.0     |
| Enabled | 200      | 40   | 40     | 64K      | 1K       | 80%        | 64%        | 59.5%      | 22184.7  | 25.9     |
| Enabled | 250      | 50   | 50     | 64K      | 1K       | 80%        | 64%        | 59.8%      | 25809.9  | 26.4     |
| Enabled | 300      | 60   | 60     | 64K      | 1K       | 80%        | 64%        | 59.9%      | 33407.4 | 26.5     |



Test tool: aisbench





### Deployment Steps

#### 1. Pull the official vllm-ascend image

Pull the official vllm-ascend image.

```shell
docker pull quay.io/ascend/vllm-ascend:v0.23.0
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

- [memfabric](https://pypi.org/project/memfabric-hybrid/#history)

- [memcache](https://pypi.org/project/memcache-hybrid/#history)

Other unreleased versions can be installed by building from source; see the [official build guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/install_run.md).



#### 3. Prepare the MemCache Configuration Files

MemCache consists of MetaService and LocalService. LocalService handles memory allocation, data reads/writes, and transfers. MetaService runs as a separate process, connects to LocalService over TCP, and handles metadata management, data eviction, primary/standby switching, and other high-availability functions.

At startup, MetaService and LocalService load settings from separate configuration files. By default, these files are in the config directory of the MemCache wheel package (run `pip show memcache_hybrid` to find the path).

##### MetaService configuration file

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



##### LocalService configuration file

Prepare an `mmc-local.conf` file on each node. Set the IP addresses in `ock.mmc.meta_service_url` and `ock.mmc.local_service.config_store_url` to the IP of the node running MetaService, and configure memory capacity according to your environment.

```ini
## Set to the IP of the node running MetaService.
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## Set to the IP of the node running MetaService.
ock.mmc.local_service.config_store_url = tcp://127.0.0.1:6000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

 ## Typical A2 configuration for card-side RoCE data transfer.
ock.mmc.local_service.protocol = device_rdma

## Memory allocated per card; adjust according to available memory in your environment.
ock.mmc.local_service.dram.size = 50GB

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md)
```



#### 4. Prepare the Inference Service Scripts

##### 4.1 Prepare the `launch_online_dp.py` script on each node.

```python
import argparse
import multiprocessing
import os
import subprocess
import sys


def parse_args():
    parser = argparse.ArgumentParser()
    parser.add_argument("--dp-size", type=int, required=True, help="Data parallel size.")
    parser.add_argument("--tp-size", type=int, default=1, help="Tensor parallel size.")
    parser.add_argument("--dp-size-local", type=int, default=-1, help="Local data parallel size.")
    parser.add_argument("--dp-rank-start", type=int, default=0, help="Starting rank for data parallel.")
    parser.add_argument("--dp-address", type=str, required=True, help="IP address for data parallel master node.")
    parser.add_argument("--dp-rpc-port", type=str, default=12345, help="Port for data parallel master node.")
    parser.add_argument("--vllm-start-port", type=int, default=9000, help="Starting port for the engine.")
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
        process = multiprocessing.Process(target=run_command, args=(visible_devices, dp_rank, vllm_engine_port))
        processes.append(process)
        process.start()

    for process in processes:
        process.join()
```



##### 4.2 Prepare the `run_dp_template.sh` script on each node.

Update the following values:

- `nic_name`: Use `ifconfig` to find the network interface name.
- Model weight path.
- `MMC_LOCAL_CONFIG_PATH`: Path to the MemCache LocalService configuration file.
- `kv_port`: Must be unique for each instance, for example 30000, 30100, etc.



###### P0

```shell
unset ftp_proxy
unset https_proxy
unset http_proxy
rm -rf ~/ascend/log

nic_name="enp189s0f0"
local_ip=`hostname -I|awk -F " " '{print$1}'`

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
export TASK_QUEUE_ENABLE=1

export MMC_LOCAL_CONFIG_PATH=/home/hsy/config/mmc-local.conf
export HYBM_RDMA_SWAP_SPACE_SIZE=0
export PYTHONHASHSEED=0

vllm serve /home/weights/Eco-Tech/DeepSeek-V4-Flash-w8a8-mtp \
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
    --max-model-len 135000 \
    --max-num-batched-tokens 4096 \
    --max-num-seqs 16 \
    --block-size 128 \
    --enforce-eager \
    --async-scheduling \
    --no-disable-hybrid-kv-cache-manager \
    --enable-prefix-caching \
    --trust-remote-code \
    --gpu-memory-utilization 0.9 \
    --quantization ascend \
    --safetensors-load-strategy 'prefetch' \
    --model-loader-extra-config='{"enable_multithread_load": "true", "num_threads": 128}' \
    --tokenizer-mode deepseek_v4 \
    --tool-call-parser deepseek_v4 \
    --enable-auto-tool-choice \
    --reasoning-parser deepseek_v4 \
    --additional-config '{"enable_cpu_binding": true, "enable_shared_expert_dp": true}' \
    --speculative-config '{"num_speculative_tokens": 1, "method": "mtp","enforce_eager": true}' \
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
                            "dp_size": 2,
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
    }'
```

###### P1

```shell
unset ftp_proxy
unset https_proxy
unset http_proxy
rm -rf ~/ascend/log

nic_name="enp189s0f0"
local_ip=`hostname -I|awk -F " " '{print$1}'`

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
export TASK_QUEUE_ENABLE=1

export MMC_LOCAL_CONFIG_PATH=/home/hsy/config/mmc-local.conf
export HYBM_RDMA_SWAP_SPACE_SIZE=0
export PYTHONHASHSEED=0

vllm serve /home/weights/Eco-Tech/DeepSeek-V4-Flash-w8a8-mtp \
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
    --max-model-len 135000 \
    --max-num-batched-tokens 4096 \
    --max-num-seqs 16 \
    --block-size 128 \
    --enforce-eager \
    --async-scheduling \
    --no-disable-hybrid-kv-cache-manager \
    --enable-prefix-caching \
    --trust-remote-code \
    --gpu-memory-utilization 0.9 \
    --quantization ascend \
    --safetensors-load-strategy 'prefetch' \
    --model-loader-extra-config='{"enable_multithread_load": "true", "num_threads": 128}' \
    --tokenizer-mode deepseek_v4 \
    --tool-call-parser deepseek_v4 \
    --enable-auto-tool-choice \
    --reasoning-parser deepseek_v4 \
    --additional-config '{"enable_cpu_binding": true, "enable_shared_expert_dp": true}' \
    --speculative-config '{"num_speculative_tokens": 1, "method": "mtp","enforce_eager": true}' \
    --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_producer",
        "kv_connector_extra_config": {
            "connectors": [
                {
                    "kv_connector": "MooncakeHybridConnector",
                    "kv_role": "kv_producer",
                    "kv_port": "30100",
                    "kv_connector_extra_config": {
                        "prefill": {
                            "dp_size": 2,
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
    }'
```



###### D0

```shell
unset ftp_proxy
unset https_proxy
unset http_proxy
rm -rf ~/ascend/log

nic_name="enp189s0f0"
local_ip=`hostname -I|awk -F " " '{print$1}'`

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

export MMC_LOCAL_CONFIG_PATH=/home/hsy/config/mmc-local.conf
export HYBM_RDMA_SWAP_SPACE_SIZE=0
export PYTHONHASHSEED=0

vllm serve /home/weights/Eco-Tech/DeepSeek-V4-Flash-w8a8-mtp \
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
    --max-model-len 135000 \
    --max-num-batched-tokens 60 \
    --max-num-seqs 30 \
    --async-scheduling \
    --block-size 128 \
    --no-disable-hybrid-kv-cache-manager \
    --no-enable-prefix-caching \
    --trust-remote-code \
    --gpu-memory-utilization 0.9 \
    --quantization ascend \
    --safetensors-load-strategy 'prefetch' \
    --model-loader-extra-config='{"enable_multithread_load": "true", "num_threads": 128}' \
    --tokenizer-mode deepseek_v4 \
    --tool-call-parser deepseek_v4 \
    --enable-auto-tool-choice \
    --reasoning-parser deepseek_v4 \
    --speculative-config '{"num_speculative_tokens": 1, "method": "mtp","enforce_eager": true}' \
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
                    "kv_port": "30400",
                    "kv_connector_extra_config": {
                        "prefill": {
                            "dp_size": 2,
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
    }'
```

###### D1

```shell
unset ftp_proxy
unset https_proxy
unset http_proxy
rm -rf ~/ascend/log

nic_name="enp189s0f0"
local_ip=`hostname -I|awk -F " " '{print$1}'`

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

export MMC_LOCAL_CONFIG_PATH=/home/hsy/config/mmc-local.conf
export HYBM_RDMA_SWAP_SPACE_SIZE=0
export PYTHONHASHSEED=0

vllm serve /home/weights/Eco-Tech/DeepSeek-V4-Flash-w8a8-mtp \
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
    --max-model-len 135000 \
    --max-num-batched-tokens 60 \
    --max-num-seqs 30 \
    --async-scheduling \
    --block-size 128 \
    --no-disable-hybrid-kv-cache-manager \
    --no-enable-prefix-caching \
    --trust-remote-code \
    --gpu-memory-utilization 0.9 \
    --quantization ascend \
    --safetensors-load-strategy 'prefetch' \
    --model-loader-extra-config='{"enable_multithread_load": "true", "num_threads": 128}' \
    --tokenizer-mode deepseek_v4 \
    --tool-call-parser deepseek_v4 \
    --enable-auto-tool-choice \
    --reasoning-parser deepseek_v4 \
    --speculative-config '{"num_speculative_tokens": 1, "method": "mtp","enforce_eager": true}' \
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
                    "kv_port": "30400",
                    "kv_connector_extra_config": {
                        "prefill": {
                            "dp_size": 2,
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
    }'
```



#### 5. Start the Services

##### 5.1 Start the MetaService Metadata Management Service

On the node selected earlier to run MetaService, start the metadata management service with the following script.

```shell
# Replace with the path to the MetaService configuration file.
export MMC_META_CONFIG_PATH=/home/hsy/config/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```



##### 5.2 Start the vLLM Inference Service

###### P0、P1

Set `--dp-address` to the IP address of the node hosting each instance.

```shell
python launch_online_dp.py --dp-size 2 --tp-size 4 --dp-size-local 2 --dp-rank-start 0 --dp-address <p_node_0_ip> --dp-rpc-port 12321 --vllm-start-port 7100

python launch_online_dp.py --dp-size 2 --tp-size 4 --dp-size-local 2 --dp-rank-start 0 --dp-address <p_node_1_ip> --dp-rpc-port 12321 --vllm-start-port 7100
```



###### D0、D1

Set `--dp-address` to the IP address of the primary D node (D0).

Start `--dp-rank-start` at 0 and increment it on each node by `--dp-size-local`.

```shell
python launch_online_dp.py --dp-size 16 --tp-size 1 --dp-size-local 8 --dp-rank-start 0 --dp-address <d_node_0_ip> --dp-rpc-port 12321 --vllm-start-port 7100

python launch_online_dp.py --dp-size 16 --tp-size 1 --dp-size-local 8 --dp-rank-start 8 --dp-address <d_node_0_ip> --dp-rpc-port 12321 --vllm-start-port 7100
```



##### 5.3 Start the Request Forwarding Service on Any Node

vllm-ascend provides two proxy implementations with different request-routing behavior:

- **`load_balance_proxy_layerwise_server_example.py`**: Requests are first routed to a D node, which forwards them to a P node as needed. This proxy is designed for use with MooncakeLayerwiseConnector. [load_balance_proxy_layerwise_server_example.py](https://github.com/vllm-project/vllm-ascend/blob/main/examples/disaggregated_prefill_v1/load_balance_proxy_layerwise_server_example.py)
- **`load_balance_proxy_server_example.py`**: Requests are first routed to a P node, which forwards them to a D node for subsequent processing. This proxy is designed for use with MooncakeConnector. [load_balance_proxy_server_example.py](https://github.com/vllm-project/vllm-ascend/blob/main/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py)

Run the following script on any node:

```shell
python3 load_balance_proxy_server_example.py \
  --port 1999 \
  --host 127.0.0.1 \
  --prefiller-hosts \
    <p_node_0_ip> <p_node_0_ip> \
    <p_node_1_ip> <p_node_1_ip> \
  --prefiller-ports \
    7100 7101 \
    7100 7101 \
  --decoder-hosts \
    <d_node_0_ip> <d_node_0_ip> <d_node_0_ip> <d_node_0_ip> <d_node_0_ip> <d_node_0_ip> <d_node_0_ip> <d_node_0_ip> \
    <d_node_1_ip> <d_node_1_ip> <d_node_1_ip> <d_node_1_ip> <d_node_1_ip> <d_node_1_ip> <d_node_1_ip> <d_node_1_ip> \
  --decoder-ports \
    7100 7101 7102 7103 7104 7105 7106 7107 \
    7100 7101 7102 7103 7104 7105 7106 7107
    
```



### Testing Method

Install ais_bench_benchmark.

```shell
pip3 install ais_bench_benchmark[full]
```

Clone the [rayn-zzz/aisbench_auto_tools_prefix](https://github.com/rayn-zzz/aisbench_auto_tools_prefix) repository and configure `config.py` according to its README.



Example test command:

```
python3 aisbench_test.py --input_len 65536 --output_len 1024 --data_num 250 --concurrency 50 --dataset_type prefix_cache --prefix_num 50 --repeat_rate 0.8 --seed $(date +%s)
```

