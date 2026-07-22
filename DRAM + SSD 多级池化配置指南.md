# 配置指南

UBSIO 与 Memcache 共用 `mmc-local.conf`。

## A2 配置示例

```ini
# UBSIO 配置（写入 mmc-local.conf）

# A2 必须配置本地 SSD 缓存盘或分区路径；多个路径使用英文冒号分隔。
ubsio.disk.path = /dev/nvmexn1:/dev/nvmexn2:/dev/nvmexn3:/dev/nvmexn4:/dev/nvmexn5:/dev/nvmexn6:/dev/nvmexn7:/dev/nvmexn8

# UBSIO 内存池大小，单位 GB。A2 默认配置 10GB。
ubsio.mem.size_in_gb = 10

# A2 推荐内存淘汰水位配置为 0，使数据尽快下沉到 SSD。
ubsio.wcache.evict_water_level = 0

# 应与已启用 DRAM 的 local_service 进程数一致（即ock.mmc.local_service.dram.size非0）
ubsio.standalone.device_count = 8

# A2 可关闭 scratch pool，使 BDM 直接写入上层 buffer。
ubsio.bdm.batch_read.standalone.use_scratch_pool = false
```

## A3 配置示例

```ini
# UBSIO 配置（写入 mmc-local.conf）

# A3 可配置一块或多块独占 SSD，多个路径使用英文冒号分隔。
ubsio.disk.path = /dev/nvmexnx:/dev/nvmexnx

# UBSIO 内存池大小，单位 GB。A3 默认配置 800GB，推荐按 600-800GB 配置。
ubsio.mem.size_in_gb = 800

# A3 推荐内存淘汰水位配置为 85。
ubsio.wcache.evict_water_level = 85

# 应与已启用 DRAM 的 local_service 进程数一致（即 `ock.mmc.local_service.dram.size` 非 `0`）
ubsio.standalone.device_count = 1
```

## 配置表

| 配置项                                                                                                     | 值类型 | 是否必填 | 默认值                  | 有效范围                                  | 说明                                                                         |
| ------------------------------------------------------------------------------------------------------- | --- | ---- | -------------------- | ------------------------------------- | -------------------------------------------------------------------------- |
| [`ubsio.disk.path`](#ubsio.disk.path)                                                                   | 字符串 | 必填   | `xxx:xxx:xxx`（代码占位值） | 最多 64 个有效块设备路径，以英文冒号分隔                | 缓存盘或分区列表；部署时应显式配置有效路径，或留空表示仅使用内存缓存。                                        |
| [`ubsio.mem.size_in_gb`](#ubsio.mem.size_in_gb)                                                         | 整数  | 必填   | `50`                 | `0` 到 `1024`；不得超过系统可用内存               | UBSIO 单进程内存池容量，单位 GB。                                                      |
| [`ubsio.standalone.device_count`](#ubsio.standalone.device_count)                                       | 整数  | 必填   | `0`                  | `0` 到 `16`                            | 已启用 DRAM 的 local_service 进程数；`0` 表示保留旧逻辑。                                  |
| [`ubsio.wcache.evict_water_level`](#ubsio.wcache.evict_water_level)                                     | 整数  | 必填   | `0`                  | `0` 到 `100`                           | 写缓存内存淘汰水位，单位百分比。                                                           |
| `ubsio.wcache.disk_evict_water_level`                                                                   | 整数  | 可选   | `90`                 | `0` 到 `100`                           | 写缓存磁盘淘汰水位，单位百分比。                                                           |
| `ubsio.bdm.io_engine`                                                                                   | 字符串 | 可选   | `sync`               | `sync`、`io_uring`                     | BDM I/O 引擎。                                                                |
| `ubsio.bdm.io_uring.sqpoll_mode`                                                                        | 字符串 | 可选   | `auto`               | `auto`、`required`、`disabled`          | io_uring SQPOLL 模式；`auto` 会在不支持时回退到普通 io_uring。                            |
| `ubsio.bdm.sync.worker_num`                                                                             | 整数  | 可选   | `16`                 | `1` 到 `64`                            | sync 引擎执行 BDM 批量阻塞 I/O 的内部线程数。                                             |
| `ubsio.bdm.batch_read.window_keys`                                                                      | 整数  | 可选   | `128`                | `1` 到 `1024`                          | BatchGet 经 BDM 读盘时单窗口 key 数上限。                                             |
| `ubsio.bdm.batch_read.window_bytes_mb`                                                                  | 整数  | 可选   | `64`                 | `1` 到 `1024`                          | BatchGet 经 BDM 读盘时单窗口字节数上限，单位 MB。                                          |
| `ubsio.bdm.batch_read.pipeline_depth`                                                                   | 整数  | 可选   | `4`                  | `1` 到 `64`                            | BatchGet 经 BDM 读盘时允许同时在途的窗口数。                                              |
| `ubsio.bdm.batch_read.temp_pool_mb`                                                                     | 整数  | 可选   | `0`                  | `0` 到 `65535`                         | BDM 批量读独立临时缓冲池大小，单位 MB；`0` 表示按需申请。                                         |
| [`ubsio.bdm.batch_read.standalone.use_scratch_pool`](#ubsio.bdm.batch_read.standalone.use_scratch_pool) | 布尔值 | 可选   | `true`               | `true`、`false`                        | standalone BatchGet 是否先读入 scratch pool；A2 可关闭，A3 必须开启。                     |
| `ubsio.log.level`                                                                                       | 字符串 | 可选   | `info`               | `error`、`warn`、`info`、`debug`、`trace` | 日志输出级别。                                                                    |
| `ubsio.segment.size_in_mb`                                                                              | 整数  | 可选   | `4`                  | `1` 到 `16`                            | 缓存 segment 大小。单机内存池 block、BDM chunk 和 SDK data-message block 都使用该大小，单位 MB。 |
| `ubsio.sdkmem.size_in_mb`                                                                               | 整数  | 可选   | `0`                  | `0` 到 `4194304`                       | SDK data-message 内存池大小，单位 MB；`0` 表示不预申请。                                   |
| `ubsio.net.tls.enable.switch`                                                                           | 布尔值 | 可选   | `false`              | `true`、`false`                        | TLS 网络安全开关。单机本地部署通常保持为 `false`。                                            |
| `ubsio.trace.enable`                                                                                    | 布尔值 | 可选   | `true`               | `true` 或 `false`                      | 启用或禁用 HTrace 采集。单机模式会初始化 tracer 模块。                                        |
| `ubsio.cache.qos.enable`                                                                                | 布尔值 | 可选   | `false`              | `true`、`false`                        | Cache QoS 和过载控制开关。                                                         |
| `ubsio.rcache.evict_water_level`                                                                        | 整数  | 可选   | `90`                 | `0` 到 `100`                           | 读缓存淘汰水位。内存和磁盘读缓存淘汰阈值使用同一个值。                                                |
| `ubsio.cache.mem_read_write_ratio`                                                                      | 字符串 | 可选   | `0:10`               | 两个 `0` 到 `10` 的整数，和为 `10`             | 内存读缓存与写缓存资源配比。                                                             |
| `ubsio.cache.disk_read_write_ratio`                                                                     | 字符串 | 可选   | `0:10`               | 两个 `0` 到 `10` 的整数，和为 `10`             | 磁盘读缓存与写缓存资源配比。                                                             |
| `ubsio.work.io.timeout`                                                                                 | 整数  | 可选   | `60`                 | `60` 到 `300`                          | 通过单机 runtime config 传给 SDK 的 I/O 超时时间，单位为秒。                                |
| `ubsio.batchget.thread.num`                                                                             | 整数  | 可选   | `32`                 | `8` 到 `512`                           | MirrorServer BatchGet executor 线程数。                                        |
| `ubsio.cli_tools.enable`                                                                                | 布尔值 | 可选   | `false`              | `true` 或 `false`                      | 启用 CLI agent 诊断能力。使用 `cli_client --attach` 连接 `bio_console` 时应设为 `true`。   |
| `ubsio.underfs.file_system_type`                                                                        | 字符串 | 可选   | `none`               | `ceph`、`hdfs`、`none`                  | UnderFS 后端类型；单机本地部署使用 `none`。                                              |

# 详细配置项说明

## ubsio.disk.path

### 配置含义与示例

`ubsio.disk.path` 用于指定三级池化缓存层使用的块设备或分区，决定是否启用本地 SSD 缓存层。可配置整盘、已有分区或 loop 设备；多个路径使用英文冒号（`:`）分隔，最多 **64** 条。**留空表示无盘模式，仅使用 `ubsio.mem.size_in_gb` 配置的本地内存池，重启后缓存数据不持久化。**

```ini
# 整盘
ubsio.disk.path = /dev/nvme0n1

# 多个分区
ubsio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2

# loop 设备
ubsio.disk.path = /dev/loop2

# 无盘模式
ubsio.disk.path =
```
---

## ubsio.standalone.device_count

`ubsio.standalone.device_count` 表示参与本地缓存路径分配的 device 数量。**为简化表述，下文使用 `device_count` 代指该配置项。其值应与已启用 DRAM 的 local_service 进程数一致；这里“已启用 DRAM”是指 `ock.mmc.local_service.dram.size` 非 `0`。**

- `0`：兼容旧逻辑。每个 deviceId 按 `ubsio.disk.path` 的路径下标选择设备，例如 `deviceId = 1` 选择第 2 条路径。
- `1`：直接配置目标整盘，无需分区。
- 大于 `1`：需要执行[分区操作](#分区操作)，并按配置的 device 数量对 `ubsio.disk.path` 路径做均衡分配。每个 deviceId 至少分配 1 个分区，分区总数不得小于 `device_count`。建议每块盘切分为 `device_count` 个相同容量的分区，即分区总数为 `device_count × 盘数`。

示例：有 4 个已启用 DRAM 的 local_service 进程时，`device_count` 配置为 `4`。2 块盘共切出 8 个分区时，每个 deviceId 分配 2 个分区。

```ini
ubsio.disk.path = /dev/nvmexn1p1:/dev/nvmexn1p2:/dev/nvmexn1p3:/dev/nvme0n1p4:/dev/nvmeyn1p1:/dev/nvmeyn1p2:/dev/nvmeyn1p3:/dev/nvmeyn1p4
ubsio.standalone.device_count = 4
```

没有已启用 DRAM 的 local_service 进程时，配置为 `0`：

```ini
ubsio.disk.path =
ubsio.standalone.device_count = 0
```

只有 1 个已启用 DRAM 的 local_service 进程且使用磁盘时，配置为 `1`，直接使用目标整盘，无需分区：

```ini
ubsio.disk.path = /dev/nvmexn1
ubsio.standalone.device_count = 1
```

---

## ubsio.mem.size_in_gb

`ubsio.mem.size_in_gb` 表示 UBSIO 可使用的本地内存池容量，单位为 GB。该值需要结合 A2 / A3 场景和节点剩余内存规划，不能超过推理进程、Memcache 等业务进程占用后可留给 UBSIO 的内存空间。

### A2

A2 场景下，UBSIO 通常随推理进程启动，每个推理进程中的 UBSIO 都会使用本地缓存资源，`ubsio.mem.size_in_gb` 建议默认配置为 `10`。

```ini
# 示例：A2 默认配置 10GB
ubsio.mem.size_in_gb = 10
```

### A3

A3 场景下，UBSIO 随 memcache 独立进程启动，通常只有一个 UBSIO 服务。

```ini
# 示例：A3 默认配置 800GB，推荐按 600-800GB 配置
ubsio.mem.size_in_gb = 800
```

---

## ubsio.wcache.evict_water_level

`ubsio.wcache.evict_water_level` 是写缓存的内存淘汰水位，取值范围 `0` 到 `100`，表示写缓存内存容量使用到多少百分比后开始触发淘汰策略。

无盘模式下没有 SSD 承接写缓存淘汰，通常设置为较高水位，让缓存尽量留在内存中；当前最小单机模式示例使用 `85`：

```ini
ubsio.disk.path =
ubsio.wcache.evict_water_level = 85
```

有盘模式下，如果希望写缓存更快进入 SSD 层，可设置为较低水位，例如：

```ini
ubsio.disk.path = /dev/nvmexn1p1:/dev/nvmexn1p2:/dev/nvmexn1p3:/dev/nvmexn1p4
ubsio.wcache.evict_water_level = 0
```

配置建议：无盘场景优先使用较高水位；有盘场景可根据内存压力和 SSD 容量降低该值。

---

## ubsio.bdm.batch_read.standalone.use_scratch_pool

`ubsio.bdm.batch_read.standalone.use_scratch_pool` 控制 standalone BatchGet 经 BDM 读盘时是否先将数据读入 scratch pool，再复制到上层 buffer。默认值为 `true`。

### A2

A2 场景可以关闭 scratch pool，使 BDM 直接写入上层 buffer：

```ini
ubsio.bdm.batch_read.standalone.use_scratch_pool = false
```

### A3

A3 场景必须开启 scratch pool，不允许关闭：

```ini
ubsio.bdm.batch_read.standalone.use_scratch_pool = true
```

---

# 其他

## 分区操作

推荐使用 `partition_disks.sh` 进行分区，具体用法见[分区脚本使用指导](#分区脚本使用指导)。

分区操作统一按 `device_count` 规划；该值应与已启用 DRAM 的 local_service 进程数一致。
- `device_count = 1` 时直接配置目标整盘，无需分区；
- 大于 `1` 时，每个 deviceId 至少分配 1 个分区，分区总数不得小于 `device_count`。
建议每块盘切分为 `device_count` 个相同容量的分区，即分区总数为 `device_count × 盘数`。

### 分区脚本使用指导

> <span style="color:red">**⚠️ 高危操作：选错磁盘将造成不可逆的数据丢失。请务必在执行前通过 `lsblk` 反复确认目标磁盘路径。**</span>

**[partition_disks.sh](https://gitcode.com/openeuler/ubs-io/blob/develop/scripts/ubsio-boostio/partition_disks.sh)脚本下载：**

```bash
wget -O /path/to/partition_disks.sh https://raw.gitcode.com/openeuler/ubs-io/blobs/b81de180c6a15059a28155f17c8c96f502ef70cf/partition_disks.sh
chmod +x /path/to/partition_disks.sh
```

脚本创建 GPT 分区表，在每块盘上创建相同数量、相同容量的分区，并输出 `ubsio.disk.path`。默认 dry-run，加 `--yes` 才写入。

执行 `partition_disks.sh` 后会输出可直接使用的配置行，例如：

```ini
ubsio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2:...
```

将实际输出的配置行按需写入 `mmc-local.conf` 即可。

| 参数                    | 说明                                                        |
| --------------------- | --------------------------------------------------------- |
| `-n, --parts N`       | 总分区数，建议配置为 `device_count × 盘数`；最大不超过 64              |
| `-d, --disk PATH`     | 磁盘路径；多块磁盘时重复配置                                           |
| `-c, --dev-cnt N`     | device 数，应与 `device_count` 一致                 |
| `-y, --yes`           | 实际写入分区表                                                   |
| `-f, --force`         | 覆盖已有分区表                                                   |
| `-w, --wipe-metadata` | 擦除旧文件系统签名和 BDM 头                                          |

### 场景一：有盘部署（推荐）

**背景：** 单节点一块或多块 NVMe 盘，分区总数需提前规划，每个分区作为一个本地缓存路径。

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
```

#### 2. 典型场景

以下示例按推荐值配置：单盘时 `--parts = device_count`；多盘时 `--parts = device_count × 盘数`。

**`device_count = 1`：直接使用目标盘，不分区**

```ini
ubsio.disk.path = /dev/nvme0n1
ubsio.standalone.device_count = 1
```

**`device_count = 4`：2 块盘，每块盘 4 个分区，共 8 个分区**

```bash
sudo bash /path/to/partition_disks.sh \
    --parts 8 \
    --disk /dev/nvme0n1 \
    --disk /dev/nvme1n1 \
    -c 4 --yes -f -w

# 输出：
# ubsio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2:/dev/nvme0n1p3:/dev/nvme0n1p4:/dev/nvme1n1p1:/dev/nvme1n1p2:/dev/nvme1n1p3:/dev/nvme1n1p4
```

**`device_count = 8`：1 块盘，8 个分区**

```bash
sudo bash /path/to/partition_disks.sh \
    --parts 8 --disk /dev/nvme0n1 -c 8 --yes -f -w

# 输出：
# ubsio.disk.path = /dev/nvme0n1p1:/dev/nvme0n1p2:/dev/nvme0n1p3:/dev/nvme0n1p4:/dev/nvme0n1p5:/dev/nvme0n1p6:/dev/nvme0n1p7:/dev/nvme0n1p8
```

---

### 场景二：模拟盘部署（loop 设备方案）

无可用裸盘时，可用 `dd` + `losetup --find --show` 创建 loop 块设备模拟，由系统自动选择空闲 loop 设备，默认开启 direct_io 直通模式，并按 `device_count` 规划分区。

> <span style="color:red">**⚠️ 高危操作：创建 loop 设备前，建议先确认当前 loop 设备占用情况；不要手动复用已被占用的 `/dev/loopN`，否则可能导致数据丢失。**</span>

创建 1TB 镜像，并自动分配空闲 loop 设备；分区步骤使用 [分区脚本使用指导](#分区脚本使用指导) 中的 `partition_disks.sh`：

```bash
# 创建 1TB 镜像文件（count=1024，可根据需求调整）
sudo dd if=/dev/zero of=/data/boostio_disk.img bs=1G count=1024 status=progress

# 挂载 loop 设备，默认开启 direct_io 直通；命令会输出实际分配的设备路径
LOOP_DEV=$(sudo losetup --find --show --direct-io=on /data/boostio_disk.img)
echo "${LOOP_DEV}"
# 示例输出：/dev/loop2

# 分区
sudo bash /path/to/partition_disks.sh \
    --parts 4 --disk "${LOOP_DEV}" -c 4 --yes -f -w

# 输出：
# ubsio.disk.path = /dev/loop2p1:/dev/loop2p2:/dev/loop2p3:/dev/loop2p4
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

| 事项    | 说明                                     |
| ----- | -------------------------------------- |
| 磁盘独占  | UBSIO 独占配置的磁盘/分区，不可与其他服务共享   |
| 物理盘隔离 | 同一物理盘的分区不会分配给同一 deviceId（算法自动打散）       |
| 路径上限  | `ubsio.disk.path` 最多 64 条路径            |
| 动态加盘  | standalone 模式不支持运行时加盘                  |
| 复用磁盘  | 不同 deviceId 会写入 BDM disk head，不匹配将拒绝恢复 |
| 权限    | 进程需对块设备有读写权限，建议 root 运行                |
| 磁盘规格  | 读写带宽建议不低于 7GB/s                        |
