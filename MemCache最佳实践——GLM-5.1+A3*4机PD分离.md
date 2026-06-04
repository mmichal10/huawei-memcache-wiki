### 概述

[MemCache](https://gitcode.com/Ascend/memcache) 是针对AI推理场景设计的高性能分布式KVCache缓存系统，针对昇腾超节点、服务器的实现了精细化性能调优，提供较好的High Availability能力，同时对接到vllm-ascend、sglang、mindie等开源推理框架。

**GLM-5.1** 是智谱最新旗舰模型，代码能力大大增强，**长程任务**显著提升，能够在单次任务中持续、自主地工作长达 8 小时，完成从规划、执行到迭代优化的完整闭环，交付工程级成果。使用MemCache作为vLLM KV Pool后端，可以实现高效的KV Cache存储和请求间复用，从而大幅优化长文本、高复用（prefix cache）场景下的TTFT和吞吐水平。



### 部署

本文采用的部署方案基于 [vllm-ascend部署GLM-5.1官方指导文档](https://docs.vllm.ai/projects/vllm-ascend-cn/zh-cn/latest/tutorials/models/GLM5.html) 修改

【硬件配置】Atlas 900 A3 SuperPoD 四机64卡

【NPU驱动固件】25.5.1

【CANN软件】CANN 8.5.1

【推理框架】vLLM

【部署规格】2P2D （Prefill TP16 DP2，Decode TP4 DP8）

```
torch                                    2.9.0
torch_npu                                2.9.0.post1+gitee7ba04
torchaudio                               2.9.0
torchvision                              0.24.0
transformers                             5.9.0
triton-ascend                            3.2.0.dev20260322
vllm                                     0.18.0+empty
vllm_ascend                              0.18.0rc2.dev100+g9a0b786f2
```



#### 1. 安装MF

在线安装：

```shell
pip install memfabric_hybrid==1.1.1
pip install memcache_hybrid==1.1.1
```

如果因网络原因无法在线安装，可以在 [PyPI · The Python Package Index](https://pypi.org/) 下载安装包离线安装。

安装其他非正式发布版可以直接源码编译安装，参考官方文档即可

[memcache/doc/install_run.md](https://gitcode.com/Ascend/memcache/blob/master/doc/install_run.md)



#### 2. 修改配置文件

MemCache由MetaService和LocalService两个组件构成，其中LocalService集成到推理框架中随推理进程一同启动，负责内存申请、数据读写和传输，MetaService作为独立进程启动，通过TCP与LocalService建立连接，负责元数据管理、数据淘汰、主备切换（高可用等功能）。

启动时MetaService、LocalService的配置由各自配置文件导入，默认配置文件在memcache的whl包config目录下

```
/usr/local/lib/python3.11/site-packages/memcache_hybrid/config
```



###### MetaService 配置文件

```ini
## Meta service start-up url
ock.mmc.meta_service_url = tcp://127.0.0.1:5000               ## IP 需要修改为启动节点IP

## Config store url
ock.mmc.meta_service.config_store_url = tcp://127.0.0.1:6000  ## IP 需要修改为启动节点IP

## HTTP service url
ock.mmc.meta_service.metrics_url = http://127.0.0.1:8000      ## IP 需要修改为启动节点IP

## Log level: debug, info, warn, error
ock.mmc.log_level = info
```



###### LocalService 配置文件

```ini
## Meta service start-up url
## Keep consistent with the same name configuration in mmc-meta.conf
ock.mmc.meta_service_url = tcp://127.0.0.1:5000                       ## 配置为MetaService启动节点IP

## Config store url
## Keep consistent with the same name configuration in mmc-meta.conf
ock.mmc.local_service.config_store_url = tcp://127.0.0.1:6000         ## 配置为MetaService启动节点IP

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

## Data transfer protocol options
ock.mmc.local_service.protocol = device_sdma                          ## 超节点内sdma传输为A3典型配置

## DRAM space usage
ock.mmc.local_service.dram.size = 25GB                                ## 每个推理进程提供25GB池化空间，可根据环境情况进行调整，参考https://gitcode.com/Ascend/memfabric_hybrid/wiki/FAQ.md#2-%E8%B0%83%E7%94%A8halmemcreate%E8%BF%94%E5%9B%9E%E9%94%99%E8%AF%AF%E7%A0%81-6
```

更多配置可以参考官方文档 https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_config.md



#### 3. 准备推理服务启动相关脚本

##### 3.1 服务拉起辅助脚本

在每个节点上准备脚本 `launch_online_dp.py`：

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



##### 3.2 vLLM启动脚本

在每个节点上准备脚本 `run_dp_template.sh`

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
                        "prefill": {"dp_size": 2, "tp_size": 16},
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
                        "prefill": {"dp_size": 2, "tp_size": 16},
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
                        "prefill": {"dp_size": 2, "tp_size": 16},
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
                        "prefill": {"dp_size": 2, "tp_size": 16},
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

**需要根据环境做相应修改的包括：**

- local_ip：本机IP地址
- nic_name：本地网卡名称
- 权重路径



**相比基线脚本，启用MF时仅涉及以下两点变化**

1. 添加了环境变量导入MemCache LocalService配置文件路径（这里给的是whl包默认配置文件路径）

```
export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
```

2. kv-transfer-config从仅使用MooncakeConnectorV1变为同时使用MooncakeConnectorV1+AscendStoreConnector

- MooncakeConnectorV1负责每个请求P=>D传输kvcache
- AscendStoreConnector负责调用memcache读写接口实现请求间prefix cache复用



##### 3.3 请求转发脚本

从vllm-ascend仓库中获取代理程序[load_balance_proxy_server_example.py](https://github.com/vllm-project/vllm-ascend/blob/main/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py)

使用如下脚本启动转发（需要修改对应ip、port为服务实际启动时所设置的ip、port）

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



#### 4. 拉起服务

##### 4.1 拉起MetaService服务
首先需要拉起MetaService元数据管理服务（任一节点拉起）

```shell
export MMC_META_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```

##### 4.2 拉起独立进程服务

在每个节点上都要拉起一个独立进程，进行内存提前占用（拉起前建议使用 echo 3 > /proc/sys/vm/drop_caches 命令先清理缓存）

在每个节点上准备脚本 `memcache.py`

```
import time

from memcache_hybrid import DistributedObjectStore


if __name__ == "__main__":
    store = DistributedObjectStore()
    res = store.init(0)
    if res != 0:
        print(f"Failed to init memcache, res = {res}")
        exit(1)
    print("MemCache init success. ")

    while True:
        time.sleep(1)

```

复制一份mmc-local.conf配置文件给独立进程使用
```
cp /usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf /data/
```

修改mmc-local.conf配置文件，将ock.mmc.local_service.dram.size设置为独立进程占用内存空间的大小。
```ini
ock.mmc.local_service.dram.size = 600GB
```

启动独立进程
```
export MMC_LOCAL_CONFIG_PATH=/data/mmc-local.conf

#启动独立进程
python memcache.py
```

##### 4.3 拉起推理服务

然后在每个节点上通过以下命令拉起推理服务

###### P0

```shell
python launch_online_dp.py --dp-size 2 --tp-size 16 --dp-size-local 1 --dp-rank-start 0 --dp-address 141.61.39.233 --dp-rpc-port 10521 --vllm-start-port 6700
```

###### P1

```shell
python launch_online_dp.py --dp-size 2 --tp-size 16 --dp-size-local 1 --dp-rank-start 1 --dp-address 141.61.39.233 --dp-rpc-port 10521 --vllm-start-port 6700
```

###### D0

```shell
python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 4 --dp-rank-start 0 --dp-address 141.61.39.237 --dp-rpc-port 10523 --vllm-start-port 6721
```

###### D1

```shell
python launch_online_dp.py --dp-size 8 --tp-size 4 --dp-size-local 4 --dp-rank-start 4 --dp-address 141.61.39.237 --dp-rpc-port 10523 --vllm-start-port 6721
```

最后拉起请求转发脚本

```shell
bash start_proxy.sh
```



### 测试

推荐使用vLLM官方多轮会话测试工具 [vllm/benchmarks/multi_turn/README.md at main · vllm-project/vllm](https://github.com/vllm-project/vllm/blob/main/benchmarks/multi_turn/README.md)

在generate_multi_turn.json中可以配置对话数量、轮数、每轮输入输出等关键参数，这里以下面第一组数据为例

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

使用如下命令启动测试

```shell
python benchmark_serving_multi_turn.py \
    --model /home/weights/GLM5.1-W8A8 \
    --served-model-name glm-5 \
    --input-file generate_multi_turn.json \
    --num-clients 16 \                       # 并发数
    --max-active-conversations 16 \
    --no-early-stop \
    --request-timeout-sec 600 \
    -u http://<ip>:<port>                    # proxy脚本启动的ip port
```



**测试结果**

|        | MF      | 轮数 | 每轮输入 | 每轮输出 | 并发 | 总会话数 | mean ttft (ms)  | mean tpot (ms) | 总耗时 (s) | HBM命中率 | DRAM命中率 |
| ------ | ------- | ---- | -------- | -------- | ---- | -------- | --------------- | -------------- | ---------- | --------- | ---------- |
| 第一组 | disable | 20   | 9216     | 1024     | 16   | 16       | 48166.97        | 8.61           | 594.513    | 2.8%      | /          |
| 第一组 | enable  | 20   | 9216     | 1024     | 16   | 16       | 27450.59 (↓43%) | 9.92           | 377.931    | 1.6%      | 68.6%      |
| 第二组 | disable | 40   | 4096     | 1024     | 16   | 16       | 37498.54        | 15.53          | 1070.425   | 4.0%      | /          |
| 第二组 | enable  | 40   | 4096     | 1024     | 16   | 16       | 17744.16 (↓52%) | 17.33          | 710.006    | 4.2%      | 82.3%      |
| 第三组 | disable | 20   | 9216     | 1024     | 30   | 30       | 90407.57        | 7.56           | 1048.888   | 0.1%      | /          |
| 第三组 | enable  | 20   | 9216     | 1024     | 30   | 30       | 41756.93 (↓54%) | 8.91           | 519.047    | 0.2%      | 69.2%      |