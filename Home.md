# 欢迎使用 MemCache 插件
MemCache插件是一个由社区维护的高性能分布式KVCache缓存插件，用于在昇腾超节点、服务器上提供较好的High Availability能力。
该插件是Ascend社区支持更大规模的Prefix-cache的推荐方案，提供了基于KVCache缓存精细化性能调优能力，同时对接到vllm-ascend、sglang、mindie等开源推理框架。

# 文档

## 快速开始
### 快速安装
   #### 先决条件
   - 操作系统：Linux
   -  Python: 3.11 以上
   #### 安装
   `pip install memcache_hybrid`
### 部署
   #### A2系列产品，芯片型号910B
   #### A3系列产品，芯片型号910C
   #### K5系列产品
### 示例
   #### [基于ascend vllm拓展Prefill Cache能力](https://gitcode.com/Ascend/memcache/wiki/%E5%9F%BA%E4%BA%8Eascend%20vllm%E6%8B%93%E5%B1%95Prefill%20Cache%E8%83%BD%E5%8A%9B[v1.1%E7%89%88%E6%9C%AC].md)
   #### 基于sglang拓展Prefill Cache能力
   #### 基于mindie拓展Prefill Cache能力
## 用户指南
### 特性指南
   #### 支持的模型
   #### 支持的功能
### 配置指南
   #### 环境变量
   #### mmc-meta配置文件
   #### mmc-local配置文件
### 部署指南
   #### 在Kubernetes 上部署
### 版本说明
   #### 1.1版本
   #### 1.0.8版本

## 开发者指南
### 设计文档
### 性能与调试
### API参考
### 单元测试

## 社区
### 治理
### 版本管理策略

