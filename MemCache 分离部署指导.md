### 概述

使用vLLM等推理框架拉起memcache时，某些时候由于推理框架首先加载权重后初始化kvconnector，导致memcache无法申请到足够内存空间（典型场景为A3使用device_sdma传输协议）。此时可以采用分离部署模式，在拉起推理进程前首先在每个节点启动memcache独立进程提前申请内存，在随后启动的推理进程中memcache使用不贡献内存的配置。

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

然后复制一份推理进程的配置文件给memcache独立进程使用，命名为"memcache.conf"，并将ock.mmc.local_service.dram.size修改为独立进程内存申请量。

```ini
# 修改配置
ock.mmc.local_service.dram.size = 400GB

# 增加配置
ock.mmc.local_service.max.dram.size = 1024GB
```



##### 2. 准备脚本

在每个节点上准备如下脚本，命名为memcache.py

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
   export MMC_LOCAL_CONFIG_PATH="memcache.conf"
   python3 memcache.py
   ```

   **等待所有节点都启动成功（看到打印"Successfully initizlized memcache. "）后再进行下一步**

3. 最后按原有方法拉起推理进程。