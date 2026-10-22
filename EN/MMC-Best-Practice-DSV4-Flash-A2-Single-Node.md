### Overview

[MemCache](https://gitcode.com/Ascend/memcache) is a high-performance distributed KV Cache system designed for AI inference. It is finely tuned for Ascend supernodes and servers, provides strong high-availability capabilities, and integrates with open-source inference frameworks such as vllm-ascend, sglang, and mindie.



DeepSeek-V4 supports million-character long contexts and leads domestic and open-source models in agent capabilities, world knowledge, and reasoning performance. Compared with DeepSeek-V4-Pro, DeepSeek-V4-Flash has somewhat less world knowledge but delivers similar reasoning capability. Its smaller model and activation sizes allow V4-Flash to provide faster, more economical API services. Using MemCache as the vLLM KV Pool backend enables efficient KV Cache storage and reuse across requests, substantially improving TTFT and throughput for long-text workloads with high prefix-cache reuse.



### Environment

**Hardware:** A2 × 1

**Model weights:** [DeepSeek-V4-Flash-w8a8-mtp](https://www.modelscope.cn/models/Eco-Tech/DeepSeek-V4-Flash-w8a8-mtp)


### Deployment


#### 1. Pull the official vllm-ascend image

```shell
docker pull quay.io/ascend/vllm-ascend:v0.23.0rc1
```

For the container launch command, see the [official guide for deploying DeepSeek-V4-Flash with vllm-ascend](https://docs.vllm.ai/projects/ascend/zh-cn/latest/tutorials/models/DeepSeek-V4-Flash.html).


#### 2. Install MemCache

The official vllm-ascend image already includes MemCache and its dependencies. Check the installed versions with:
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



#### 3. Edit the MemCache configuration files

MemCache consists of MetaService and LocalService. LocalService is integrated with the inference framework and starts with the inference process; it handles memory allocation, data reads/writes, and transfers. MetaService runs as a separate process, connects to LocalService over TCP, and handles metadata management, data eviction, primary/standby switching, and other high-availability functions.

At startup, MetaService and LocalService load their settings from separate configuration files. By default, these files are in the config directory of the MemCache wheel package:

```
/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/
```



##### MetaService configuration file

```ini
## Meta service start-up url
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## Config store url
ock.mmc.meta_service.config_store_url = tcp://127.0.0.1:6000

## HTTP service url
ock.mmc.meta_service.metrics_url = http://127.0.0.1:8000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md)
```



##### LocalService configuration file

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
## Set the per-card DRAM size according to your environment
ock.mmc.local_service.dram.size = 80GB

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_config.md)
```



#### 4. Prepare the vLLM launch script

```shell
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export LD_PRELOAD=/usr/lib/aarch64-linux-gnu/libjemalloc.so.2:$LD_PRELOAD
export HCCL_BUFFSIZE=1024
export VLLM_ASCEND_ENABLE_FLASHCOMM1=1
export TASK_QUEUE_ENABLE=1
export HCCL_OP_EXPANSION_MODE="AIV"

# This environment variable can reduce some KV Cache reads and writes and may improve performance in certain scenarios, but it can also affect prefix hit rate. Benchmark and set it as needed.
# Leaving it unset is the default behavior. Setting it to 0 is the most aggressive policy. Other values can be multiples of the minimum number of tokens that can hit; for the DSV4 model with block_size=128, use a multiple of 16384.
# For vLLM environment variables, see https://docs.vllm.com.cn/en/latest/configuration/env_vars/?h=environment
export VLLM_PREFIX_CACHE_RETENTION_INTERVAL=0

export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf

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
        },
        "kv_load_failure_policy": "recompute"
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



**Compared with the baseline script, enabling MemCache mode requires only the following changes:**

1. Add an environment variable that points to the MemCache LocalService configuration file (the path shown here is the wheel package's default configuration file).

```shell
export MMC_LOCAL_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-local.conf
```

2. Add `kv-transfer-config`.

```shell
--kv-transfer-config \
    '{
        "kv_connector": "AscendStoreConnector",
        "kv_role": "kv_both",
        "kv_connector_extra_config": {
            "backend": "memcache"
        },
        "kv_load_failure_policy": "recompute"
    }'
```



#### 5. Start the service

##### 5.1 Configure huge pages on each node

The service can start with host_shm without huge pages, but using 4 KB pages performs much worse than static huge pages. Static huge pages are recommended for best performance.

Run the following command on the host to inspect the huge-page configuration, obtain Hugepagesize, and calculate the number of huge pages to configure.

```shell
grep -i huge /proc/meminfo
```

Calculate it as follows:

```
total_hugepages_number = dram_size_per_rank * 8 / Hugepagesize
```

For example, with 100 GB per card and a 2 MB huge-page size, the total number of huge pages required is `100*8*1024/2=409600`.

Set the number of huge pages with:

```shell
echo 409600 > /proc/sys/vm/nr_hugepages
```
For a non-root user, use:
```shell
echo 409600 | sudo tee /proc/sys/vm/nr_hugepages
```

**Tip: When running in a container or pod, ensure that the `/dev/hugepages/` directory is mounted inside the container.**


##### 5.2 Start the MetaService metadata management service

```shell
export MMC_META_CONFIG_PATH=/usr/local/python3.12.13/lib/python3.12/site-packages/memcache_hybrid/config/mmc-meta.conf
python3 -c "from memcache_hybrid import MetaService; MetaService.main()"
```

##### 5.3 Start the inference service with the script


### Testing


Validation with the aisbench prefix testing tool:
Case 1: 50% prefix; 40K input tokens; 200 output tokens; concurrency 32; 128 requests total, with 32 distinct prefixes shared by every four requests.

Results:

| MemCache enabled | mean ttft(ms) | mean tpot(ms) | HBM hit rate | DRAM hit rate |
|--|--|--|--|--|
| No | 24735 | 226.9 | 7.46% | / |
| Yes | 11696 | 238.3 | 7.11% | 35.13% |


Case 2: 90% prefix; 128K input tokens; 1 output token; concurrency 16; 64 requests total, with 16 distinct prefixes shared by every four requests.

Results:

| MemCache enabled | mean ttft(ms) | mean tpot(ms) | HBM hit rate | DRAM hit rate |
|--|--|--|--|--|
| No | 109819 | / | 0% | / |
| Yes | 34991 | / | 0% | 87.5% |
