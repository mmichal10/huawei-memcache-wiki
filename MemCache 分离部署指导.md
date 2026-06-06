### 概述

A3 环境中，由于[地址交集](https://gitcode.com/Ascend/memfabric_hybrid/wiki/FAQ.md#2-%E8%B0%83%E7%94%A8halmemcreate%E8%BF%94%E5%9B%9E%E9%94%99%E8%AF%AF%E7%A0%81-6)的限制，2T内存环境使用device_sdma只能使用约680GB内存做池化

在``融合部署模式``（MemCache和vllm在同一个进程）下，使用vLLM等推理框架拉起memcache时，由于推理框架首先加载权重，然后才初始化kvconnector，导致memcache无法申请到足够内存空间，[典型报错](https://gitcode.com/Ascend/memfabric_hybrid/wiki/FAQ.md#2-%E8%B0%83%E7%94%A8halmemcreate%E8%BF%94%E5%9B%9E%E9%94%99%E8%AF%AF%E7%A0%81-6)。

此时可以采用``分离部署模式``（MemCache和vllm在同一个进程）:
第一步：在拉起推理进程前，首先在每个节点启动memcache独立进程，即由独立的进程贡献内存
第二步：拉起推理进程，此时推理进程中的memcache不贡献内存。

![分离部署.png](https://raw.gitcode.com/user-images/assets/7672915/5b2ef6f2-700f-4c5b-a60f-5879d8c07eaf/分离部署.png '分离部署.png')

### 操作方式

本文假定读者已经掌握了常规方式拉起推理框架+memcache的能力。

##### 1. 准备配置文件

首先修改推理进程使用的mmc-local.conf，将ock.mmc.local_service.dram.size修改为0，然后增加ock.mmc.local_service.max.dram.size = 1024GB。

```ini
# 修改配置
ock.mmc.local_service.dram.size = 0GB

# 增加配置
ock.mmc.local_service.max.dram.size = 1024GB
```

然后复制一份推理进程的配置文件给memcache独立进程使用，命名为"mmc-local-standalone.conf"，并将ock.mmc.local_service.dram.size修改为独立进程内存申请量。

```ini
# 修改配置
ock.mmc.local_service.dram.size = 600GB

# 增加配置
ock.mmc.local_service.max.dram.size = 1024GB
```



##### 2. 准备脚本

在每个节点上准备如下脚本，命名为memcache_standalone.py

```python
import time

from memcache_hybrid import DistributedObjectStore


if __name__ == "__main__":
    store = DistributedObjectStore()
    res = store.init(0)
    if res != 0:
        print(f"Failed to initialize memcache, res = {res}")
        exit(1)
    print("Successfully initialized memcache. ")

    while True:
        time.sleep(1)
```


##### 3. 拉起

1. 首先按原有方案拉起metaservice。

   ```shell
   export MMC_META_CONFIG_PATH=...
   python3 -c 'from memcache_hybrid import MetaService; MetaService.main()'
   ```

2. 然后拉起memcache独立进程。

   ```shell
   export MMC_LOCAL_CONFIG_PATH="mmc-local-standalone.conf"
   python3 memcache_standalone.py
   ```

   **等待所有节点都启动成功（看到打印"Successfully initizlized memcache. "）后再进行下一步**

3. 最后按原有方法拉起推理进程。