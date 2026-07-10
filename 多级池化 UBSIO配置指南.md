# UBSIO配置参考

## A2 配置示例

```ini
# UBSIO 配置
[bio]

# A2 必须配置本地 SSD 缓存盘或分区路径；多个路径使用英文冒号分隔。
bio.disk.path = /dev/nvmexn1:/dev/nvmexn2:/dev/nvmexn3:/dev/nvmexn4:/dev/nvmexn5:/dev/nvmexn6:/dev/nvmexn7:/dev/nvmexn8

# UBSIO 内存池大小，单位 GB。A2 默认配置 10GB。
bio.mem.size_in_gb = 10

# A2 推荐内存淘汰水位配置为 0，使数据尽快下沉到 SSD。
bio.wcache.evict_water_level = 0

# 写缓存磁盘淘汰水位，推荐配置为 90。
bio.wcache.disk_evict_water_level = 90

# A2 需要与推理进程使用的 NPU 卡数保持一致；此处假设推理进程启用 8 张卡。
bio.standalone.device_count = 8

# 设置日志输出级别。可选 `error`、`warn`、`info`、`debug`、`trace`。
bio.log.level = info

# 以下配置无需修改。
bio.cache.mem_read_write_ratio = 0:10
bio.cache.disk_read_write_ratio = 0:10
bio.underfs.file_system_type = none
bio.net.tls.enable.switch = false
bio.sdkmem.size_in_mb = 0
bio.cache.qos.enable = false
```

## A3 配置示例

```ini
# UBSIO 配置
[bio]

# A3 可配置一块或多块独占 SSD，多个路径使用英文冒号分隔。
bio.disk.path = /dev/nvmexnx:/dev/nvmexnx

# UBSIO 内存池大小，单位 GB。A3 默认配置 800GB，推荐按 600-800GB 配置。
bio.mem.size_in_gb = 800

# A3 推荐内存淘汰水位配置为 85。
bio.wcache.evict_water_level = 85

# 写缓存磁盘淘汰水位，推荐配置为 90。
bio.wcache.disk_evict_water_level = 90

# A3 只有一个 UBSIO 服务时配置为 1。
bio.standalone.device_count = 1

# 设置日志输出级别。可选 `error`、`warn`、`info`、`debug`、`trace`。
bio.log.level = info

# 以下配置无需修改。
bio.cache.mem_read_write_ratio = 0:10
bio.cache.disk_read_write_ratio = 0:10
bio.underfs.file_system_type = none
bio.net.tls.enable.switch = false
bio.sdkmem.size_in_mb = 0
bio.cache.qos.enable = false
```

## 配置表

| 配置项                                                                     | 值类型 | 是否必填 | 默认值                         | 有效范围                                              | 说明                                                                                            |
| ----------------------------------------------------------------------- | --- | ---- | --------------------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| [`bio.disk.path`](#biodiskpath)                                         | 字符串 | 可选   | `空` | 使用冒号分隔的已存在路径                                      | 缓存盘或分区列表，不填表示不使用磁盘设备仅使用内存缓存。A2 场景下路径数量建议不小于当前推理使用的 NPU 卡数；A3 场景下只有一个 UBSIO 服务时可直接配置独占磁盘、分区或 loop 设备 |
| `bio.log.level`                                                         | 字符串 | 可选   | `info`                      | `error`、`warn`、`info`、`debug`、`trace`             | 配置初始化后设置 BoostIO server 日志级别。                                                                 |
| [`bio.standalone.device_count`](#biostandalonedevice_count)             | 整数  | 可选   | `0`                         | `0` 到 `16`                                        | 当前推理服务对应的 UBSIO 服务数量。`0` 表示保留旧逻辑，即每个 `deviceId` 按路径下标选择一个设备。                                  |
| `bio.segment.size_in_mb`                       | 整数  | 可选   | `4`                         | `1` 到 `16`                                        | 缓存 segment 大小。单机内存池 block、BDM chunk 和 SDK data-message block 都使用该大小。                          |
| [`bio.mem.size_in_gb`](#biomemsize_in_gb)                             | 整数  | 可选   | `50`                        | `0` 到 `512`；不得超过当前系统空闲内存                          | 单机 server 内存池容量。                                                                              |
| `bio.sdkmem.size_in_mb`                                                 | 整数  | 可选   | `5120`                      | `0` 到 `4194304`                                   | SDK data-message 内存池大小，主要用于 BatchGet 数据缓冲。                                                    |
| `bio.net.tls.enable.switch`                                             | 布尔值 | 可选   | `true`                      | `true` 或 `false`                                  | 本地单机运行且没有 TLS 文件时，将该项设为 `false`。                                                              |
| `bio.trace.enable`                                                      | 布尔值 | 可选   | `true`                      | `true` 或 `false`                                  | 启用或禁用 HTrace 采集。单机模式会初始化 tracer 模块。                                                           |
| `bio.data.crc.enable`                                                   | 布尔值 | 可选   | `false`                     | `true` 或 `false`                                  | 启用缓存数据 CRC 校验。单机 direct-call 路径会把该设置传给 SDK 和 cache 模块。                                        |
| `bio.cache.qos.enable`                                                  | 布尔值 | 可选   | `true`                      | `true` 或 `false`                                  | 启用 cache QoS 和过载控制行为。                                                                         |
| [`bio.wcache.evict_water_level`](#biowcacheevict_water_level)           | 整数  | 可选   | `0`                         | `0` 到 `100`                                       | 写缓存内存淘汰水位，占配置的写缓存内存容量的百分比。                                                                    |
| [`bio.wcache.disk_evict_water_level`](#biowcachedisk_evict_water_level) | 整数  | 可选   | `100`                       | `0` 到 `100`                                       | 写缓存磁盘淘汰水位，占配置的写缓存磁盘容量的百分比。                                                                    |
| `bio.rcache.evict_water_level`                                          | 整数  | 可选   | `90`                        | `0` 到 `100`                                       | 读缓存淘汰水位。内存和磁盘读缓存淘汰阈值使用同一个值。                                                                   |
| `bio.cache.mem_read_write_ratio`                                        | 字符串 | 可选   | `5:5`                       | 两个 `0` 到 `10` 的整数；总和必须为 `10`                      | 将 `bio.mem.size_in_gb` 划分给读缓存和写缓存。例如：`0:10` 表示把全部内存缓存容量预留给写缓存。                                |
| `bio.cache.disk_read_write_ratio`                                       | 字符串 | 可选   | `5:5`                       | 两个 `0` 到 `10` 的整数；总和必须为 `10`                      | 将缓存盘容量划分给读缓存和写缓存。例如：`0:10` 表示把全部磁盘缓存容量预留给写缓存。                                                 |
| `bio.work.io.timeout`                                                   | 整数  | 可选   | `60`                        | `60` 到 `300`                                      | 通过单机 runtime config 传给 SDK 的 I/O 超时时间，单位为秒。                                                   |
| `bio.batchget.thread.num`                                               | 整数  | 可选   | `32`                        | `8` 到 `512`                                       | MirrorServer BatchGet executor 线程数。                                                           |
| `bio_cli_tools_enable`                                                  | 布尔值 | 可选   | `false`                     | `true` 或 `false`                                  | 启用 CLI agent 诊断能力。使用 `cli_client --attach` 连接 `bio_console` 时应设为 `true`。                      |
| `bio.underfs.file_system_type`                                          | 字符串 | 可选   | `ceph`                      | `ceph`、`hdfs` 或 `none`                            | UnderFS 后端类型。单机模式使用 `none`。需要 load 或 write-through 后端访问时使用 `ceph` 或 `hdfs`。                   |
| `bio.underfs.ceph.cfg.path`                                             | 字符串 | 可选   | `/etc/ceph/ceph.conf`       | 当 `bio.underfs.file_system_type = ceph` 时必须是已存在路径 | Ceph 配置文件路径。当 UnderFS 类型为 `ceph` 且该路径无效时，配置初始化失败。                                             |
| `bio.underfs.ceph.cluster`                                              | 字符串 | 可选   | `ceph`                      | 非空字符串                                             | 后端为 Ceph 时，UnderFS 插件使用的 Ceph 集群名。                                                            |
| `bio.underfs.ceph.user`                                                 | 字符串 | 可选   | `client.admin`              | 非空字符串                                             | 后端为 Ceph 时，UnderFS 插件使用的 Ceph 用户。                                                             |
| `bio.underfs.ceph.pool`                                                 | 字符串 | 可选   | `0:jfspool1,1:jfspool2`     | 使用逗号分隔的 `poolId:poolName` 条目，poolId 为非负整数         | Ceph pool 映射。当前 UnderFS helper 使用 pool id `0` 初始化插件。                                          |

# 详细配置项说明

## bio.disk.path

### 配置含义与示例

`bio.disk.path` 用于指定三级池化缓存层使用的块设备或分区，决定是否启用本地 SSD 缓存层：留空表示无盘模式，仅使用 `bio.mem.size_in_gb` 配置的本地内存池；配置为块设备或分区路径时，UBSIO 会将这些路径作为 SSD 缓存空间使用。多个路径使用英文冒号（`:`）分隔，最多 **64** 条。

#### A2

A2 场景下，`bio.disk.path` 需要与当前推理使用的 NPU 卡数或 deviceId 分配关系匹配，路径数量建议不小于当前推理使用的 NPU 卡数；单实例多盘时可配置为“卡数 × 盘数”，让每个 deviceId 获得多条分区路径。详细分区操作见 [A2 分区操作](#a2-分区操作)。

```ini
# 4 卡推理实例，1 块盘切 4 个分区
bio.disk.path = /dev/nvmexn1p1:/dev/nvmexn1p2:/dev/nvmexn1p3:/dev/nvmexn1p4

# 4 卡推理实例，2 块盘共 8 个分区，配合 device_count 做路径分配
bio.disk.path = /dev/nvmexn1p1:/dev/nvmexn1p2:/dev/nvmexn1p3:/dev/nvmexn1p4:/dev/nvmeyn1p1:/dev/nvmeyn1p2:/dev/nvmeyn1p3:/dev/nvmeyn1p4
bio.standalone.device_count = 4
```

#### A3

A3 场景下，UBSIO 独立于推理进程，随 memcache 独立进程启动，通常只有一个 UBSIO 服务，不需要按 NPU 卡数切分 `bio.disk.path`。如果磁盘由 UBSIO 独占使用，可以直接配置整盘路径，不强制分区；也可以配置已有分区路径。无可用物理盘时，也可以使用 loop 设备作为 `bio.disk.path`。

```ini
# 1 个 UBSIO 服务，直接使用独占磁盘，不分区
bio.disk.path = /dev/nvmexn1
bio.standalone.device_count = 1

# 多块独占磁盘时使用冒号分隔
bio.disk.path = /dev/nvmexn1:/dev/nvmexn1
bio.standalone.device_count = 1

# 无可用物理盘时，也可以使用 loop 设备
bio.disk.path = /dev/loopx
bio.standalone.device_count = 1 
```

无盘模式表示不接入本地 SSD 作为 L3 缓存扩容层，UBSIO 仅使用 `bio.mem.size_in_gb` 配置的本地内存池承载缓存读写，重启后缓存数据不持久化；配置示例如下。

```ini
# 无盘模式：不启用 SSD 缓存层
bio.disk.path =
```

---

## bio.standalone.device_count

`bio.standalone.device_count` 表示当前单机推理服务中参与本地缓存路径分配的 device 数量。它主要用于有盘模式下把 `bio.disk.path` 中的多条路径分配给不同 deviceId。

- `0`：兼容旧逻辑。每个 deviceId 按 `bio.disk.path` 的路径下标选择设备，例如 `deviceId = 1` 选择第 2 条路径。
- 大于 `0`：按配置的 device 数量对 `bio.disk.path` 路径做均衡分配。默认情况下按当前推理使用的 NPU 卡数填写；A3 分离部署且只有一个 UBSIO 服务时可填 `1`。

示例：4 卡推理服务使用 2 块盘切出的 8 个分区时，填写 `4`，让 4 个 deviceId 分摊这些分区路径。

```ini
bio.disk.path = /dev/nvmexn1p1:/dev/nvmexn1p2:/dev/nvmexn1p3:/dev/nvme0n1p4:/dev/nvmeyn1p1:/dev/nvmeyn1p2:/dev/nvmeyn1p3:/dev/nvmeyn1p4
bio.standalone.device_count = 4
```

无盘模式下通常保持默认值即可：

```ini
bio.disk.path =
bio.standalone.device_count = 0
```

---

## bio.mem.size_in_gb

`bio.mem.size_in_gb` 表示 UBSIO 可使用的本地内存池容量，单位为 GB。该值需要结合 A2 / A3 场景和节点剩余内存规划，不能超过推理进程、Memcache 等业务进程占用后可留给 UBSIO 的内存空间。

### A2

A2 场景下，UBSIO 通常随推理进程按卡维度使用本地缓存资源，`bio.mem.size_in_gb` 建议默认配置为 `10`。

```ini
# 示例：A2 默认配置 10GB
bio.mem.size_in_gb = 10
bio.standalone.device_count = 8
```

### A3

A3 场景下，UBSIO 随 memcache 独立进程启动，通常只有一个 UBSIO 服务。`bio.mem.size_in_gb` 需要按整机维度规划，不超过推理进程和 Memcache 占用后剩余的内存总量。

```ini
# 示例：A3 默认配置 800GB，推荐按 600-800GB 配置
bio.mem.size_in_gb = 800
bio.standalone.device_count = 1
```

---

## bio.wcache.evict_water_level

`bio.wcache.evict_water_level` 是写缓存的内存淘汰水位，取值范围 `0` 到 `100`，表示写缓存内存容量使用到多少百分比后开始触发淘汰策略。

无盘模式下没有 SSD 承接写缓存淘汰，通常设置为较高水位，让缓存尽量留在内存中；当前最小单机模式示例使用 `85`：

```ini
bio.disk.path =
bio.wcache.evict_water_level = 85
```

有盘模式下，如果希望写缓存更快进入 SSD 层，可设置为较低水位，例如：

```ini
bio.disk.path = /dev/nvmexn1p1:/dev/nvmexn1p2:/dev/nvmexn1p3:/dev/nvmexn1p4
bio.wcache.evict_water_level = 0
```

配置建议：无盘场景优先使用较高水位；有盘场景可根据内存压力和 SSD 容量降低该值。

---

## bio.wcache.disk_evict_water_level

`bio.wcache.disk_evict_water_level` 是写缓存的磁盘淘汰水位，取值范围 `0` 到 `100`，表示写缓存磁盘容量使用到多少百分比后开始触发磁盘侧淘汰。

该配置只在 `bio.disk.path` 非空时有实际意义。常见有盘配置如下：

```ini
bio.disk.path = /dev/nvmexn1p1:/dev/nvmexn1p2:/dev/nvmexn1p3:/dev/nvmexn1p4
bio.wcache.disk_evict_water_level = 90
```

无盘模式下该项不会启用磁盘缓存层，可保留当前示例配置：

```ini
bio.disk.path =
bio.wcache.disk_evict_water_level = 90
```

配置建议：磁盘空间充足时可使用较高水位，提高 SSD 缓存利用率；希望更早释放磁盘缓存空间时可适当降低该值。

---

# 其他

## A2 分区操作

A2 启用 SSD 缓存层通常需要先把物理盘或 loop 设备切分为与 deviceId 匹配的分区，再把输出写入 `bio.disk.path`。A3 不需要执行本节分区操作；只有一个 UBSIO 服务时，可直接配置独占磁盘或 loop 设备路径。

### 场景一：有盘部署（推荐）

**背景：** 单节点多块 NVMe 盘，分区总数需提前规划，每个分区对应一个推理实例的缓存存储路径。

#### 1. 确认磁盘状态

通过以下命令确认哪些磁盘为可用裸盘（无分区、无挂载、无文件系统签名）。

```bash
# 1. 列出所有磁盘，确认哪些是裸盘（无分区即为裸盘）
lsblk

# 输出示例：
# NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
# nvme0n1     259:0    0   3.5T  0 disk              # <-- 裸盘，可用
# nvme1n1     259:1    0   3.5T  0 disk              # <-- 裸盘，可用
# nvme2n1     259:2    0   1.8T  0 disk
# +-nvme2n1p1 259:3    0   1.8T  0 part /data        # <-- 已分区挂载，不可用

# 2. 确认目标盘为裸盘（以 /dev/nvme0n1 为例，替换为实际磁盘）
lsblk /dev/nvme0n1                    # 应无分区
mount | grep nvme0n1                  # 应无挂载
sudo blkid /dev/nvme0n1               # 应无文件系统签名
```

#### 2. 使用 partition_disks.sh 分区

> <span style="color:red">**⚠️ 高危操作：选错磁盘将造成不可逆的数据丢失。请务必在执行前通过 `lsblk` 反复确认目标磁盘路径。**</span>

**[partition_disks.sh](https://gitcode.com/openeuler/ubs-io/blob/develop/scripts/ubsio-boostio/partition_disks.sh)脚本下载：**

```bash
wget -O /path/to/partition_disks.sh https://raw.gitcode.com/openeuler/ubs-io/blobs/b81de180c6a15059a28155f17c8c96f502ef70cf/partition_disks.sh
chmod +x /path/to/partition_disks.sh
```

脚本创建 GPT 分区表，均分磁盘并输出 `bio.disk.path`。默认 dry-run，加 `--yes` 才写入。

| 参数                    | 说明                                     |
| --------------------- | -------------------------------------- |
| `-n, --parts N`       | 总分区数，一般与卡数一致；单推理实例多盘时可为卡数 × 盘数；最大不超过64 |
| `-d, --disk PATH`     | 磁盘路径                                   |
| `-c, --dev-cnt N`     | 占用的卡数；单节点多推理实例场景可不填（默认 0）              |
| `-y, --yes`           | 实际写入分区表                                |
| `-f, --force`         | 覆盖已有分区表                                |
| `-w, --wipe-metadata` | 擦除旧文件系统签名和 BDM 头                       |

#### 2.1 单推理实例

`--parts` 一般与节点卡数一致。如果有多块盘，建议 `--parts` 设为卡数 × 盘数（`-c` 仍填卡数），以优化性能。

**示例：1 个推理实例 4 张卡，1 块盘**

```bash
bash /path/to/partition_disks.sh --parts 4 --disk /dev/nvme0n1 -f

sudo bash /path/to/partition_disks.sh \
    --parts 4 --disk /dev/nvme0n1 --yes -f -w

# 输出：
# bio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2:/dev/nvme0n1p3:/dev/nvme0n1p4
```

**`vim ubsio.conf`，增加或修改下述配置：**

```
bio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2:/dev/nvme0n1p3:/dev/nvme0n1p4
```

**示例：1 个推理实例 4 张卡，2 块盘（`--parts = 卡数 × 盘数 = 8`）**

```bash
sudo bash /path/to/partition_disks.sh \
    --parts 8 --disk /dev/nvme0n1 --disk /dev/nvme1n1 -c 4 --yes -f -w

# 输出：
# bio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2:/dev/nvme0n1p3:/dev/nvme0n1p4:/dev/nvme1n1p1:/dev/nvme1n1p2:/dev/nvme1n1p3:/dev/nvme1n1p4
```

**`vim ubsio.conf`，增加或修改下述配置：**

```
bio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2:/dev/nvme0n1p3:/dev/nvme0n1p4:/dev/nvme1n1p1:/dev/nvme1n1p2:/dev/nvme1n1p3:/dev/nvme1n1p4
bio.standalone.device_count = 4
```

#### 2.2 多推理实例

`--parts` 一般与节点卡数一致。多实例容器隔离，独立配置文件。

**示例：8 分区，2 个推理实例各占 4 张卡，2 块盘**

```bash
sudo bash /path/to/partition_disks.sh \
    --parts 8 \
    --disk /dev/nvme0n1 \
    --disk /dev/nvme1n1 \
    --yes -f -w

# 输出全部分区：
# bio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2:/dev/nvme0n1p3:/dev/nvme0n1p4:/dev/nvme1n1p1:/dev/nvme1n1p2:/dev/nvme1n1p3:/dev/nvme1n1p4
```

**`vim ubsio.conf`（实例 0），增加或修改下述配置：**

```
bio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2:/dev/nvme0n1p3:/dev/nvme0n1p4
```

**`vim ubsio.conf`（实例 1），增加或修改下述配置：**

```
bio.disk.path = /dev/nvme1n1p1:/dev/nvme1n1p2:/dev/nvme1n1p3:/dev/nvme1n1p4
```

---


### 场景二：模拟盘部署（loop 设备方案）

A2 无可用裸盘时，可用 `dd` + `losetup --find --show` 创建 loop 块设备模拟，由系统自动选择空闲 loop 设备，默认开启 direct_io 直通模式，并继续按本节步骤分区。A3 可直接使用 loop 设备路径，不需要执行分区命令。

> <span style="color:red">**⚠️ 高危操作：创建 loop 设备前，建议先确认当前 loop 设备占用情况；不要手动复用已被占用的 `/dev/loopN`，否则可能导致数据丢失。**</span>

创建 1TB 镜像，并自动分配空闲 loop 设备；分区步骤使用 [分区脚本说明](#分区脚本说明) 中的 `partition_disks.sh`：

```bash
# 创建 1TB 镜像文件（count=1024，可根据需求调整）
sudo dd if=/dev/zero of=/data/boostio_disk.img bs=1G count=1024 status=progress

# 挂载 loop 设备，默认开启 direct_io 直通；命令会输出实际分配的设备路径
LOOP_DEV=$(sudo losetup --find --show --direct-io=on /data/boostio_disk.img)
echo "${LOOP_DEV}"
# 示例输出：/dev/loop2

# 验证 direct_io 已开启
losetup -l "${LOOP_DEV}" | grep -i direct
# 期望输出包含：Direct I/O: on

# 分区
sudo bash /path/to/partition_disks.sh \
    --parts 4 --disk "${LOOP_DEV}" -c 4 --yes -f -w

# 输出：
# bio.disk.path = /dev/loop2p1:/dev/loop2p2:/dev/loop2p3:/dev/loop2p4
```

**`vim ubsio.conf`，将脚本输出的实际分区路径增加或修改到下述配置：**

```
bio.disk.path = /dev/loop2p1:/dev/loop2p2:/dev/loop2p3:/dev/loop2p4
```

---

### 取消 loop 设备

> <span style="color:red">**⚠️ 注意： 仅在确认不再需要 loop 模拟盘时执行，卸载后缓存数据将丢失。**</span>

```bash
# 1. 查看当前 loop 设备及其关联文件
losetup -a

# 2. 卸载 loop 设备（替换为上面 losetup --find --show 输出的设备）
sudo losetup -d /dev/loop2

# 3. 删除镜像文件（可选）
sudo rm -f /data/boostio_disk.img
```

---

### 注意事项

| 事项 | 说明 |
|------|------|
| 磁盘独占 | UBSIO / BoostIO 独占配置的磁盘/分区，不可与其他服务共享 |
| 物理盘隔离 | 同一物理盘的分区不会分配给同一 deviceId（算法自动打散） |
| 路径上限 | `bio.disk.path` 最多 64 条路径 |
| 动态加盘 | standalone 模式不支持运行时加盘 |
| 复用磁盘 | 不同 deviceId 会写入 BDM disk head，不匹配将拒绝恢复 |
| 权限 | 进程需对块设备有读写权限，建议 root 运行 |
