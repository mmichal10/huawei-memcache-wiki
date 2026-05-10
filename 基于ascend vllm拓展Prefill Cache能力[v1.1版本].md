# 1.方案介绍
## 1.1背景
在大模型驱动的智能对话和内容生成应用中，如客服机器人或长文档摘要系统，用户往往需要处理大量文本或与模型进行多轮交互。这类场景有着大量重复内容且对响应速度有较高要求。为提升用户体验且控制部署成本，业界常用Prefix-cache技术，将历史数据缓存到HBM显存，以查代算提升性能。当前存在HBM空间有限的挑战，使用大显存推理卡的成本较高。通过HBM+DRAM的分级缓存可以大幅增加KVCache的缓存空间，提升命中率，有效缓解计算资源压力，显著降低TTFT（首token响应时延）。

本文将基于MemCache技术方案，介绍分级缓存的参考实践，帮助读者提高大模型推理服务性能。

# 2.部署方案配置
## 2.1软件版本

## 2.2典型配置

# 3.部署指导
## 3.1前置条件
已完成OS、昇腾HDK、Docker安装，并已下载对应大模型的权重文件和测试数据集，集群（多机互联）场景下需完成参数面组网，确保网络互通。

因为有镜像拉取和安装依赖，推荐服务器处于联网环境。

1. 本实践使用Qwen3-32B的int8量化为例进行测试。模型权重可从Hugging Face等社区下载后按照昇腾msmodelslim工具要求，进行W8A8量化。

Hugging Face权重下载链接：https://huggingface.co/Qwen/Qwen3-32B，https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Qwen-32B

2. HDK安装参考链接：https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/softwareinst/instg/instg_0005.html?Mode=PmIns&InstallType=local&OS=openEuler

3. 组网参考链接：https://support.huawei.com/enterprise/zh/doc/EDOC1100543351/ebeddbba?idPath=23710424|251366513|254884019|261408772|258915651

## 3.2 部署流程汇总
安装步骤：

|安装部署  |步骤说明  |
|--|--|
| 步骤1：创建vLLM Ascend容器 |拉取vLLM Ascend镜像，创建容器  | 
| 步骤2：安装MemCache |容器内安装MemCache  | 
| 步骤3：拉起MetaService |配置mmc-meta.conf+mmc-local.conf文件，拉起MetaService | 
| 步骤4：拉起开启MemCache特性的vLLM Ascend服务 |拉起命令中，配置相关环境变量并配置开启MemCache | 

## 3.2  详细部署操作
### 3.3.1 创建vLLM Ascend容器
本文介绍使用容器部署vLLM Ascend的方式，镜像下载地址：https://quay.io/repository/ascend/vllm-ascend?tab=tags

若需采用其它安装方式（使用Python安装），请参考官方安装指导：https://docs.vllm.ai/projects/ascend/zh-cn/v0.13.0/installation.html#set-up-using-docker

单机部署和双机部署，均参考如下操作步骤：
```
# docker拉取镜像
# 也可以采用下载镜像的tar包，然后使用docker load -i xx.tar的方式安装镜像
docker pull quay.io/ascend/vllm-ascend:releases-v0.13.0-openeuler

# 根据实际下载镜像版本进行替换
export IMAGE=quay.io/ascend/vllm-ascend:releases-v0.13.0-openeuler

# 查看NPU设备号
ls /dev/davinci*
# 根据查询到的davinci设备号，配置容器创建时的device参数
/dev/davinci0  /dev/davinci1  /dev/davinci2  /dev/davinci3  /dev/davinci4  /dev/davinci5  /dev/davinci6  /dev/davinci7  /dev/davinci_manager

# 创建容器，注意权重挂载路径根据真实情况替换，NPU卡数据根据实际情况替换（此处以8卡为例）
docker run -dit \
    --name vllm-ascend-memcache \# 依据实际情况配置
    --shm-size=500g \
    --net=host \
    --device /dev/davinci0 \# 依据实际情况配置，执行时删除注释，不要有空格
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
    -v /dev/hugepages:/dev/hugepages \# 挂载大页内存
    -v /data/weights:/root/.cache \# 根据真实情况替换，执行时删除注释，不要有空格
    -it $IMAGE bash

# 进入容器
docker exec -ti vllm-ascend-memcache bash
```

