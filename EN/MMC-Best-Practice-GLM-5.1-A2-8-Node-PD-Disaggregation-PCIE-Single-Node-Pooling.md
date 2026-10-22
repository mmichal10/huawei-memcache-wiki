### Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.

**GLM-5.1** is Zhipu's latest flagship model, with greatly enhanced coding ability and **long-horizon task** performance. It can work continuously and autonomously for up to 8 hours on a single task, completing the full cycle from planning and execution to iterative optimization and delivering engineering-grade results. Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.



### Environment

**Hardware:** 8 × A2

**Model weights:** GLM-5.1-w8a8c8-MTP

**Note: MemCache's preferred A2 solution (multi-node pooling over card-side RoCE) requires specific NPU driver firmware and OS page size, which may not be available in some customer environments. This example uses the alternative MemCache solution (single-node L2 pooling over PCIe) to avoid low-level software compatibility issues.**

**NPU driver firmware:** No special requirements

**OS page size:** No special requirements


### Test Results

Detailed results:

| Pooling | Total requests | Concurrency | Prefixes | Input length | Output length | Prefix reuse rate | Prefix warmup | Actual hit rate | TTFT(ms) | TPOT(ms) |
| ---- | -------- | ---- | ------ | -------- | -------- | ---------- | -------- | ---------- | -------- | -------- |
| MMC  | 80       | 16   | 16     | 64K      | 1K       | 80%        | Y        | 48.65%     | 10634.9  | 35.3     |
| MMC  | 160      | 32   | 32     | 64K      | 1K       | 80%        | Y        | 48.675%    | 15038.9  | 41.4     |
| None | 80       | 16   | 16     | 64K      | 1K       | 80%        | Y        | 0.125%     | 29632.5  | 34.9     |
| None | 160      | 32   | 32     | 64K      | 1K       | 80%        | Y        | 0.05%      | 81990.6  | 35.2     |


Test tool: aisbench



### Deployment

#### 1. Pull the official vllm-ascend image

```shell
docker pull quay.io/ascend/vllm-ascend:v0.23.0
```

Add `-v /dev/hugepages:/dev/hugepages` when starting the container to use huge-page memory and improve MemCache read/write bandwidth.


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



**This example uses a separate memory pool on each node, so MetaService must be started on every node.**



##### MetaService Configuration File

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



##### LocalService Configuration File

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

## Memory contributed per card. Since the D node does not read or write data, this can be reduced to 1 GB.
ock.mmc.local_service.dram.size = 80GB

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md)
```



#### 4. Prepare the vLLM Launch Scripts

##### 4.1 Prepare the `launch_online_dp.py` Script on Each Node

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
    parser.add_argument("--pp-size", type=int, default=1, help="Pipeline parallel size.")
    parser.add_argument("--dp-size-local", type=int, default=-1, help="Local data parallel size.")
    parser.add_argument("--dp-rank-start", type=int, default=0, help="Starting rank for data parallel.")
    parser.add_argument("--dp-address", type=str, required=True, help="IP address for data parallel master node.")
    parser.add_argument("--dp-rpc-port", type=str, default="12321", help="Port for data parallel master node.")
    parser.add_argument("--vllm-start-port", type=int, default=8000, help="Starting port for the engine.")
    return parser.parse_args()

args = parse_args()
dp_size = args.dp_size
tp_size = args.tp_size
pp_size = args.pp_size
dp_size_local = args.dp_size_local
if dp_size_local == -1:
    dp_size_local = dp_size
dp_rank_start = args.dp_rank_start
dp_address = args.dp_address
dp_rpc_port = args.dp_rpc_port
vllm_start_port = args.vllm_start_port
gpus_per_dp_rank = tp_size * pp_size

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
        str(pp_size),
    ]
    subprocess.run(command, check=True)

if __name__ == "__main__":
    template_path = "./run_dp_template.sh"
    if not os.path.exists(template_path):
        print(f"Template file {template_path} does not exist.")
        sys.exit(1)

    processes = []
    num_cards = dp_size_local * gpus_per_dp_rank

    for i in range(dp_size_local):
        dp_rank = dp_rank_start + i
        vllm_engine_port = vllm_start_port + i
        visible_devices = ",".join(str(x) for x in range(i * gpus_per_dp_rank, (i + 1) * gpus_per_dp_rank))
        process = multiprocessing.Process(
            target=run_command,
            args=(visible_devices, dp_rank, vllm_engine_port)
        )
        processes.append(process)
        process.start()

    for process in processes:
        process.join()
```



##### 4.2 Prepare the `run_dp_template.sh` Script on Each Node

###### P Node

```shell
export VLLM_RPC_TIMEOUT=3600000
export VLLM_EXECUTE_MODEL_TIMEOUT_SECONDS=30000
export HCCL_EXEC_TIMEOUT=204
export HCCL_CONNECT_TIMEOUT=120

nic_name="enp189s0f0"
local_ip=`hostname -I|awk -F " " '{print$1}'`

# The following environment variables do not need to be changed.
export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_ASCEND_ENABLE_MLAPO=1
export HCCL_BUFFSIZE=256
export TASK_QUEUE_ENABLE=1
export HCCL_OP_EXPANSION_MODE="AIV"
export VLLM_USE_V1=1
export ASCEND_RT_VISIBLE_DEVICES=$1
export LD_LIBRARY_PATH=/usr/local/python3.11.10/lib:/usr/local/lib:$LD_LIBRARY_PATH
#export LD_LIBRARY_PATH=/usr/local/python3.11.10/lib/python3.11/site-packages/mooncake:$LD_LIBRARY_PATH
export ASCEND_AGGREGATE_ENABLE=1
export ASCEND_TRANSPORT_PRINT=1
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1

export PYTHONHASHSEED=0
export MMC_LOCAL_CONFIG_PATH="/mnt/share/h30028424/config/mmc-local-p.conf"
export HCCL_INTRA_ROCE_ENABLE=1

# vLLM service launch configuration.
vllm serve /home/weights/Eco-Tech/GLM-5.1-w8a8c8-MTP/ \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-expert-parallel \
    --enable-prefix-caching \
    --seed 1024 \
    --enable-chunked-prefill \
    --served-model-name glm-5 \
    --async-scheduling \
    --max-model-len 141312 \
    --max-num-batched-tokens 8192 \
    --trust-remote-code \
    --max-num-seqs 256 \
    --gpu-memory-utilization 0.9 \
    --safetensors-load-strategy prefetch \
    --quantization ascend \
    --enforce-eager \
    --enable-auto-tool-choice \
    --tool-call-parser glm47 \
    --reasoning-parser glm45 \
    --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_producer",
        "kv_load_failure_policy": "recompute",
        "kv_connector_extra_config": {
            "connectors": [
                {
                    "kv_connector": "MooncakeConnectorV1",
                    "kv_role": "kv_producer",
                    "kv_port": "30000",
                    "kv_connector_extra_config": {
                        "prefill": {
                            "dp_size": 4,
                            "tp_size": 8
                        },
                        "decode": {
                            "dp_size": 8,
                            "tp_size": 4
                        }
                    }
                },
                {
                    "kv_connector": "AscendStoreConnector",
                    "kv_role": "kv_producer",
                    "kv_connector_extra_config": {
                        "lookup_rpc_port": "0",
                        "backend": "memcache"
                    }
                }  
            ]
        }
    }' \
    --additional-config \
    '{
        "multistream_overlap_shared_expert": true,
	    "enable_dsa_cp": true,
	    "enable_sparse_sfa_c8": true, 
        "enable_sparse_li_c8": true,
	    "fuse_muls_add": true,
	    "recompute_scheduler_enable": false
    }' \
    --profiler-config \
    '{
        "profiler": "torch", 
        "torch_profiler_dir": "/mnt/share/xxx/prof", 
        "torch_profiler_with_stack": false
    }' \
    --speculative-config '{"num_speculative_tokens": 1, "method": "deepseek_mtp", "enforce_eager": true}'

```



###### D Node

```shell
export VLLM_RPC_TIMEOUT=3600000
export VLLM_EXECUTE_MODEL_TIMEOUT_SECONDS=30000
export HCCL_EXEC_TIMEOUT=204
export HCCL_CONNECT_TIMEOUT=120

nic_name="enp189s0f0"
local_ip=`hostname -I|awk -F " " '{print$1}'`

# The following environment variables do not need to be changed.
export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_ASCEND_ENABLE_MLAPO=1
export HCCL_BUFFSIZE=2560
export TASK_QUEUE_ENABLE=1
export HCCL_OP_EXPANSION_MODE="AIV"

export VLLM_USE_V1=1
export ASCEND_RT_VISIBLE_DEVICES=$1
export LD_LIBRARY_PATH=/usr/local/python3.11.10/lib:/usr/local/lib:$LD_LIBRARY_PATH
#export LD_LIBRARY_PATH=/usr/local/python3.11.10/lib/python3.11/site-packages/mooncake:$LD_LIBRARY_PATH

export PYTHONHASHSEED=0
export MMC_LOCAL_CONFIG_PATH="/mnt/share/h30028424/config/mmc-local-d.conf"
export HCCL_INTRA_ROCE_ENABLE=1

export ACL_OP_INIT_MODE=1

# vLLM service launch configuration.
vllm serve /home/weights/Eco-Tech/GLM-5.1-w8a8c8-MTP/ \
    --host 0.0.0.0 \
    --port $2 \
    --data-parallel-size $3 \
    --data-parallel-rank $4 \
    --data-parallel-address $5 \
    --data-parallel-rpc-port $6 \
    --tensor-parallel-size $7 \
    --enable-prefix-caching \
    --seed 1024 \
    --served-model-name glm-5 \
    --async-scheduling \
    --max-model-len 200000 \
    --max-num-batched-tokens 256 \
    --trust-remote-code \
    --max-num-seqs 128 \
    --gpu-memory-utilization 0.9 \
    --safetensors-load-strategy prefetch \
    --quantization ascend \
    --enable-auto-tool-choice \
    --tool-call-parser glm47 \
    --reasoning-parser glm45 \
    --kv-transfer-config \
    '{
        "kv_connector": "MultiConnector",
        "kv_role": "kv_consumer",
        "kv_load_failure_policy": "recompute",
        "kv_connector_extra_config": {
            "connectors": [
                {
                    "kv_connector": "MooncakeConnectorV1",
                    "kv_role": "kv_consumer",
                    "kv_port": "30100",
                    "kv_connector_extra_config": {
                        "prefill": {
                            "dp_size": 4,
                            "tp_size": 8
                        },
                        "decode": {
                            "dp_size": 8,
                            "tp_size": 4
                        }
                    }
                },
                {
                    "kv_connector": "AscendStoreConnector",
                    "kv_role": "kv_consumer",
                    "kv_connector_extra_config": {
                        "lookup_rpc_port": "0",
                        "backend": "memcache"
                    }
                }
            ]
        }
    }' \
     --compilation-config \
    '{
        "cudagraph_mode": "FULL_DECODE_ONLY",
        "cudagraph_capture_sizes": [4,8,16,24,32,40,48,56,64,96,128,160,192,224,256,298,320,352,384]
    }' \
    --profiler-config \
    '{
        "profiler": "torch", 
        "torch_profiler_dir": "/mnt/share/xxx/prof", 
        "torch_profiler_with_stack": false
    }' \
    --additional-config \
    '{
        "multistream_overlap_shared_expert": true,
	    "fuse_muls_add": true,
	    "recompute_scheduler_enable": true,
	    "enable_sparse_sfa_c8": true, 
        "enable_sparse_li_c8": true
    }' \
    --speculative-config '{"num_speculative_tokens": 3, "method":"deepseek_mtp", "enforce_eager":true}'

```



**Adjust the following values for your environment:**

- `nic_name`: Local network interface name.
- Model weight path.
- `MMC_LOCAL_CONFIG_PATH`: Path to the MemCache LocalService configuration file (`mmc-local.conf`).



##### 4.3 Request Forwarding Script

Get the proxy program [load_balance_proxy_server_example.py](https://github.com/vllm-project/vllm-ascend/blob/main/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py) from the vllm-ascend repository.

Prepare the following request forwarding script named `start_proxy.sh`. Update the IP addresses and ports to match the values used to start the services.

```shell
unset http_proxy
unset https_proxy

python load_balance_proxy_server_example.py \
    --port 9000 \
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

First, run the following command on the host to inspect the huge-page configuration and obtain Hugepagesize. Example:

```shell
[root@node-106 memcache]# grep -i huge /proc/meminfo
AnonHugePages:   8689664 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:   200000
HugePages_Free:    200000
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:        409600000 kB
```

Here, `Hugepagesize: 2048 kB` means the system huge-page size is 2 MB.



Then calculate the number of huge pages to configure as follows:

```
Huge pages required per machine = memory per card × cards used by the service / huge-page size
```

For example, with 80 GB of DRAM per card, a service using eight cards, and a 2 MB huge-page size, the total number of huge pages required is `80GB * 8 / 2MB = 327680`.



Finally, configure the number of huge pages with the following command (assuming 327680 pages):

```shell
echo 327680 > /proc/sys/vm/nr_hugepages
```

For a non-root user, use:

```shell
echo 327680 | sudo tee /proc/sys/vm/nr_hugepages
```


##### 5.2 Start the MetaService Metadata Management Service on Each Node
Start the MetaService metadata management service on each node, using the actual configuration file path.
```shell
export MMC_META_CONFIG_PATH=/xxx/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```


##### 5.3 Start the Inference Service on Each Node
P0~P4
```shell
nohup python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 1 --dp-rank-start 0 --dp-address <prefill_node_0_ip> --dp-rpc-port 63521 --vllm-start-port 6700 > p0.log 2>&1 &

nohup python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 1 --dp-rank-start 1 --dp-address <prefill_node_0_ip> --dp-rpc-port 63521 --vllm-start-port 6700 > p1.log 2>&1 &

nohup python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 1 --dp-rank-start 2 --dp-address <prefill_node_0_ip> --dp-rpc-port 63521 --vllm-start-port 6700 > p2.log 2>&1 &

nohup python launch_online_dp.py --dp-size 4 --tp-size 8 --dp-size-local 1 --dp-rank-start 3 --dp-address <prefill_node_0_ip> --dp-rpc-port 63521 --vllm-start-port 6700 > p3.log 2>&1 &
```


D0~D4
```shell
nohup python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 2 --dp-rank-start 0 --dp-address <decode_node_0_ip> --dp-rpc-port 63523 --vllm-start-port 6721 > d0.log 2>&1 &

nohup python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 2 --dp-rank-start 2 --dp-address <decode_node_0_ip> --dp-rpc-port 63523 --vllm-start-port 6721 > d1.log 2>&1 &

nohup python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 2 --dp-rank-start 4 --dp-address <decode_node_0_ip> --dp-rpc-port 63523 --vllm-start-port 6721 > d2.log 2>&1 &

nohup python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 2 --dp-rank-start 6 --dp-address <decode_node_0_ip> --dp-rpc-port 63523 --vllm-start-port 6721 > d3.log 2>&1 &
```



##### 5.4 Start the Request Forwarding Script on Any Node

```shell
bash start_proxy.sh
```


### Testing Method

1. Install ais_bench_benchmark.
```shell
pip3 install ais_bench_benchmark[full]
```

2. Configure the aisbench_auto_tools_prefix tool.

Clone the [rayn-zzz/aisbench_auto_tools_prefix](https://github.com/rayn-zzz/aisbench_auto_tools_prefix) repository and configure `config.py` according to its README.

Note: Prefix warmup is enabled for this test, so configure the `POD_INFO` field as shown below:
```python
# Dataset directory path; it must be accessible (use an absolute path).
DATASET_PATH = "/home/hsy/aisbench_auto_tools_prefix/dataset"

# aisbench working directory: absolute path to the benchmark directory created by cloning aisbench.
# Find the location with `pip show ais-bench-benchmark`.
WORK_PATH = "/usr/local/python3.12.13/lib/python3.12/site-packages"

# Model name configured in the service.
MODEL_NAME = "glm-5"
# Model weight path, used to load the tokenizer.
MODEL_PATH = "/home/weights/Eco-Tech/GLM-5.1-w8a8c8-MTP"
# Destination IP for requests.
HOST_IP = "127.0.0.1"
# Destination port for requests.
HOST_PORT = "9000"

## Authentication information
API_KEY = ""

# For steady-state testing, set this field to `stable_stage`.
DEFAULT_PERFORMANCE_TEST = "default_perf"

# Path for saving aisbench output logs.
OUTPUT_DIR = "./outputs/default"

# Node information in the format `["{ip}:{port}"]`.
# Used to query vLLM metrics and calculate the prefix cache hit rate for each DP domain. Defaults to HOST_IP:HOST_PORT if not configured.
# For PD disaggregation, specify the IP addresses of all P nodes and the port for each DP domain.
POD_INFO = ["192.168.41.142:6700", "192.168.41.144:6700", "192.168.41.145:6700", "192.168.41.146:6700"]
```

3. Start the test with the following command.
```shell
python3 aisbench_test.py --input_len 65536 --output_len 1024 --data_num 160 --concurrency 32 --dataset_type prefix_cache --prefix_num 32 --repeat_rate 0.8 --seed $(date +%s) --prefix_test --dp 4
```

Note:

`--seed $(date +%s)` uses the current time as a random seed so test data does not interfere across runs.

`--prefix_test --dp 4` enables prefix warmup and warms each prefix on four prefill DP domains.
