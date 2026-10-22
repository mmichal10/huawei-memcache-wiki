### Overview

In an A3 environment, address-space overlap limits how much memory can be pooled. On a 2 TB system, `device_sdma` can pool only about 680 GB. See the [address overlap FAQ](https://gitcode.com/Ascend/memfabric_hybrid/wiki/FAQ.md#2-%E8%B0%83%E7%94%A8halmemcreate%E8%BF%94%E5%9B%9E%E9%94%99%E8%AF%AF%E7%A0%81-6).

In **fused deployment mode** (MemCache and vLLM in the same process), the inference framework loads model weights before initializing the KV connector and MemCache. MemCache may therefore be unable to allocate enough memory. See the [FAQ for a typical error](https://gitcode.com/Ascend/memfabric_hybrid/wiki/FAQ.md#2-%E8%B0%83%E7%94%A8halmemcreate%E8%BF%94%E5%9B%9E%E9%94%99%E8%AF%AF%E7%A0%81-6).

Use **separated deployment mode** (MemCache and vLLM in different processes) to allocate as much memory as possible:

Step 1: Before starting inference, start a standalone MemCache process on each node. These processes contribute memory to the pool.

Step 2: Start the inference process. MemCache in the inference process does not contribute memory.

The deployment is illustrated below:
![memcache_separated_deployment.png](https://raw.gitcode.com/user-images/assets/7672915/faac25cd-1e48-4d5d-9649-eb5fad1850b3/memcache_separated_deployment.png 'memcache_separated_deployment.png')

### Procedure

This guide assumes that you already know how to start the inference framework with MemCache using the standard deployment method.

##### 1. Scan the Available SDMA Memory

Ensure that the `buff/cache` value reported by `free -h` is as low as possible.

![image.png](https://raw.gitcode.com/user-images/assets/7672915/838d16fc-206d-4bad-9488-12d71cc93a49/image.png 'image.png')
		
You can clear the file cache with the following command. <span style="color:#e60000;">Warning:</span> this clears cached files, so model weights and other data must be reloaded from the filesystem. The next vLLM startup will take longer. Use this command only if you understand the effect.

```shell
echo 3 > /proc/sys/vm/drop_caches
```

Run [mem_scan.py](https://gitcode.com/Ascend/memfabric_hybrid/blob/master/src/smem/python/memfabric_hybrid/memfabric_hybrid/mem_scan.py) to scan available physical memory:

`python3 mem_scan.py`

![image.png](https://raw.gitcode.com/user-images/assets/7672916/348c0854-4efb-4803-8222-f6bf19589fa1/image.png 'image.png')

If the scan still reports too little available SDMA memory, trigger memory compaction with:

```shell
echo 1 /proc/sys/vm/compact_memory
```

If available memory is still insufficient, reboot the environment.

##### 2. Prepare the Configuration Files

Assume the scan found at least 600 GB of available memory.

First, edit the `mmc-local.conf` used by the inference process. Set `ock.mmc.local_service.dram.size` to 0, then add `ock.mmc.local_service.max.dram.size = 1024GB`.

```ini
# Update the configuration
ock.mmc.local_service.dram.size = 0GB

# Add the configuration
ock.mmc.local_service.max.dram.size = 1024GB
```

Next, copy the inference configuration for the standalone MemCache process. This example names the copy `mmc-local-standalone.conf`, but you can choose any name. Set `ock.mmc.local_service.dram.size` to the amount of memory the standalone process should allocate.

```ini
# Update the configuration
ock.mmc.local_service.dram.size = 600GB

# Add the configuration
ock.mmc.local_service.max.dram.size = 1024GB
```


##### 3. Prepare the Script

Create the following script on every node. This example names it `memcache_standalone.py`.

```python
import time

from memcache_hybrid import DistributedObjectStore


if __name__ == "__main__":
    store = DistributedObjectStore()
    res = store.init(0)
    if res != 0:
        print(f"Failed to initialize memcache, res = {res}", flush=True)
        exit(1)
    print("Successfully initialized memcache. ", flush=True)

    while True:
        time.sleep(1)
```


##### 4. Start the Services

1. Start MetaService using your existing procedure.

   ```shell
   export MMC_META_CONFIG_PATH=[mmc-meta.conf PATH]
   python3 -c 'from memcache_hybrid import MetaService; MetaService.main()'
   ```

2. Start the standalone MemCache process.

   ```shell
   export MMC_LOCAL_CONFIG_PATH="mmc-local-standalone.conf"
   python3 memcache_standalone.py
   ```

   **Wait until the process starts successfully on every node (the log prints `Successfully initialized memcache.`) before continuing.**

3. Finally, start the inference process using your existing procedure.
