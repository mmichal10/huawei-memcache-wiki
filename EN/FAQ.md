# Frequently Asked Questions and Troubleshooting
## 1. Store Object Initialization Fails
**Symptoms**

When the store initialization API is called, HCOM fails to obtain a GID and logs the following errors:

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

**Cause**

The `mmc-local.conf` file sets `ock.mmc.local_service.protocol` to `host_rdma`, but the IP address in `ock.mmc.local_service.hcom_url` is incorrect or missing.

**Resolution**

Run `show_gids` in the environment to find the RDMA interface IP address, then set it in `ock.mmc.local_service.hcom_url`.

## 2. Data Inconsistency During Cross-Node HOST_RDMA Transfers
**Symptoms**
The data returned by `get` does not match the data written by `put`.
- A single-node test with two clients succeeds.
- A cross-node test with two clients fails.

**Troubleshooting**
Check the MTU settings of the RDMA interfaces on both nodes:
```
ip link show
14: enp67s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 4500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether e8:eb:d3:3a:53:cc brd ff:ff:ff:ff:ff:ff
```
The latest HCOM version requires an RDMA interface MTU greater than 4096 for HOST_RDMA transfers. If necessary, change it with:
```
ip link set enp67s0f0np0 mtu 4500
```

## 3. Local Processes Fail to Start Because Pooled Sizes Differ Across Nodes
Multiple local processes fail to start, `mmap` fails, and the log reports `AddVaInfo failed: address overlap`.
**Symptoms**

The following messages appear during initialization:
```
AddVaInfo failed: address overlap
```
**Cause**

1. The `ock.mmc.local_service.max.dram.size` and `ock.mmc.local_service.max.hbm.size` values differ between local processes. For example, one process may set `ock.mmc.local_service.max.dram.size` to 1 GB while another sets it to a different value.

2. `ock.mmc.local_service.max.dram.size` or `ock.mmc.local_service.max.hbm.size` is unset or set to 0, while the corresponding `ock.mmc.local_service.dram.size` or `ock.mmc.local_service.hbm.size` values differ between local processes. For example, one process sets `ock.mmc.local_service.hbm.size` to 1 GB and another sets it to 2 GB or leaves it unset.

**Resolution**

1. Set `ock.mmc.local_service.max.dram.size` and `ock.mmc.local_service.max.hbm.size` explicitly and use the same values for every local process.

2. If you do not set `ock.mmc.local_service.max.dram.size` or `ock.mmc.local_service.max.hbm.size`, use the same `ock.mmc.local_service.dram.size` and `ock.mmc.local_service.hbm.size` values for every local process.

## 4. The Master Keeps Switching Between Master and Backup in HA Mode

**Symptoms**

The following messages appear during initialization:
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
**Cause**
Common causes include:

**1. 🔥 Renewal fails → leadership is relinquished → the process sees itself as the holder and takes leadership back → repeat**
- RenewLoop renews every 3 seconds. Network instability, a slow API server, or a 409 conflict can cause renewal to fail; `isLeader_` immediately becomes `false` and the label changes to `backup`.
- ElectionLoop in the same process also checks the lease every 3 seconds. If it still sees itself as the holder, it immediately takes leadership back and changes the label to `master`.
- The label can flip back and forth within 3 seconds.

**Resolution**

Check the cluster network configuration, or increase `leaseDurationSeconds` from 10 to 20–30.

**2. 🔥 The clocks on two nodes are out of sync**

- `check_leader_status` uses local time to determine whether the lease has expired.
- If A's clock is 2 seconds ahead of B's, B may consider the lease expired and attempt to become leader.
- A still considers the lease valid, so the nodes conflict.
- Deployment instructions require setting the time manually, and clocks can drift in production.

**Resolution**

Synchronize the clocks on the pods.

**3. 🔥 An external controller automatically deletes Kubernetes lease data**

Most cases are caused by scenarios 1 or 2. Otherwise, user code that periodically deletes Kubernetes lease data is a likely cause.

**Resolution**

Identify and resolve the issue in the user's external controller.

## 5. Mixing MemCache and MemFabric Across Python Environments Causes Startup or Data Access Failures
**Symptoms**

The inference service fails to start or access data and reports an error such as:

```
unpack response body failed, result: -1
```
The shared-library version printed in the logs may also differ from the installed MemCache and MemFabric versions:
```
hybm init successfully,  library version: xxx"
```

**Cause**

The inference framework's Python environment does not match the MemCache and MemFabric paths specified by `LD_LIBRARY_PATH`, causing startup or data access to fail.

**Resolution**

Check that the installed MemCache and MemFabric versions match the paths in `LD_LIBRARY_PATH`. Also check the inference framework and MemCache MetaService startup scripts. If the versions differ, configure them to use the same versions.

## 6. Inference Reports Errors Such as `lease expired`
**Symptoms**

The inference service fails to retrieve KV cache data and reports errors such as:

```
batch get lease expired.
batch copy read lease expired
lease expired after read.
```

**Cause**

When reading data, the client first requests metadata from MetaService. MetaService records the current time and a lease duration (10 seconds by default). If the subsequent data read takes longer than the lease, the lease expires. A large batch copy can trigger this issue.

**Resolution**

Increase the MetaService setting `ock.mmc.meta.lease_ttl_ms`; for example, set it to `50000`.

## 7. Memory Pool Initialization Fails with `AutoRanking failed`
**Symptoms**

The inference service reports the following error during startup:

```
2026-07-08 16:20:08.129923 ERROR [1757-1757][SMEM smem_bm_entry_manager.cpp:152 AutoRanking] AutoRanking failed, deviceId: 0, ret: -1, dataSize: 0
2026-07-08 16:20:08.129934 ERROR [1757-1757][SMEM smem_bm_entry_manager.cpp:78 Initialize] [RECOVER] Initialize attempt 5/5 failed, ret=-1 deviceId=0
2026-07-08 16:20:08.129939 ERROR [1757-1757][SMEM smem_bm_entry_manager.cpp:86 Initialize] initialize failed: -1
2026-07-08 16:20:08.130005 ERROR [1757-1757][SMEM smem_bm.cpp:86 smem_bm_init] init bm entry manager failed, result: -1
2026-07-08 16:20:08.130018 ERROR [1757-1757][MMC mmc_bm_proxy.cpp:55

```

**Cause**

The cluster's total rank count exceeds `ock.mmc.local_service.world_size` in `mmc_local.conf` (default 256). Calculate the rank count as the total number of devices in use plus the number of separated deployment processes.

**Resolution**

Increase `ock.mmc.local_service.world_size` in `mmc_local.conf` so it is at least the total rank count.
