# <span style="color:#008a00;">MemCache Roadmap: 2026 H1</span>

In the first half of 2026, MemCache aims to strengthen multi-tier caching while continuing to improve cluster reliability and ease of use. The development roadmap is:

---

## Key Features

### <span style="color:#0066cc;">Multi-Tier Caching</span>

- [ ] [**630**] Support HBM + DRAM + SSD three-tier caching. A POC build is available; release 630 will introduce the new architecture.
- [x] [**2025H4**] Support two-tier HBM + DRAM caching.
- [x] [**2025H3**] Support single-tier DRAM caching.

### <span style="color:#0066cc;">Cluster Reliability</span>

- [x] [**2025H4**] Support Meta Service high availability, including primary/standby failover and metadata recovery.
- [x] [**2025H4**] Support dynamic Local Service scaling and multi-replica data placement.

### <span style="color:#0066cc;">Operations and Observability</span>

- [x] [**130**] Support cluster management, data queries, and metric queries through the [RESTful API](https://gitcode.com/Ascend/memcache/blob/master/docs/zh/memcache_restful_api.md), with Prometheus and Grafana integration.
