#  MemCache SSD 缓存特性介绍
## 1. 背景与动机

MemCache 是面向 LLM 推理的高性能分布式 KV Cache 引擎，基于 MemFabric 池化内存架构构建。在长上下文推理场景中，单次请求的 KV Cache 数据量可达数十 GB，纯 HBM + DRAM 两级缓存成本极高。

SSD 缓存是 MemCache 新增的第三级存储特性，通过对接外部 UBS IO 键值存储引擎，利用 NVMe 本地盘的大容量、低成本优势，将访问频率低的冷数据自动下沉到 SSD，内存中仅保留热数据。在获得接近全内存缓存体验的同时，显著降低单位容量成本。


 **适用场景:**

- **长上下文 LLM 推理**：单个请求 KV Cache 可达几十 GB，总缓存需求远超内存，SSD 可承接大量冷上下文数据
- **多轮对话**：历史轮次的 KV Cache 访问频率下降后可自然下沉到 SSD，释放内存给新请求
- **批处理 / 离线推理**：对延迟不极端敏感，优先追求吞吐和缓存容量
- **成本敏感部署**：相比堆内存，用 NVMe SSD 扩展缓存容量更具性价比

## 2. 三级缓存架构

系统按性能和容量将缓存分为三个层级，对用户完全透明：

```
┌──────────────┐  介质            容量特征      核心优势
│   HBM (L1)   │  NPU 高带宽内存    ~数十 GB     极致带宽，近存计算
├──────────────┤
│  DRAM (L2)   │  主机 DDR 内存     ~数百 GB     容量与带宽的平衡点
├──────────────┤
│   SSD (L3)   │  NVMe 本地盘       ~数 TB       低成本大容量持久化
└──────────────┘
```

用户在调用 `Put` / `Get` 接口时无需关心数据实际驻留在哪一层。

## 3. 部署

### 3.1 安装

```bash
# memfabric_hybrid 安装
pip install memfabric_hybrid==1.2.0

# memcache_hybrid 安装（UBSIO 已集成在 MemCache 中，无需单独安装）
pip install memcache_hybrid==1.2.0
```

MemCache 的安装位置会随 Python 版本、虚拟环境和安装方式变化，使用以下命令动态获取配置目录：

```bash
MMC_CONFIG_DIR="$(python3 -c 'from pathlib import Path; import memcache_hybrid; print(Path(memcache_hybrid.__file__).resolve().parent / "config")')"
echo "$MMC_CONFIG_DIR"
```

配置文件路径分别为：

| 内容                      | 路径                              |
| ------------------------- | --------------------------------- |
| 配置文件（`mmc-meta.conf`）  | `$MMC_CONFIG_DIR/mmc-meta.conf`   |
| 配置文件（`mmc-local.conf`） | `$MMC_CONFIG_DIR/mmc-local.conf`  |

### 3.2 修改配置文件

#### 配置文件 `mmc-local.conf`：

```ini
# 启用 SSD 池化特性开关
ock.mmc.local_service.storage.enabled = true

# 配置 UBS IO 用作本地 SSD 缓存的块设备路径，支持整盘、分区或 loop 设备；多个路径使用英文冒号分隔；配置的设备必须由 UBS IO 独占，且不能存在挂载点
ubsio.disk.path = /dev/nvmexn1:/dev/nvmexn2:/dev/nvmexn3:/dev/nvmexn4:/dev/nvmexn5:/dev/nvmexn6:/dev/nvmexn7:/dev/nvmexn8

# UBS IO 单进程内存池大小，单位 GB。典型配置：A3分离模式配置为80GB。其他情况下配置为10GB。
ubsio.mem.size_in_gb = 10

# 一次VLLM启动中，启用了SSD池化开关的memcache local_service进程数量。例如A3分离模式固定填1，其他情况均填写VLLM进程数量。
ubsio.standalone.device_count = 8

# 强制将配置的 SSD 设备初始化为新磁盘
ubsio.standalone.force_new_disk = true
```

| 参数                                      | 含义                   | 默认配置    | 说明                                                                                                                                                                                                                                        |
| --------------------------------------- | -------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ock.mmc.local_service.storage.enabled` | SSD 缓存开关             | `false` | 默认关闭；设置为 `true` 后启用 SSD 多级池化                                                                                                                                                                                                              |
| `ubsio.disk.path`                       | UBS IO 本地 SSD 缓存设备路径 | `空`     | 指定 UBS IO 用作本地 SSD 缓存的设备路径，支持整盘、分区或 loop 设备；多个路径使用英文冒号分隔。<span style="color:red">**配置的设备必须由 UBS IO 独占，且不能存在挂载点。**</span> |
| `ubsio.mem.size_in_gb`                  | UBS IO 单进程内存池大小      | `50`    | 取值范围为 `0`～`3072` 的整数；SSD 场景至少配置 `5` GB，推荐显式配置为 `10` GB。单机UBSIO占用内存总量为单进程内存池大小 乘以 启用了SSD池化的进程数。UBSIO占用内存总量要 **小于** 节点减去操作系统固定内存、推理服务活动内存和 MemCache DRAM 后的剩余内存。典型配置：A3分离模式配置为80GB。其他情况下配置为10GB。 |
| `ubsio.standalone.device_count`         | UBSIO进程总数 | `0`     | 启用了SSD池化开关的memcache local_service进程数量，A3分离模式只有独立进程会启用SSD池化开关，因此固定填1，其他情况所有VLLM进程都应该启动SSD池化，因此填VLLM进程数量。  |
| `ubsio.standalone.force_new_disk`       | 是否强制初始化为新磁盘          | `false` | 设置为 `true` 时，UBS IO 将配置的 SSD 设备作为新磁盘初始化，不恢复已有元数据。当前版本不支持故障恢复，因此需设置为 `true`。                                                 |

`ubsio.mem.size_in_gb` 表示每个启用 SSD池化 的 `local_service` 进程创建的 UBS IO 内存池大小。单进程可配置上限的计算方式如下：

```text
单进程可配置上限 = min(3072, floor(节点可供 UBS IO 使用的剩余内存（GB） / 已启用 DRAM 的 local_service 进程数))
```

例如，节点可供 UBS IO 使用的剩余内存为 `200` GB，已启用 SSD池化 的 `local_service` 进程数为 `4`，则单进程可配置上限为 `50` GB，推荐值 `10` GB 在有效范围内。如果计算出的上限小于 `5` GB，需要释放更多内存或减少进程数。

对于 MemCache 分离部署场景，单进程建议配置为 `50` GB；其他场景建议配置为 `10` GB。需要使用 L2.5 能力时，可根据实际需求增大该值，同时需要调整 [`ubsio.wcache.evict_water_level`](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md#ubsiowcacheevict_water_level)。

UBS IO 详细配置项及参数说明，请参见 [DRAM + SSD 多级池化配置指南](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md)。

#### 配置文件 `mmc-meta.conf`：

```ini
# 空间使用率达到高水位时，Put 操作触发数据淘汰
ock.mmc.evict_threshold_high = 70

# 数据淘汰完成后的目标空间使用率，即低水位
ock.mmc.evict_threshold_low = 60

# DRAM 空间使用率达到该水位时，淘汰时会快速删除数据释放内存，降低内存占用压力。
ock.mmc.rewarm.dram_watermark = 95
```

| 参数                            | 含义                       | 默认配置 | 说明                                                                                                  |
| ------------------------------- | -------------------------- | -------- | ----------------------------------------------------------------------------------------------------- |
| `ock.mmc.evict_threshold_high`  | 触发淘汰的空间使用率高水位 | `90`     | 取值范围为 `1`～`99` 的整数；空间使用率达到该水位时，`Put` 操作触发数据淘汰。单次 `Put` 的数据量超过容量的 1% 时，无法触发淘汰 |
| `ock.mmc.evict_threshold_low`   | 淘汰完成后的目标空间使用率 | `80`     | 取值范围为 `1`～`98` 的整数；数据淘汰至该水位后停止   |
| `ock.mmc.rewarm.dram_watermark` | DRAM 高压水位    | `95`     | 取值范围为 `0`～`100` 的整数；DRAM 空间使用率达到该水位时，淘汰时会快速删除数据释放内存，降低内存占用压力。 |

> MemCache 仅控制 SSD 的启用开关和回温策略，SSD 的具体容量、淘汰水位等由 UBS IO 参数控制。


#### 使用

**部署样例参考：**

A2：[MMC最佳实践—GLM-5.1+A2-8机PD分离](https://gitcode.com/Ascend/memcache/wiki/MMC%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E2%80%94GLM-5.1+A2-8%E6%9C%BAPD%E5%88%86%E7%A6%BB.md)

A3：[MemCache+vLLM+A3分离部署案例](https://gitcode.com/Ascend/memcache/wiki/MemCache+vLLM+A3%E5%88%86%E7%A6%BB%E9%83%A8%E7%BD%B2%E6%A1%88%E4%BE%8B.md)

#### 卸载

```bash
pip uninstall memcache_hybrid
```


## 4. 限制与注意事项

- SSD 缓存依赖本地 NVMe 盘，需确保节点上有可用 SSD 设备
- SSD 延迟（毫秒级）明显高于内存，对延迟极度敏感的实时推理场景建议优先扩容 DRAM
- 启用 SSD 缓存需要系统中装有 ubs-io 运行时库
- SSD 大IO随机读带宽不低于 7GB/s，压测方法见 [磁盘带宽压测指南](https://gitcode.com/zhangjialee/ubs-io/wiki/%E7%A3%81%E7%9B%98%E5%B8%A6%E5%AE%BD%E5%8E%8B%E6%B5%8B%E6%8C%87%E5%8D%97.md)
