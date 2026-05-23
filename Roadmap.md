# <span style="color:#008a00;">MemCache Roadmap 2026 H1</span>

MemCache在2026年H1阶段的目标是增强在多级缓存方面的能力、持续提升集群可靠性和易用性，具体的研发Roadmap如下：

---

## 关键特性

### <span style="color:#0066cc;">多级缓存能力</span>

- [ ] [**630**] 支持HBM+DRAM+SSD三层缓存，已提供POC测试版本，630推出新架构版本
- [x] [**2025H4**] 支持HBM + DRAM两级缓存
- [x] [**2025H3**] 支持DRAM单级缓存

### <span style="color:#0066cc;">集群可靠性</span>

- [x] [**2025H4**] 支持Meta Service高可用，支持主备切换和元数据恢复
- [x] [**2025H4**] 支持Local Service动态扩缩，支持数据多副本放置策略

### <span style="color:#0066cc;">可维测</span>

- [x] [**130**] Meta Service支持基于[restful](https://gitcode.com/Ascend/memcache/blob/master/doc/memcache_restful_api.md)进行集群管理、数据查询和指标查询，支持对接Prometheus和Grafana

---

## <span style="color:#008a00;">版本发布</span>
**release** 开头的分支是我们发布的稳定分支。比如**release/1.1**，第一个版本为v1.1.0，有bugfix修复，则版本递增v1.1.1/v1.1.2等。
每发布一个版本，会有对应的tag和相关的release note。具体规则可以参考：[开发与发布节奏原则](开发与发布节奏原则.md)
		
- [x] [**release/1.1**]  已于522已推出v1.1.0
- [x] [**v1.0.0**] 1.0.x稳定分支，只会合入必要修改的bugfix，在必要bugfix合入之后，会推送到[pypi](https://pypi.org/project/memcache-hybrid/#history)，对应版本号 1.0.x，x为从0开始递增的数字

---
