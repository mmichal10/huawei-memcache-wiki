## [Home](https://gitcode.com/Ascend/memcache/wiki/Home.md)
# 1. Solution Overview
## 1.1 Background
In LLM-powered conversational and content-generation applications, such as customer-service bots and long-document summarization systems, users often process large volumes of text or interact with models over multiple turns. These workloads contain substantial repeated content and require fast responses. To improve user experience while controlling deployment costs, prefix caching is commonly used to store historical data in HBM and serve it from cache instead of recomputing it. HBM capacity is limited, and inference accelerators with large HBM are expensive. Tiered HBM+DRAM caching can greatly expand KV Cache capacity, increase the hit rate, reduce compute pressure, and significantly lower TTFT (time to first token).

This document presents a reference practice for tiered caching based on MemCache to help improve LLM inference service performance.

# 2. Deployment Configuration
## 2.1 Software Versions

## 2.2 Typical Configuration

# 3. Deployment Guide
## 3.1 Prerequisites
Install the OS, Ascend HDK, and Docker, and download the corresponding model weights and test dataset. For a multi-node cluster, configure the parameter network and ensure connectivity.

The server should have internet access because the process pulls container images and installs dependencies.

1. This practice uses int8-quantized Qwen3-32B as an example. Download model weights from Hugging Face or another community and apply W8A8 quantization according to the Ascend msmodelslim tool requirements.

Hugging Face model weight links: https://huggingface.co/Qwen/Qwen3-32B and https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Qwen-32B

2. For HDK installation, see: https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/softwareinst/instg/instg_0005.html?Mode=PmIns&InstallType=local&OS=openEuler

3. For network configuration, see: https://support.huawei.com/enterprise/zh/doc/EDOC1100543351/ebeddbba?idPath=23710424|251366513|254884019|261408772|258915651

## 3.2 Deployment Workflow Summary
Installation steps:

|Deployment step | Description |
|--|--|
| Step 1: Create a vLLM Ascend container | Pull the vLLM Ascend image and create a container |
| Step 2: Install MemCache | Install MemCache in the container |
| Step 3: Start MetaService | Configure `mmc-meta.conf` and `mmc-local.conf`, then start MetaService |
| Step 4: Start vLLM Ascend with MemCache enabled | Set the relevant environment variables and enable MemCache in the launch command |

## 3.2 Detailed Deployment Steps
### 3.3.1 Create a vLLM Ascend Container
This section describes deploying vLLM Ascend in a container. Images are available at https://quay.io/repository/ascend/vllm-ascend?tab=tags

For other installation methods (such as installing with Python), see the official installation guide: https://docs.vllm.ai/projects/ascend/zh-cn/v0.13.0/installation.html#set-up-using-docker

For both single-node and two-node deployments, follow these steps:
```
# Pull the image with Docker.
# Alternatively, download the image tarball and load it with `docker load -i xx.tar`.
docker pull quay.io/ascend/vllm-ascend:releases-v0.13.0-openeuler

# Replace this with the version of the image you downloaded.
export IMAGE=quay.io/ascend/vllm-ascend:releases-v0.13.0-openeuler

# List the NPU device IDs.
ls /dev/davinci*
# Set the container device parameters using the discovered davinci device IDs.
/dev/davinci0  /dev/davinci1  /dev/davinci2  /dev/davinci3  /dev/davinci4  /dev/davinci5  /dev/davinci6  /dev/davinci7  /dev/davinci_manager

# Create the container. Replace the model weight mount path and NPU device list with actual values (this example uses eight cards).
docker run -dit \
    --name vllm-ascend-memcache \# Set as appropriate for your environment.
    --shm-size=500g \
    --net=host \
    --device /dev/davinci0 \# Set as appropriate; remove the comment before running, with no extra spaces.
    --device /dev/davinci1 \
    --device /dev/davinci2 \
    --device /dev/davinci3 \
    --device /dev/davinci4 \
    --device /dev/davinci5 \
    --device /dev/davinci6 \
    --device /dev/davinci7 \
    --device /dev/davinci_manager \
    --device /dev/devmm_svm \
    --device /dev/hisi_hdc \
    -v /usr/local/dcmi:/usr/local/dcmi \
    -v /usr/local/Ascend/driver/tools/hccn_tool:/usr/local/Ascend/driver/tools/hccn_tool \
    -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi \
    -v /usr/local/Ascend/driver/lib64/:/usr/local/Ascend/driver/lib64/ \
    -v /usr/local/Ascend/driver/version.info:/usr/local/Ascend/driver/version.info \
    -v /etc/ascend_install.info:/etc/ascend_install.info \
    -v /dev/hugepages:/dev/hugepages \# Mount huge-page memory.
    -v /data/weights:/root/.cache \# Replace as appropriate; remove the comment before running, with no extra spaces.
    -it $IMAGE bash

# Enter the container.
docker exec -ti vllm-ascend-memcache bash
```
### 3.3.2 Install MemCache
`pip install memcache_hybrid`
For other installation methods, see [Installation](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/install_whl.md).

### 3.3.3 Configure the Files and Start MetaService
The MemCache feature requires two configuration files (`mmc-meta.conf` and `mmc-local.conf`). Configure `mmc-meta.conf` before starting MetaService and `mmc-local.conf` before starting vLLM Ascend.

In a two-node deployment, use identical configuration file contents on both nodes and start MetaService only on the primary node.

1. Enter the container:
```
# Enter the container; set the container name as appropriate.
docker exec -ti vllm-ascend-memcache bash
```

2. Create the configuration files:
`mmc-meta.conf` configuration file
```
# For a two-node deployment, set the following IP address to the primary node's IP on both nodes.
# The port can be changed as needed; update `mmc-local.conf` to use the same value.
ock.mmc.meta_service_url = tcp://xx.xx.xx.xx:5000
ock.mmc.meta_service.config_store_url = tcp://xx.xx.xx.xx:6000
ock.mmc.meta_service.metrics_url = http://xx.xx.xx.xx:8000
ock.mmc.log_level = info
```
`mmc-local.conf` configuration file; set the number of NPU cards and the amount of DRAM used by MemCache as needed.
```
# The IP address and port below must match those in `mmc-meta.conf`.
ock.mmc.meta_service_url = tcp://xx.xx.xx.xx:5000
ock.mmc.local_service.config_store_url = tcp://xx.xx.xx.xx:6000
ock.mmc.log_level = info

# Set the total number of NPU cards (for multiple nodes, set the total across all devices).
ock.mmc.local_service.world_size = 256

# For a single-node 300I A2 deployment, set to `host_shm`.
# For a single- or two-node 800I A2 deployment, set to `device_rdma`.
ock.mmc.local_service.protocol = device_rdma

# Memory allocated per card. At 128 GB per card, a two-card service consumes 256 GB of system memory in total.
# Reserve enough memory for system applications; the remaining memory can be allocated to MemCache.
ock.mmc.local_service.dram.size = 128GB
```

For additional configuration parameters, see [Configuration](url).

3. Start the MetaService (in a two-node deployment, start it only on the primary node).
```
# Start MetaService using the Python script.
python3 metaservice.py
```
The script (`metaservice.py`) is as follows:
```
from memcache_hybrid import MetaService
MetaService.main()
```
For other ways to start it, see [Starting MetaService](url).

### 3.3.4 Start vLLM Ascend with MemCache Enabled
#### Scenario 1: Single-node service
To start a single-node vLLM Ascend service, add the parameters that enable MemCache. Keep existing workload configuration (such as device IDs and model weights) unchanged.
```
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/nnal/atb/set_env.sh

export MMC_LOCAL_CONFIG_PATH=/usr/local/memcache_hybrid/latest/config/mmc-local.conf
export VLLM_USE_V1=1
export ASCEND_RT_VISIBLE_DEVICES=2,3 # NPU device IDs to use
export ACL_OP_INIT_MODE=1
export PYTORCH_NPU_ALLOC_CONF="expandable_segments:True"
export PYTHONHASHSEED=0
export HCCL_BUFFSIZE=1024

python -m vllm.entrypoints.openai.api_server \
  --model=/root/.cache/DeepSeek-R1-Distill-Qwen-32B-w8a8 \# Replace with the actual model weight path.
  --served-model-name qwen3 \
  --trust-remote-code \
  --async-scheduling \
  --quantization ascend \
  --distributed-executor-backend mp \
  --tensor-parallel-size 2 \
  --port 8113 \
  --max-model-len 32K \
  --compilation_config '{"cudagraph_mode":"FULL_DECODE_ONLY"}' \
  --additional_config='{"ascend_scheduler_config":{"enabled":false}, "enable_shared_expert_dp":false, "chunked_prefill_for_mla":true}' \
  --gpu-memory-utilization 0.90 \
  --kv-transfer-config \# Enable the MemCache feature.
  '{
      "kv_connector": "AscendStoreConnector",
      "kv_role": "kv_both",
      "kv_connector_extra_config": {
        "backend": "memcache",
        "mooncake_rpc_port":"0"
      }
  }' > log_vllm_memcache.log 2>&1
```
In the runtime logs of vLLM Ascend with MemCache enabled, the `External prefix cache hit rate` keyword indicates the MemCache hit rate. HBM hits take priority; this rate counts cases where HBM misses but DRAM hits.

When enabling MemCache for multiple services on one node, keep `mmc-meta.conf` and `mmc-local.conf` identical across services. Start one MetaService, then start multiple inference services.
#### Scenario 2: Two-node service
To enable MemCache for a two-node DeepSeek-V3 service, start MetaService on the primary node and add the MemCache parameters to the existing two-node launch commands.

For the two-node launch command, see the official documentation: https://docs.vllm.ai/projects/ascend/zh-cn/v0.13.0/tutorials/DeepSeek-V3.1.html

Prepare the model weights as described in the prerequisites and convert them using modelslim.

The following is an example command for a two-node DeepSeek deployment. Assume the primary node IP is xx.xx.xx.0 and its interface is xxx0; the standby node IP is xx.xx.xx.1 and its interface is xxx1.

Primary node launch command:
```
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/nnal/atb/set_env.sh

export MMC_LOCAL_CONFIG_PATH=/usr/local/memcache_hybrid/latest/config/mmc-local.conf

# Obtain the following parameters with `ifconfig`.
# `nic_name` is the interface name corresponding to `local_ip` (for example, enp***); `local_ip` is this node's communication IP.
nic_name="xxx0"
local_ip="xx.xx.xx.0"

# Set `node0_ip` to the primary node IP.
node0_ip="xx.xx.xx.0"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTHONHASHSEED=0
export HCCL_BUFFSIZE=1024
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_USE_V1=1

vllm serve /root/.cache/DeepSeek-V3.1-w8a8-mtp-QuaRot \
  --host 0.0.0.0 \
  --port 8004 \
  --enforce-eager \
  --data-parallel-size 2 \
  --data-parallel-size-local 1 \
  --api-server-count 2 \
  --data-parallel-address $node0_ip \
  --data-parallel-rpc-port 13389  \
  --tensor-parallel-size 8 \
  --seed 1024 \
  --served-model-name deepseek \
  --max-model-len 65536 \
  --max-num-batched-tokens 16384 \
  --trust-remote-code \
  --gpu-memory-utilization 0.9 \
  --quantization ascend \
  --max-num_seqs 20 \
  --enable-expert-parallel \
  --no-enable-prefix-caching \
  --additional_config='{"ascend_scheduler_config":{"enabled":false}, "enable_shared_expert_dp":false, "chunked_prefill_for_mla":true}' \
  --kv-transfer-config \
  '{
        "kv_connector": "AscendStoreConnector",
        "kv_role": "kv_both",
        "kv_connector_extra_config": {
                "backend": "memcache",
                "mooncake_rpc_port":"0"
           }
  }' > log_hunbu_0.log 2>&1
```
Standby node launch command:
```
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/Ascend/nnal/atb/set_env.sh

export MMC_LOCAL_CONFIG_PATH=/usr/local/memcache_hybrid/latest/config/mmc-local.conf

# Obtain the following parameters with `ifconfig`.
# `nic_name` is the interface name corresponding to `local_ip` (for example, enp***); `local_ip` is this node's communication IP.
nic_name="xxx1"
local_ip="xx.xx.xx.1"

# Set `node0_ip` to the primary node IP.
node0_ip="xx.xx.xx.0"

export HCCL_IF_IP=$local_ip
export GLOO_SOCKET_IFNAME=$nic_name
export TP_SOCKET_IFNAME=$nic_name
export HCCL_SOCKET_IFNAME=$nic_name
export OMP_PROC_BIND=false
export OMP_NUM_THREADS=10
export PYTHONHASHSEED=0
export HCCL_BUFFSIZE=1024
export PYTORCH_NPU_ALLOC_CONF=expandable_segments:True
export VLLM_USE_V1=1

vllm serve /weights/DeepSeek-V3.1-w8a8-mtp-QuaRot \
  --host 0.0.0.0 \
  --port 8004 \
  --headless  \
  --enforce-eager \
  --data-parallel-size 2 \
  --data-parallel-size-local 1 \
  --data-parallel-start-rank 1 \
  --data-parallel-address $node0_ip \
  --data-parallel-rpc-port 13389  \
  --tensor-parallel-size 8 \
  --seed 1024 \
  --served-model-name deepseek \
  --max-model-len 65536 \
  --max-num-batched-tokens 16384 \
  --trust-remote-code \
  --gpu-memory-utilization 0.9 \
  --quantization ascend \
  --max-num_seqs 20 \
  --enable-expert-parallel \
  --no-enable-prefix-caching \
  --additional_config='{"ascend_scheduler_config":{"enabled":false}, "enable_shared_expert_dp":false, "chunked_prefill_for_mla":true}' \
  --kv-transfer-config \
   '{
        "kv_connector": "AscendStoreConnector",
        "kv_role": "kv_both",
        "kv_connector_extra_config": {
                "backend": "memcache",
                "mooncake_rpc_port":"0"
           }
  }' > log_hunbu_1.log 2>&1
```
