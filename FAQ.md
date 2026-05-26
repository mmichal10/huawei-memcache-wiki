# 常见问题及解决办法
## 1. store对象初始化失败
**问题现象**

调用store初始化接口时，HCOM获取gid失败，日志中报以下错误：

```
ERROR [HCOM rdma_device_helper.cpp:322] Failed to get proper gid by address for ip 127.0.0.1
ERROR [HCOM net_rdma_driver.cpp:188] Failed to get device by ip
ERROR [HCOM net_rdma_driver.cpp:60] RDMA failed to create ctx
INFO [HCOM service_periodic_manager.cpp:189] PeriodicManager for timeout [name: hybm_hcom_service, index: 0] working thread exit
[HCOM service_imp.cpp:151] Driver start failed, res:210
ERROR [952-952][HYBM host_hcom_transport_manager.cpp:115 OpenDevice] Failed to start hcom service, nic: tcp://127.0.0.1:8100 type: 0 ret: 210
ERROR [952-952][HYBM compose_transport_manager.cpp:59 OpenDevice] Failed to open device transport nic: tcp://127.0.0.1:8100
ERROR [952-952][HYBM host_hcom_transport_manager.cpp:135 CloseDevice] Assert rpcService_ != 0
ERROR [952-952][HYBM hybm_entity_default.cpp:1048 InitTransManager] Failed to open device, ret: -1
ERROR [952-952][HYBM hybm_entity_default.cpp:96 Initialize] Failed to InitTransManager
ERROR [952-952][HYBM hybm_big_mem_entry.cpp:34 hybm_create_entity] initialize entity failed: -1
ERROR [952-952][SMEM smem_bm_entry.cpp:54 Initialize] create entity failed
ERROR [952-952][SMEM smem_bm.cpp:202 smem_bm_create2] entry init failed, result: -1
ERROR [952-952][MMC mmc_bm_proxy.cpp:103 InternalCreateBm] Failed to create smem bm
ERROR [952-952][MMC mmc_bm_proxy.cpp:60 InitBm] Internal create bm failed
INFO [952-952][HYBM hybm_entry.cpp:134 hybm_uninit] uninitialize GVA memory return: 0
INFO [952-952][SMEM smem_bm.cpp:105 smem_bm_uninit] smem_bm_uninit finished
ERROR [952-952][MMC mmc_local_service_default.cpp:38 Start] Failed to init bm of local service local_service, error code -1
ERROR [952-952][MMC mmc_service.cpp:53 mmcs_local_service_start] create or start local service failed
ERROR [952-952][MMC mmc.cpp:70 mmc_init] failed to create or start local service
```

**问题根因**

此问题是因为mmc-local.conf配置文件中将ock.mmc.local_service.protocol设置为host_rdma，但是没有正确设置ock.mmc.local_service.hcom_url中的ip地址

**解决方案**

需要在环境上执行show_gids查询RDMA网卡的IP地址，并将其设置到ock.mmc.local_service.hcom_url中

## 2. HOST_RDMA跨机传输后出现数据一致性问题
**问题现象**
get获取到的数据和put不相符
- 单机双client测试无问题
- 跨机双client测试有问题

**定位思路**
检查两节点rdma网卡的MTU设置：
```
ip link show
14: enp67s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 4500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether e8:eb:d3:3a:53:cc brd ff:ff:ff:ff:ff:ff
```
HCOM最新版本HOST_RDAM传输需要rdma网卡MTU大于4096，如果不正确可通过如下命令修改：
```
ip link set enp67s0f0np0 mtu 4500
```

## 3. 不同节点池化Size不一致，多个local进程启动失败，日志中报错 AddVaInfo failed: address overlap
**问题现象**

初始化时，日志中有如下打印：
```
AddVaInfo failed: address overlap
```
**问题根因**

1、每个local进程的配置参数 `max.dram.size` 和 `max.hbm.size` 不一致，比如A进程的  `max.dram.size`  设置 1GB，B进程的 `max.dram.size`设置为1GB，则不一致。

2、`max.dram.size` 和 `max.hbm.size` 没有设置或者设置为0；而不同local进程的 `dram.size` 和 `hbm.size` 配置不一样，比如A进程`hbm.size`设置为1GB，B进程`hbm.size`设置为2GB或者为设置，则不一致。

**解决方案**

1、手动设置 `max.dram.size`和 `max.hbm.size`，让每个local进程一致。

2、如果不设置 `max.dram.size` 或者 `max.hbm.size`，则每个local进程对应的 `dram.size` 和 `hbm.size`要一致。