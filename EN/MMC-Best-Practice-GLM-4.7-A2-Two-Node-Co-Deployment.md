### Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.

The GLM-4.7 series from Zhipu is a high-intelligence model family. It strengthens coding, long-horizon task planning, and tool collaboration for **Agentic Coding**, and performs strongly among open-source models on several public benchmarks. Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.


### Environment
**Hardware:** Two Atlas 800T A2 nodes, 16 cards total

<span style="color:#e60000;">Note: This deployment requires the following driver and software versions.</span>

**NPU driver firmware:** 25.5.1 or later [Download](https://www.hiascend.com/hardware/firmware-drivers/old/community?product=4&model=32&cann=All&driver=Ascend+HDK+25.5.1)



### Deployment

This deployment uses the environment and scripts provided in [GLM-4.7_vLLM-ascend](https://ai.gitcode.com/Ascend-SACT/GLM-4.7_vLLM-ascend/tree/main) as its baseline.

**CANN software:** CANN 8.5

**Inference framework:** vLLM

```
torch                             2.9.0+cpu
torch_npu                         2.9.0
torchvision                       0.24.0
transformers                      4.57.6
triton-ascend                     3.2.0
vllm                              0.14.1+empty               /vllm-workspace/vllm
vllm_ascend                       0.14.0rc2.dev68+gf55ded242 /vllm-workspace/vllm-ascend
```



#### 1. Install MF

Online installation:

```shell
pip install memfabric_hybrid==1.1.5
pip install memcache_hybrid==1.1.5
```

If network restrictions prevent online installation, download the package from [PyPI · The Python Package Index](https://pypi.org/) and install it offline.

Other unreleased versions can be installed by building from source; see the official documentation.

[memcache/docs/zh/install_run.md](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/install_run.md)



#### 2. Edit the Configuration Files

MemCache consists of MetaService and LocalService. LocalService is integrated with the inference framework and starts with the inference process; it handles memory allocation, data reads/writes, and transfers. MetaService runs separately, connects to LocalService over TCP, and handles metadata management, data eviction, primary/standby switching, and other high-availability functions.

At startup, MetaService and LocalService load settings from separate configuration files. By default, these files are in the config directory of the MemCache wheel package.

```
/usr/local/lib/python3.11/site-packages/memcache_hybrid/config
```



###### MetaService configuration file

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



###### LocalService configuration file

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

## Using the card-side RDMA network transfer protocol is typical for an A2 cluster.
ock.mmc.local_service.protocol = device_rdma

## Provide 80 GB of pooled space for each inference process.
ock.mmc.local_service.dram.size = 80GB
```

For more configuration options, see the official documentation: https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md



#### 3. Prepare the vLLM Service Launch Script

node0

```shell
local_ip="61.47.16.106"
nic_name="enp189s0f0"

export HCCL_IF_IP=$local_ip

#export PYTHONPATH=/workspace/vllm:$PYTHONPATH
export PYTHONPATH=/vllm-workspace/vllm:$PYTHONPATH

export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name
export HCCL_BUFFSIZE=1024
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_USE_V1=1
export HCCL_OP_EXPANSION_MODE=AIV
export VLLM_VERSION=0.14.1
export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export VLLM_ASCEND_ENABLE_TOPK_OPTIMIZE=1
export VLLM_ASCEND_BALANCE_SCHEDULING=1
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

export LD_PRELOAD=/usr/lib/aarch64-linux-gnu/libjemalloc.so.2:$LD_PRELOAD

export VLLM_TORCH_PROFILER_DIR=/home
export VLLM_TORCH_PROFILER_WITH_STACK=0

export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf

vllm serve /data/GLM-4.7-w8a8/ \
    --host 0.0.0.0 \
    --port 8013 \
    --enable-expert-parallel \
    --tensor-parallel-size 8 \
    --data_parallel_size 2 \
    --data-parallel-size-local 1 \
    --data-parallel-address 61.47.16.106 \
    --data-parallel-rpc-port 13345  \
    --seed 1024 \
    --served-model-name GLM-4.7-w8a8 \
    --max-model-len 133120 \
    --max-num-batched-tokens 8192 \
    --max-num-seqs 32 \
    --async-scheduling \
    --quantization ascend \
    --trust-remote-code \
    --gpu-memory-utilization 0.9 \
    --speculative-config '{"num_speculative_tokens": 3, "model":"/data/GLM-4.7-w8a8/", "method":"mtp"}' \
    --compilation-config '{"cudagraph_capture_sizes": [1,4,8,16,32,64], "cudagraph_mode": "FULL_DECODE_ONLY"}' \
    --additional-config '{"ascend_compilation_config": {"fuse_qknorm_rope": true}, "ascend_fusion_config": {"fusion_ops_gmmswigluquant": false}, "multistream_overlap_shared_expert":"true", "enable_shared_expert_dp": "true"}' \
    --kv-transfer-config \
    '{
        "kv_connector": "AscendStoreConnector",
        "kv_role": "kv_both",
        "kv_connector_extra_config": {
            "backend": "memcache",
            "mooncake_rpc_port":"0"
        }
    }' 2>&1 | tee output.log

```

node1

```shell
local_ip="61.47.16.107"
nic_name="enp189s0f0"

export HCCL_IF_IP=$local_ip

#export PYTHONPATH=/workspace/vllm:$PYTHONPATH
export PYTHONPATH=/vllm-workspace/vllm:$PYTHONPATH

export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name
export HCCL_BUFFSIZE=1024
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_USE_V1=1
export HCCL_OP_EXPANSION_MODE=AIV
export VLLM_VERSION=0.14.1
export VLLM_ASCEND_ENABLE_FUSED_MC2=1
export VLLM_ASCEND_ENABLE_TOPK_OPTIMIZE=1
export VLLM_ASCEND_BALANCE_SCHEDULING=1
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

export LD_PRELOAD=/usr/lib/aarch64-linux-gnu/libjemalloc.so.2:$LD_PRELOAD

export VLLM_TORCH_PROFILER_DIR=/home
export VLLM_TORCH_PROFILER_WITH_STACK=0

export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf

vllm serve /data/GLM-4.7-w8a8/ \
    --host 0.0.0.0 \
    --port 8013 \
    --headless \
    --enable-expert-parallel \
    --tensor-parallel-size 8 \
    --data_parallel_size 2 \
    --data-parallel-size-local 1 \
    --data-parallel-start-rank 1 \
    --data-parallel-address 61.47.16.106 \
    --data-parallel-rpc-port 13345  \
    --seed 1024 \
    --served-model-name GLM-4.7-w8a8 \
    --max-model-len 133120 \
    --max-num-batched-tokens 8192 \
    --max-num-seqs 32 \
    --async-scheduling \
    --quantization ascend \
    --trust-remote-code \
    --gpu-memory-utilization 0.9 \
    --speculative-config '{"num_speculative_tokens": 3, "model":"/data/GLM-4.7-w8a8/", "method":"mtp"}' \
    --compilation-config '{"cudagraph_capture_sizes": [1,4,8,16,32,64], "cudagraph_mode": "FULL_DECODE_ONLY"}' \
    --additional-config '{"ascend_compilation_config": {"fuse_qknorm_rope": true}, "ascend_fusion_config": {"fusion_ops_gmmswigluquant": false}, "multistream_overlap_shared_expert":"true", "enable_shared_expert_dp": "true"}' \
    --kv-transfer-config \
    '{
        "kv_connector": "AscendStoreConnector",
        "kv_role": "kv_both",
        "kv_connector_extra_config": {
            "backend": "memcache",
            "mooncake_rpc_port":"0"
        }
    }' 2>&1 | tee output.log

```

**Adjust the following values for your environment:**

- `local_ip`: IP address of the local node.
- `nic_name`: Local network interface name.
- `data-parallel-address`: Set to the IP address of the DP leader node.
- Model weight path.



**Compared with the baseline script, enabling MF requires only these two changes:**

1. Add an environment variable pointing to the MemCache LocalService configuration file (the path shown is the wheel package's default configuration path).

```shell
export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
```

2. Add the `kv-transfer-config` launch parameter.

```shell
--kv-transfer-config \
    '{
        "kv_connector": "AscendStoreConnector",
        "kv_role": "kv_both",
        "kv_connector_extra_config": {
            "backend": "memcache"
        }
    }'
```





#### 4. Start the Services

First start the MetaService metadata management service.

```shell
export MMC_META_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```

Then start the inference service using the script and wait for startup to complete.



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
            "value": 10240
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
    --model /data/GLM-4.7-w8a8/ \
    --served-model-name GLM-4.7-w8a8 \
    --input-file generate_multi_turn.json \
    --num-clients 8 \                        # Concurrency
    --max-active-conversations 16 \
    --no-early-stop \
    --request-timeout-sec 600 \
    -u http://<ip>:<port>                    # IP and port of the inference service
```



**Measured results on two A2 nodes**

| MF | Turns | Input per turn | Output per turn | Concurrency | Total sessions | HBM hit rate | DRAM hit rate | mean ttft (ms) | mean tpot (ms) | Total time (s) |
| ------- | ---- | -------- | -------- | ---- | -------- | --------- | ---------- | -------------- | -------------- | ---------- |
| disable | 20   | 10240    | 1024     | 8    | 16       | 17.8%     | /          | 18320.94       | 126.96         | 2835.736   |
| enable  | 20   | 10240    | 1024     | 8    | 16       | 10.25%    | 55.75%     | 7646.07        | 65.63          | 1451.111   |
| disable | 20   | 9216     | 1024     | 8    | 20       | 24.15%    | /          | 15152.60       | 109.57         | 2647.894   |
| enable  | 20   | 9216     | 1024     | 8    | 20       | 21.25%    | 52.9%      | 6182.19        | 60.95          | 1411.294   |
| disable | 40   | 4096     | 1024     | 8    | 20       | 27.85%    | /          | 15090.55       | 107.69         | 5130.660   |
| enable  | 40   | 4096     | 1024     | 8    | 20       | 23.1%     | 60.85%     | 4253.99        | 48.70          | 2210.721   |

Average TTFT is reduced by 60%–70%.



### Metric Observations

Collect the following metrics from vLLM logs to plot runtime trends.

```
GPU KV cache usage: XX%, Prefix cache hit rate: XX%, External prefix cache hit rate: XX%
```
![metrics-trend.png](https://raw.gitcode.com/user-images/assets/7672915/b3ec1e88-c776-453e-9fec-8909a7321f54/metrics-trend.png 'metrics-trend.png')


Analysis:

Stage 1: As sequence length gradually increases, GPU KV Cache utilization and the HBM hit rate rise together.

Stage 2: GPU KV Cache utilization exceeds 90%, and the device-side prefix cache is evicted. The HBM hit rate begins to fall (if the memory pool is enabled, the DRAM hit rate begins to rise).

Stage 3: GPU KV Cache utilization remains above 90%, the HBM hit rate approaches zero, and the DRAM hit rate continues to rise.



The higher the external cache hit rate, the greater the performance benefit.
