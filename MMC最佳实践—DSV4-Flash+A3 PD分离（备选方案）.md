### 概述

[MemCache](https://gitcode.com/Ascend/memcache) 是针对AI推理场景设计的高性能分布式KVCache缓存系统，针对昇腾超节点、服务器的实现了精细化性能调优，提供较好的High Availability能力，同时对接到vllm-ascend、sglang、mindie等开源推理框架。

**DeepSeek-V4-Flash-0731** 是 **DeepSeek-V4-Flash** 的官方正式版本，取代了预览版，并大幅增强了智能体（agentic）能力。尽管 DeepSeek-V4-Flash-0731 激活的参数量远小于 DeepSeek-V4-Pro（预览版），但其整体上可与当前最强的闭源模型相媲美。



### 环境说明

【硬件规格】：Atlas 900 A3 SuperPoD 四机64卡

**注意：由于MemCache在A3上的优选方案（基于HCCS传输的多机池化）对交换机CC包、NPU驱动固件和操作系统内核均有要求，某些客户局点难以完成升级，本案例中采用了MemCache备选方案（基于PCIE传输的单机L2池化），消解了底软不兼容的问题。**

【OS内核版本】：无特殊要求

【NPU驱动固件】：无特殊要求

【L1 1520 CC】：无特殊要求



### 测试结果

![memcache_ttft_compare.png](https://raw.gitcode.com/user-images/assets/7672915/919887de-21f6-43e2-8e46-ea8f34f2687f/memcache_ttft_compare.png 'memcache_ttft_compare.png')

【测试工具】aisbench



### 部署

#### 1. 拉取vllm-ascend官方镜像

拉取vllm-ascend官方镜像

```shell
docker pull quay.io/ascend/vllm-ascend:v0.23.0rc1
```

<span style="color:#e60000;">**备注：** 当前v0.23.0rc1版本的vllm-ascend在A3上开启memcache作为内存池化后端时存在已知兼容性问题</span>，需要在镜像中将以下代码修改，或者通过git补丁的形式合入以规避此问题。目前vllm-ascend仓库中已修复此问题，预计下个release版本正式解决。

```diff
--- a/vllm_ascend/distributed/kv_transfer/kv_pool/ascend_store/pool_worker.py
+++ b/vllm_ascend/distributed/kv_transfer/kv_pool/ascend_store/pool_worker.py
@@ -737,8 +737,6 @@ class KVPoolWorker:
         )
         # Initialize store, register buffers, and start transfer threads
         # directly here (like main) — no separate init_backend handshake.
-        if hasattr(self.m_store, "init_store"):
-            self.m_store.init_store()
         self.m_store.register_buffer(ptrs, lengths)
         self._start_kv_transfer_threads()
```



#### 2. 安装MF

vllm-ascend官方镜像中已安装memcache。如需手动安装参考如下：

```shell
pip install memfabric_hybrid==1.1.4
pip install memcache_hybrid==1.1.4
```

如果因网络原因无法在线安装，可以在 [PyPI · The Python Package Index](https://pypi.org/) 下载安装包离线安装。

下载链接：

[memcache-hybrid · PyPI](https://pypi.org/project/memcache-hybrid/1.1.4/#files)

[memfabric-hybrid · PyPI](https://pypi.org/project/memfabric-hybrid/1.1.4/#files)

安装其他非正式发布版可以直接源码编译安装，参考 [官方编译文档](https://gitcode.com/Ascend/memcache/blob/master/doc/install_run.md) 即可。



#### 3. 修改配置文件

MemCache由MetaService和LocalService两个组件构成，其中LocalService负责内存申请、数据读写和传输，MetaService作为独立进程启动，通过TCP与LocalService建立连接，负责元数据管理、数据淘汰、主备切换（高可用等功能）。

启动时MetaService、LocalService的配置由各自配置文件导入，默认配置文件在memcache的whl包config目录下 （通过 pip show memcache_hybrid 查看具体的路径）

```
/usr/local/lib/python3.11/site-packages/memcache_hybrid/config
```

**本案采用了每个节点独立作为一个内存池的方案，每个节点都需要拉起metaservice。**



###### 准备 MetaService 配置文件

在所有节点准备mmc-meta.conf

```ini
## IP 需要修改为启动节点IP
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## IP 需要修改为启动节点IP
ock.mmc.meta_service.config_store_url = tcp://127.0.0.1:6000

## HTTP service url
ock.mmc.meta_service.metrics_url = http://127.0.0.1:8000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_config.md)
```



##### LocalService 配置文件

在每个节点上准备mmc-local.conf配置文件，根据实际情况调整内存容量

```ini
## 配置为MetaService启动节点IP
ock.mmc.meta_service_url = tcp://127.0.0.1:5000

## 配置为MetaService启动节点IP
ock.mmc.local_service.config_store_url = tcp://127.0.0.1:6000

## Log level: debug, info, warn, error
ock.mmc.log_level = info

## The maximum supported rank count; once ranks are connected, no further modifications are allowed — a meta restart is required
ock.mmc.local_service.world_size = 256

 ## 卡侧ROCE网络传输为A2典型配置
ock.mmc.local_service.protocol = host_shm

## 每卡配置24GB内存，可根据环境实际共享内存容量调整（df -h查询）
ock.mmc.local_service.dram.size = 24GB

## For more configuration options and advanced parameters, please refer to the official documentation: [MemCache Configuration Guide](https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_config.md)
```



### 详细数据

基线（不启用池化）

| 输入 | 输出 | 并发 | 总请求数 | 前缀数 | 前缀重复率 | TTFT(ms) | TPOT(ms) |
| ---- | ---- | ---- | -------- | ------ | ---------- | -------- | -------- |
| 64K  | 1024 | 80   | 320      | 80     | 30%        | 16835.4  | 25.4     |
| 64K  | 1024 | 80   | 320      | 80     | 50%        | 15627.7  | 25.8     |
| 64K  | 1024 | 100  | 400      | 100    | 30%        | 24489.6  | 25.1     |
| 64K  | 1024 | 100  | 400      | 100    | 50%        | 24474.3  | 25.8     |
|      |      |      |          |        |            |          |          |
| 128K | 1024 | 80   | 320      | 80     | 30%        | 57755.3  | 24.0     |
| 128K | 1024 | 80   | 320      | 80     | 50%        | 56368.8  | 23.9     |
| 128K | 1024 | 100  | 400      | 100    | 30%        | 75295.0  | 23.9     |
| 128K | 1024 | 100  | 400      | 100    | 50%        | 75377.0  | 23.8     |



启用memcache

| 输入 | 输出 | 并发 | 总请求数 | 前缀数 | 前缀重复率 | TTFT(ms) | TPOT(ms) |
| ---- | ---- | ---- | -------- | ------ | ---------- | -------- | -------- |
| 64K  | 1024 | 80   | 320      | 80     | 30%        | 15672.2  | 25.5     |
| 64K  | 1024 | 80   | 320      | 80     | 50%        | 9988.6   | 26.6     |
| 64K  | 1024 | 80   | 320      | 80     | 70%        | 9625.7   | 26.3     |
| 64K  | 1024 | 80   | 320      | 80     | 90%        | 6499.8   | 26.8     |
| 64K  | 1024 | 100  | 400      | 100    | 30%        | 23302.1  | 25.8     |
| 64K  | 1024 | 100  | 400      | 100    | 50%        | 11167.7  | 27.2     |
| 64K  | 1024 | 100  | 400      | 100    | 70%        | 11055.9  | 27.5     |
| 64K  | 1024 | 100  | 400      | 100    | 90%        | 7008.4   | 27.5     |
| 64K  | 1024 | 120  | 480      | 120    | 30%        | 30834.6  | 25.6     |
| 64K  | 1024 | 120  | 480      | 120    | 50%        | 16037.9  | 27.3     |
| 64K  | 1024 | 120  | 480      | 120    | 70%        | 15837.2  | 27.3     |
| 64K  | 1024 | 120  | 480      | 120    | 90%        | 7739.3   | 27.8     |
| 64K  | 1024 | 140  | 560      | 140    | 30%        | 39402.4  | 25.5     |
| 64K  | 1024 | 140  | 560      | 140    | 50%        | 21184.3  | 27.5     |
| 64K  | 1024 | 140  | 560      | 140    | 70%        | 27952.3  | 26.9     |
| 64K  | 1024 | 140  | 560      | 140    | 90%        | 8419.5   | 28.2     |
|      |      |      |          |        |            |          |          |
| 128K | 1024 | 80   | 320      | 80     | 30%        | 63597.40 | 23.70    |
| 128K | 1024 | 80   | 320      | 80     | 50%        | 39609.40 | 24.90    |
| 128K | 1024 | 80   | 320      | 80     | 70%        | 26776.40 | 25.90    |
| 128K | 1024 | 80   | 320      | 80     | 90%        | 9041.0   | 27.20    |
| 128K | 1024 | 100  | 400      | 100    | 30%        | 84653.60 | 23.70    |
| 128K | 1024 | 100  | 400      | 100    | 50%        | 52333.30 | 25.20    |
| 128K | 1024 | 100  | 400      | 100    | 70%        | 37833.30 | 26.10    |
| 128K | 1024 | 100  | 400      | 100    | 90%        | 9955.10  | 28.10    |
| 128K | 1024 | 120  | 480      | 120    | 30%        | 111618.6 | 23.6     |
| 128K | 1024 | 120  | 480      | 120    | 50%        | 68280.6  | 25.0     |
| 128K | 1024 | 120  | 480      | 120    | 70%        | 49319.1  | 26.1     |
| 128K | 1024 | 120  | 480      | 120    | 90%        | 19934.5  | 28.1     |
| 128K | 1024 | 140  | 560      | 140    | 30%        | 131692.9 | 23.7     |
| 128K | 1024 | 140  | 560      | 140    | 50%        | 84138.8  | 24.8     |
| 128K | 1024 | 140  | 560      | 140    | 70%        | 60551.2  | 26.0     |
| 128K | 1024 | 140  | 560      | 140    | 90%        | 24689.0  | 28.1     |

