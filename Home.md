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
   #### [基于ascend vllm拓展Prefill Cache能力](https://gitcode.com/Ascend/memcache/wiki/MemCache%20%E6%94%AF%E6%8C%81%E6%8E%A8%E7%90%86%E5%A4%9A%E5%AE%9E%E4%BE%8B%E5%85%B1%E6%B1%A0%E7%9A%84%E6%A1%88%E4%BE%8B.md)
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

