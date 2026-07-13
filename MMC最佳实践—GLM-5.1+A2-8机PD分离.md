

### 概述

[MemCache](https://gitcode.com/Ascend/memcache) 是针对AI推理场景设计的高性能分布式KVCache缓存系统，针对昇腾超节点、服务器的实现了精细化性能调优，提供较好的High Availability能力，同时对接到vllm-ascend、sglang、mindie等开源推理框架。

**GLM-5.1** 是智谱最新旗舰模型，代码能力大大增强，**长程任务**显著提升，能够在单次任务中持续、自主地工作长达 8 小时，完成从规划、执行到迭代优化的完整闭环，交付工程级成果。使用MemCache作为vLLM KV Pool后端，可以实现高效的KV Cache存储和请求间复用，从而大幅优化长文本、高复用（prefix cache）场景下的TTFT和吞吐水平。



### 环境准备

【硬件规格】：A2 * 8

【驱动固件】：最低要求为25.5.1

【模型权重】：GLM-5.1-w8a8

**注意：使用命令 `getconf PAGESIZE` 查询系统页大小，结果为4K方可使用memcache。若查询结果为64K，需要考虑更换OS。**



#### vllm-ascend

使用vllm-ascend官方镜像v0.18.0

```shell
docker pull quay.nju.edu.cn/ascend/vllm-ascend:v0.18.0
```



#### MemCache

```shell
pip install memfabric_hybrid==1.1.1
pip install memcache_hybrid==1.1.1
```

如果因网络原因无法在线安装，可以在 [PyPI · The Python Package Index](https://pypi.org/) 下载安装包离线安装。

安装其他非正式发布版可以直接源码编译安装，参考 [官方编译文档](https://gitcode.com/Ascend/memcache/blob/master/doc/install_run.md) 即可。



### 部署

#### 1. 准备memcache配置文件

MemCache由MetaService和LocalService两个组件构成，其中LocalService集成到推理框架中随推理进程一同启动，负责内存申请、数据读写和传输，MetaService作为独立进程启动，通过TCP与LocalService建立连接，负责元数据管理、数据淘汰、主备切换（高可用等功能）。

启动时MetaService、LocalService的配置由各自配置文件导入，默认配置文件在memcache的whl包config目录下

```
/usr/local/lib/python3.11/site-packages/memcache_hybrid/config
```



##### MetaService 配置文件

```ini
## Meta service start-up url
ock.mmc.meta_service_url = tcp://127.0.0.1:5000               ## IP 需要修改为启动节点IP

## Config store url
ock.mmc.meta_service.config_store_url = tcp://127.0.0.1:6000  ## IP 需要修改为启动节点IP

## HTTP service url
ock.mmc.meta_service.metrics_url = http://127.0.0.1:8000      ## IP 需要修改为启动节点IP

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_config.md)
```



##### LocalService 配置文件

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
ock.mmc.local_service.protocol = device_rdma                          ## 卡侧ROCE网络传输为A2典型配置

## DRAM space usage
ock.mmc.local_service.dram.size = 100GB                               ## 每卡配置100GB内存容量，可根据环境实际内存大小适当调整
ock.mmc.local_service.max.dram.size = 1024GB

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_config.md)
```



#### 2. 准备vllm启动脚本

##### 2.1 服务拉起辅助脚本

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



##### 2.2 vLLM启动脚本

在所有P节点上准备脚本`run_dp_template.sh`

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

在所有D节点准备脚本`run_dp_template.sh`

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

**需要根据环境做相应修改的包括：**

- local_ip：本机IP地址
- nic_name：本地网卡名称
- 权重路径



**相比基线脚本，启用MF时仅涉及以下变化**

1. 添加了环境变量

```shell
# 导入MemCache LocalService配置文件路径（这里给的是whl包默认配置文件路径）
export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
# 设置Python哈子函数随机种子，保证不同实例接收相同请求后生成相同key
export PYTHONHASHSEED=0
```

2. kv-transfer-config从仅使用MooncakeConnectorV1变为同时使用MooncakeConnectorV1+AscendStoreConnector

- MooncakeConnectorV1负责每个请求P=>D传输kvcache
- AscendStoreConnector负责调用memcache读写接口实现请求间prefix cache复用



##### 2.3 请求转发脚本

从vllm-ascend仓库中获取代理程序[load_balance_proxy_server_example.py](https://github.com/vllm-project/vllm-ascend/blob/main/examples/disaggregated_prefill_v1/load_balance_proxy_server_example.py)

使用如下脚本启动转发（需要修改对应ip、port为服务实际启动时所设置的ip、port）

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
        6721 6722 \
```



#### 3. 拉起服务

##### 3.1 在每个节点配置大页内存

宿主机上通过如下命令查询大页配置，获取Hugepagesize，计算需要配置的大页数量

```shell
grep -i huge /proc/meminfo
```

计算方法为

```
total_hugepages_number = dram_size_per_rank * 8 / Hugepagesize
```

例如，每卡配置100GB内存，大页大小为2MB，则总共需要大页数量为 `100*8*1024/2=409600`。

通过如下命令设置大页数量

```shell
echo 409600 > /proc/sys/vm/nr_hugepages
```
如果是非root用户可使用如下命令
```shell
echo 409600 | sudo tee /proc/sys/vm/nr_hugepages
```



##### 3.2 在任一节点拉起MetaService元数据管理服务

```shell
export MMC_META_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```



##### 3.3 在所有节点拉起推理服务

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



##### 3.4 在任一节点拉起请求转发脚本

```shell
bash start_proxy.sh
```


### 测试结果

SGLang 多轮对话测试工具（高子孟）

| MMC     | 并发 | TTFT (s) | 平均耗时 (s) | 平均Decode耗时 (s) | AverageOutputLength | TPOT (ms) | 单路输出TPS | 总命中率 |
|---------|----|----------|----------|----------------|---------------------|-----------|---------|------|
| Disable | 20 | 37.44    | 77.83    | 40.39          | 450.99              | 89.56     | 11.17   |      |
| Disable | 20 | 33.7     | 76.60    | 42.90          | 450.32              | 95.27     | 10.50   |      |
| Disable | 20 | 40.65    | 82.62    | 42.97          | 457.32              | 91.77     | 10.90   |      |
| Disable | 20 | 42.65    | 81.09    | 38.44          | 456.98              | 84.12     | 11.89   | 27%  |
| Enable  | 20 | 15.62    | 37.64    | 22.02          | 457.74              | 48.11     | 20.79   | 88%  |


90%固定命中率（寇希栋）

| 场景 | InputLen | OutputLen | 并发 | TTFT无缓存(ms) | TTFT有缓存(ms) | TTFT降幅 | TPOT无缓存(ms) | TPOT有缓存(ms) |
|------|----------|-----------|------|----------------|----------------|----------|----------------|----------------|
| 32K-20并发 | 32K | 1024 | 20 | 26884.4 | 11426.4 | 57.5% | 24.5 | 25.4 |
| 32K-30并发 | 32K | 1024 | 30 | 36535.3 | 14253.5 | 61.0% | 25.1 | 26.0 |
| 32K-50并发 | 32K | 1024 | 50 | 55251.4 | 16415.4 | 70.3% | 26.0 | 32.9 |
| 32K-100并发 | 32K | 1024 | 100 | 103109.2 | 21360.9 | 79.3% | 26.7 | 42.0 |
| 32K-200并发 | 32K | 1024 | 200 | 197632.2 | 42750.5 | 78.4% | 26.3 | 48.9 |
| 64K-30并发 | 64K | 1024 | 30 | 80304.3 | 13650.8 | 83.0% | 26.1 | 32.0 |
| 64K-50并发 | 64K | 1024 | 50 | 123199.1 | 18483.4 | 85.0% | 27.1 | 40.7 |
| 64K-100并发 | 64K | 1024 | 100 | 220035.0 | 29883.4 | 86.4% | 26.4 | 51.9 |
| 64K-200并发 | 64K | 1024 | 200 | 392632.1 | 72064.3 | 81.7% | 26.7 | 65.9 |
| 130K-512-20并发 | 130K | 512 | 20 | 132383.8 | 23612.2 | 82.2% | 24.3 | 42.0 |
| 130K-1024-30并发 | 130K | 1024 | 30 | 224131.7 | 31314.0 | 86.0% | 24.3 | 33.3 |

### FAQ
1. 如遇到偶现崩溃问题，以补丁方式合入此PR：https://github.com/vllm-project/vllm/pull/34079

2. 由于A2服务器运行GLM-5.1存在显存紧张问题，如遇到测试发现使能MMC后性能反而劣化，可以适当调低gpu-memory-utilization到0.85左右。

