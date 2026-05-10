# 方案介绍
## 背景
在大模型驱动的智能对话和内容生成应用中，如客服机器人或长文档摘要系统，用户往往需要处理大量文本或与模型进行多轮交互。这类场景有着大量重复内容且对响应速度有较高要求。为提升用户体验且控制部署成本，业界常用Prefix-cache技术，将历史数据缓存到HBM显存，以查代算提升性能。当前存在HBM空间有限的挑战，使用大显存推理卡的成本较高。通过HBM+DRAM的分级缓存可以大幅增加KVCache的缓存空间，提升命中率，有效缓解计算资源压力，显著降低TTFT（首token响应时延）。

本文将基于MemCache技术方案，介绍分级缓存的参考实践，帮助读者提高大模型推理服务性能。

# 部署方案配置
## 软件版本

## 典型配置

# 部署指导
## 前置条件
已完成OS、昇腾HDK、Docker安装，并已下载对应大模型的权重文件和测试数据集，集群（多机互联）场景下需完成参数面组网，确保网络互通。

因为有镜像拉取和安装依赖，推荐服务器处于联网环境。

1. 本实践使用Qwen3-32B的int8量化为例进行测试。模型权重可从Hugging Face等社区下载后按照昇腾msmodelslim工具要求，进行W8A8量化。

Hugging Face权重下载链接：https://huggingface.co/Qwen/Qwen3-32B，https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Qwen-32B

2. HDK安装参考链接：https://www.hiascend.com/document/detail/zh/CANNCommunityEdition/850/softwareinst/instg/instg_0005.html?Mode=PmIns&InstallType=local&OS=openEuler

3. 组网参考链接：https://support.huawei.com/enterprise/zh/doc/EDOC1100543351/ebeddbba?idPath=23710424|251366513|254884019|261408772|258915651