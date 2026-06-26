### 概述

[MemCache](https://gitcode.com/Ascend/memcache) 是针对AI推理场景设计的高性能分布式KVCache缓存系统，针对昇腾超节点、服务器的实现了精细化性能调优，提供较好的High Availability能力，同时对接到vllm-ascend、sglang、mindie等开源推理框架。



DeepSeek-V4 拥有百万字超长上下文，在 Agent 能力、世界知识和推理性能上均实现国内与开源领域的领先。相比 DeepSeek-V4-Pro，DeepSeek-V4-Flash 在世界知识储备方面稍逊一筹，但展现出了接近的推理能力。而由于模型参数和激活更小，相较之下 V4-Flash 能够提供更加快捷、经济的 API 服务。使用MemCache作为vLLM KV Pool后端，可以实现高效的KV Cache存储和请求间复用，从而大幅优化长文本、高复用（prefix cache）场景下的TTFT和吞吐水平。



### 环境准备

【硬件规格】：A2 * 1

【驱动固件】：25.5.3（最低要求为25.5.1）

【模型权重】：DeepSeek-V4-Flash-w8a8-mtp



#### vllm-ascend

拉取vllm-ascend官方镜像

```shell
docker pull quay.nju.edu.cn/ascend/vllm-ascend:v0.20.2rc1
```

启动容器的命令可参考 [vllm-ascend 部署 DeepSeek-V4-Flash 官方指导文档](https://docs.vllm.ai/projects/ascend/zh-cn/latest/tutorials/models/DeepSeek-V4-Flash.html) 。

升级transformers：

```shell
pip install --upgrade transformers
```

由于目前vllm-ascend支持DSV4池化特性处于POC状态（正式支持时间为630），基于vllm-ascend官方镜像v0.20.2rc1，需要将以下PR中修改以补丁形式合并到v0.20.2rc1解决一些问题：
<span style="color:#e60000;">注意，部分补丁未合入主线，需要等待正式方案，此案例为尝鲜试用，正式版本等待后续刷新</span>

| 标题                                           | 链接                                                   | 概述                       |
| ---------------------------------------------- | ------------------------------------------------------ | -------------------------- |
| Rehash AscendStore grouped keys                | https://github.com/vllm-project/vllm-ascend/pull/10194 | 解决External Cache无命中   |
| Align AscendStore grouped hash lookup encoding | https://github.com/vllm-project/vllm-ascend/pull/10217 | 解决External Cache无命中   |
| Avoid logging KV store keys at error level     | https://github.com/vllm-project/vllm-ascend/pull/10329 | 解决报错时key刷屏问题      |
| Support AscendStore load masks                 | https://github.com/vllm-project/vllm-ascend/pull/10436 | 解决读取数据过多导致无收益 |
| Support AscendStore retention masks for v0.20.2     | https://github.com/vllm-project/vllm-ascend/pull/10439 | 减少读写的cache数据 |

~~其中PR10439可以替代PR10436，同时减少读写的数据，可以支持存储更多的cache，提升端到端收益。~~<span style="color:#e60000;">注意：此PR暂不稳定，不要用于生产环境</span>

可以基于vllm-ascend代码仓v0.20.2rc1 tag拉取分支，并将4个PR的commit通过cherry-pick方式合入，然后通过替换镜像中vllm-ascend代码文件的方式部署。可以参考分支：[SanyueHan/vllm-ascend at v0.20.2rc1](https://github.com/SanyueHan/vllm-ascend/tree/v0.20.2rc1)。或者直接拉取vllm-ascend代码仓的v0.20.2rc分支再将PR10439或者PR10436通过cherry-pick方式合入，后续替换文件或者重新安装vllm_ascend部署。




#### memcache

```shell
pip install memfabric_hybrid==1.1.2   （发布中、可以编译release/1.1分支）
pip install memcache_hybrid==1.1.2     (发布中、可以编译release/1.1分支）
```

如果因网络原因无法在线安装，可以在 [PyPI · The Python Package Index](https://pypi.org/) 下载安装包离线安装。

安装其他非正式发布版可以直接源码编译安装，参考 [官方编译文档](https://gitcode.com/Ascend/memcache/blob/master/doc/install_run.md) 即可。



### 运行

#### 1. 修改 Memcache 配置文件

MemCache由MetaService和LocalService两个组件构成，其中LocalService集成到推理框架中随推理进程一同启动，负责内存申请、数据读写和传输，MetaService作为独立进程启动，通过TCP与LocalService建立连接，负责元数据管理、数据淘汰、主备切换（高可用等功能）。

启动时MetaService、LocalService的配置由各自配置文件导入，默认配置文件在memcache的whl包config目录下

```
/usr/local/lib/python3.11/site-packages/memcache_hybrid/config
```



##### MetaService 配置文件

```ini
## Meta service start-up url
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## Config store url
ock.mmc.meta_service.config_store_url = tcp://127.0.0.1:6000

## HTTP service url
ock.mmc.meta_service.metrics_url = http://127.0.0.1:8000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_config.md)
```



##### LocalService 配置文件

```ini
## Meta service start-up url
## Keep consistent with the same name configuration in mmc-meta.conf
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## Config store url
## Keep consistent with the same name configuration in mmc-meta.conf
ock.mmc.local_service.config_store_url = tcp://127.0.0.1:6000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

## Data transfer protocol options
ock.mmc.local_service.protocol = host_shm

## DRAM space usage
ock.mmc.local_service.dram.size = 50GB

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_config.md)
```



#### 2. 准备vLLM启动脚本

```shell
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export LD_PRELOAD=/usr/lib/aarch64-linux-gnu/libjemalloc.so.2:$LD_PRELOAD
export HCCL_BUFFSIZE=1024
export VLLM_ASCEND_APPLY_DSV4_PATCH=1
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1
export TASK_QUEUE_ENABLE=1
export HCCL_OP_EXPANSION_MODE="AIV"

# 合并PR10439后需设置此环境变量
export VLLM_PREFIX_CACHE_RETENTION_INTERVAL=0

export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf

vllm serve /home/weights/DeepSeek-V4-Flash-w8a8-mtp \
    --port 8900 \
    --max_model_len 133120 \
    --max-num-batched-tokens 8192 \
    --served-model-name dsv4 \
    --gpu-memory-utilization 0.9 \
    --block-size 128 \
    --max-num-seqs 32 \
    --data-parallel-size 1 \
    --tensor-parallel-size 8 \
    --enable-expert-parallel \
    --tokenizer-mode deepseek_v4 \
    --tool-call-parser deepseek_v4 \
    --enable-auto-tool-choice \
    --reasoning-parser deepseek_v4 \
    --safetensors-load-strategy 'prefetch' \
    --model-loader-extra-config='{"enable_multithread_load": "true", "num_threads": 128}' \
    --quantization ascend \
    --speculative-config '{"num_speculative_tokens": 1, "method": "mtp", "enforce_eager": true}' \
    --compilation-config '{"cudagraph_mode": "FULL_DECODE_ONLY"}'\
    --async-scheduling \
    --no-disable-hybrid-kv-cache-manager \
    --kv-transfer-config \
    '{
        "kv_connector": "AscendStoreConnector",
        "kv_role": "kv_both",
        "kv_connector_extra_config": {
            "backend": "memcache",
            "mooncake_rpc_port":"0"
        }
    }' \
    --additional-config '
    {
        "ascend_compilation_config": {
            "enable_npugraph_ex": true,
            "enable_static_kernel": false
        },
        "enable_cpu_binding": true,
        "enable_dsa_cp": true,
        "multistream_overlap_shared_expert": true
    }' 2>&1 | tee output.log
```



**相比基线脚本，使能Memcache模式下仅涉及以下变化：**

1. 添加了环境变量导入MemCache LocalService配置文件路径（这里给的是whl包默认配置文件路径）

```shell
export MMC_LOCAL_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-local.conf
```

2. 添加了 no-disable-hybrid-kv-cache-manager 和 kv-transfer-config

```shell
--no-disable-hybrid-kv-cache-manager \
--kv-transfer-config \
    '{
        "kv_connector": "AscendStoreConnector",
        "kv_role": "kv_both",
        "kv_connector_extra_config": {
            "backend": "memcache"
        }
    }'
```



#### 3. 拉起服务

首先需要拉起MetaService元数据管理服务

```shell
export MMC_META_CONFIG_PATH=/usr/local/lib/python3.11/site-packages/memcache_hybrid/config/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```

然后使用脚本拉起推理服务



#### 4. 测试

本文使用vLLM官方多轮会话测试工具 [vllm/benchmarks/multi_turn/README.md at main · vllm-project/vllm](https://github.com/vllm-project/vllm/blob/main/benchmarks/multi_turn/README.md)

在generate_multi_turn.json中可以配置对话数量、轮数、每轮输入输出等关键参数，这里以下面第一组数据为例

```json
{
    "filetype": "generate_conversations",
    "num_conversations": 10,
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

```python3
python benchmark_serving_multi_turn.py \
    --model /home/weights/DeepSeek-V4-Flash-w8a8-mtp \
    --served-model-name dsv4 \
    --input-file generate_multi_turn.json \
    --num-clients 10 \
    --max-active-conversations 10 \
    --no-early-stop \
    --request-timeout-sec 600 \
    -u http://127.0.0.1:8900
```



### 测试结果

| MF      | 轮数 | 每轮输入 | 每轮输出 | 并发 | 总会话数 | mean ttft (ms) | mean tpot (ms) | 总耗时 (s) | HBM命中率 | DRAM命中率 |
| ------- | ---- | -------- | -------- | ---- | -------- | -------------- | -------------- | ---------- | --------- | ---------- |
| disable | 20   | 9216     | 1024     | 10   | 10       | 8775.00        | 71.75          | 830.846    | 0.0%      | /          |
| enable  | 20   | 9216     | 1024     | 10   | 10       | 4542.21        | 54.50          | 620.689    | 0.0%      | 72.5%      |