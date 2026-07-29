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

## 3. 编译

基础编译安装流程参见 [run包编译、安装及使用](./install_run.md)。当前默认以 RELEASE 模式编译并开启 UBS IO，直接执行以下命令即可：

### run 包

#### 编译

```bash
bash script/build_and_pack_run.sh --build_mode RELEASE --build_ubsio ON # 追加 --build_ubsio ON
```

构建过程中，CMake 自动从 [ubs-io](https://gitcode.com/openeuler/ubs-io) 仓库的 `openEuler-24.03-LTS-SP4` 分支拉取源码并编译 UBS IO，相关产物随 run 包一起打包。

## 4. 部署

### 4.1 安装

```bash
bash memcache_hybrid-{version}_linux_{arch}.run
source /usr/local/memcache_hybrid/set_env.sh
```

安装后路径：

| 内容                      | 路径                                                            |
| ----------------------- | ------------------------------------------------------------- |
| UBS IO 运行时库             | `<install_path>/memcache_hybrid/latest/{arch}-linux/lib64/`   |
| 配置文件（`mmc-meta.conf`）  | `<install_path>/memcache_hybrid/latest/config/mmc-meta.conf`  |
| 配置文件（`mmc-local.conf`） | `<install_path>/memcache_hybrid/latest/config/mmc-local.conf` |

### 4.2 修改配置文件

#### 配置文件 `mmc-local.conf`：

```ini
# 启用 SSD 缓存
ock.mmc.local_service.storage.enabled = true

# 本地 SSD 缓存盘或分区路径，多个路径使用英文冒号分隔；必须由 UBS IO 独占，且不能存在挂载点
ubsio.disk.path = /dev/nvmexn1:/dev/nvmexn2:/dev/nvmexn3:/dev/nvmexn4:/dev/nvmexn5:/dev/nvmexn6:/dev/nvmexn7:/dev/nvmexn8

# UBS IO 单进程内存池大小，单位 GB
ubsio.mem.size_in_gb = 10

# 参与本地缓存路径分配的数量，其值应与已启用 DRAM 的 local_service 进程数一致（即ock.mmc.local_service.dram.size 非 0 的进程数）
ubsio.standalone.device_count = 8
```

| 参数                                      | 含义                   | 默认配置    | 说明                                                                                                                                                                                                                                        |
| --------------------------------------- | -------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ock.mmc.local_service.storage.enabled` | SSD 缓存开关             | `false` | 默认关闭；设置为 `true` 后启用 SSD 多级池化                                                                                                                                                                                                              |
| `ubsio.disk.path`                       | 本地 SSD 缓存盘或分区路径      | `空`     | 多个路径使用英文冒号分隔。**配置的磁盘或分区必须由 UBS IO 独占，且不能存在挂载点。**分区操作参考：[分区操作](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md#%E5%88%86%E5%8C%BA%E6%93%8D%E4%BD%9C) |
| `ubsio.mem.size_in_gb`                  | UBS IO 单进程内存池大小      | `50`    | 取值范围为 `0`～`3072` 的整数；SSD 场景至少配置 `5` GB，推荐显式配置为 `10` GB。所有进程的配置总和不能超过节点扣除操作系统、推理服务和 MemCache DRAM 后可供 UBS IO 使用的剩余内存。                                                                                                                      |
| `ubsio.standalone.device_count`         | standalone device 数量 | `0`     | 与 `ock.mmc.local_service.dram.size` 非 `0` 的 local_service 进程数一致                                                                                                                                                                           |

`ubsio.mem.size_in_gb` 表示每个启用 DRAM 的 `local_service` 进程可使用的 UBS IO 内存池大小。单进程可配置上限的计算方式如下：

```text
单进程可配置上限 = min(3072, floor(节点可供 UBS IO 使用的剩余内存（GB） / 已启用 DRAM 的 local_service 进程数))
```

例如，节点可供 UBS IO 使用的剩余内存为 `200` GB，已启用 DRAM 的 `local_service` 进程数为 `4`，则单进程可配置上限为 `50` GB，推荐值 `10` GB 在有效范围内。如果计算出的上限小于 `5` GB，需要释放更多内存或减少进程数。

对于 SSD 场景，单进程配置 `10` GB 通常已经足够，无需配置过大。仅在需要使用 L2.5 能力时考虑增大该值，同时需要调整 [`ubsio.wcache.evict_water_level`](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md#ubsiowcacheevict_water_level)。

UBS IO 详细配置项及参数说明，请参见 [DRAM + SSD 多级池化配置指南](https://gitcode.com/Ascend/memcache/wiki/DRAM%20+%20SSD%20%E5%A4%9A%E7%BA%A7%E6%B1%A0%E5%8C%96%E9%85%8D%E7%BD%AE%E6%8C%87%E5%8D%97.md)。

#### 配置文件 `mmc-meta.conf`：

```ini
# 空间使用率达到高水位时，Put 操作触发数据淘汰
ock.mmc.evict_threshold_high = 70

# 数据淘汰完成后的目标空间使用率，即低水位
ock.mmc.evict_threshold_low = 60

# DRAM 空间使用率低于该水位时，触发 SSD 数据回温至 DRAM
ock.mmc.rewarm.dram_watermark = 95
```

| 参数                            | 含义                       | 配置值 |
| ------------------------------- | -------------------------- | ------ |
| `ock.mmc.evict_threshold_high`  | 触发淘汰的空间使用率高水位 | `70`   |
| `ock.mmc.evict_threshold_low`   | 淘汰完成后的目标空间使用率 | `60`   |
| `ock.mmc.rewarm.dram_watermark` | DRAM 回温水位               | `95`   |

> MemCache 仅控制 SSD 的启用开关和回温策略，SSD 的具体容量、淘汰水位等由 UBS IO 参数控制。


#### 使用

**部署样例参考：**

A2：[MMC最佳实践—GLM-5.1+A2-8机PD分离](https://gitcode.com/Ascend/memcache/wiki/MMC%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5%E2%80%94GLM-5.1+A2-8%E6%9C%BAPD%E5%88%86%E7%A6%BB.md)

A3：[MemCache+vLLM+A3分离部署案例](https://gitcode.com/Ascend/memcache/wiki/MemCache+vLLM+A3%E5%88%86%E7%A6%BB%E9%83%A8%E7%BD%B2%E6%A1%88%E4%BE%8B.md)

#### 卸载

```bash
bash <install_path>/memcache_hybrid/latest/uninstall.sh
```


## 5. 限制与注意事项

- SSD 缓存依赖本地 NVMe 盘，需确保节点上有可用 SSD 设备
- SSD 延迟（毫秒级）明显高于内存，对延迟极度敏感的实时推理场景建议优先扩容 DRAM
- 启用 SSD 缓存需要系统中装有 ubs-io 运行时库
- SSD 随机读带宽不低于 7GB/s，压测方法见 [磁盘带宽压测指南](https://gitcode.com/zhangjialee/ubs-io/wiki/%E7%A3%81%E7%9B%98%E5%B8%A6%E5%AE%BD%E5%8E%8B%E6%B5%8B%E6%8C%87%E5%8D%97.md)