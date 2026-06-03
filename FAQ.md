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

## 3. 不同节点池化Size不一致，多个local进程启动失败， Mmap失败，日志中报错 AddVaInfo failed: address overlap
**问题现象**

初始化时，日志中有如下打印：
```
AddVaInfo failed: address overlap
```
**问题根因**

1、每个local进程的配置参数 `ock.mmc.local_service.max.dram.size` 和 `ock.mmc.local_service.max.hbm.size` 不一致，比如A进程的  `ock.mmc.local_service.max.dram.size`  设置 1GB，B进程的 `ock.mmc.local_service.max.dram.size`设置为1GB，则不一致。

2、`ock.mmc.local_service.max.dram.size` 和 `ock.mmc.local_service.max.hbm.size` 没有设置或者设置为0；而不同local进程的 `ock.mmc.local_service.dram.size` 和 `ock.mmc.local_service.hbm.size` 配置不一样，比如A进程`ock.mmc.local_service.hbm.size`设置为1GB，B进程`ock.mmc.local_service.hbm.size`设置为2GB或者为设置，则不一致。

**解决方案**

1、手动设置 `ock.mmc.local_service.max.dram.size`和 `ock.mmc.local_service.max.hbm.size`，让每个local进程一致。

2、如果不设置 `ock.mmc.local_service.max.dram.size` 或者 `ock.mmc.local_service.max.hbm.size`，则每个local进程对应的 `ock.mmc.local_service.dram.size` 和 `ock.mmc.local_service.hbm.size`要一致。

## 4.HA 模式下观察到master节点一直在切换，观察master日志发现master在master和backup间频繁切换

**问题现象**

初始化时，日志中有如下打印：
```
2026-06-02 08:57:07.344376 20 warning [MMC_leader_election.cpp:26 Logger] Updated label of self.pod_name=grtest-mmc-1-mmc-meta-0 to labels={'role': 'master'}
2026-06-02 08:57:07.344405 20 debug [MMC_leader_election.cpp:105 ElectionLoop] Election loop circle, pod-grtest-mmc-1-mmc-meta-0, times=33
2026-06-02 08:57:08.832026 24 warning [MMC_leader_election.cpp:189 RenewLease] Renew loop acquired Python GIL, pod-grtest-mmc-1-mmc-meta-0
2026-06-02 08:57:08.848462 24 debug [MMC_leader_election.cpp:206 RenewLease] Succeed in updating lease-grtest-mmc-1-mmc-meta-Lease: curHolder=grtest-mmc-1-mmc-meta-0, leaseDuration=30, renewTime=2026-06-02 08:57:38.848462, current=2026-06-02 08:57:38.848462, retry period=30
2026-06-02 08:57:10.344572 20 debug [MMC_leader_election.cpp:105 ElectionLoop] Election loop circle, pod-grtest-mmc-1-mmc-meta-0, times=64
2026-06-02 08:57:10.344681 20 debug [MMC_leader_election.cpp:107 ElectionLoop] Election loop acquired Python GIL, pod-grtest-mmc-1-mmc-meta-0
2026-06-02 08:57:10.344710 20 debug [MMC_leader_election.cpp:147 CheckLeaderStatus] Pod grtest-mmc-1-mmc-meta-0 labels={'role': 'master'}
2026-06-02 08:57:10.360390 20 warning [MMC_leader_election.cpp:26 Logger] Updated label of self.pod_name=grtest-mmc-1-mmc-meta-0 to labels={'role': 'backup'}
```
**问题根因**
造成问题的原因有一般有一下几种：

1. 🔥 续约失败 → 降级 → 检查到自己是 holder 又夺回 → 循环
- RenewLoop 每 3s 续约，一旦网络抖动/APIServer 慢/409 冲突导致续约失败，立即 isLeader_=false，label 变 backup
- 同一进程的 ElectionLoop 同样每 3s 检查 lease，发现 holder 还是自己，又立即夺回，label 变 master
- 3s 内 label 就来回跳一次

**解决方案**

检查集群网络配置，或手动修改leaseDurationSeconds 从 10 改大到 20-30。  

2. 🔥 两节点时钟不同步

- check_leader_status 用本地时间判断租约是否过期
- A 比 B 快 2s → B 认为 lease 已过期 → 抢主
- A 认为 lease 还有效 → 双方冲突
- 部署文档要求手动 set-time，生产环境基本都会偏差

**解决方案**

手动同步POD的时钟

3. 🔥 外部控制器自动清理k8s租约信息

**解决方案**

需用户自行排查解决
