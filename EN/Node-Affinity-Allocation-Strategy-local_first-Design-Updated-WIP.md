# Node-Affinity Allocation Strategy (local_first) Design

**Authors:** @huawei_zixiqu  
**Created:** 2026-10-08  
**Updated:** 2026-10-08

***

# 1. Overview

## 1.1 Introduction

MemCache's existing ALLOC_RANDOM strategy distributes the keys in a put request across all ranks in the cluster at key granularity. For the vLLM KV blocks above it, blocks in one conversation form a **strict dependency chain** (each block depends on the preceding block). If any block is lost because its rank fails, the entire subsequent chain becomes unusable. Random distribution at large scale (for example, 48 ranks) creates a **very high single-node failure blast radius**: failure of any machine can reduce the cache hit rate for all requests in the cluster to zero at once.

This proposal adds the **node-affinity allocation strategy**, ALLOC_LOCAL_FIRST, upgrading allocation from “randomly distribute by key” to “allocate by backendId (node)”:

- **Local first:** When a writer instance (P, role=writer) allocates, it selects a target backendId according to the distribution-table weights. Its own (local) share is high and shares for other nodes are lower, while the overall distribution remains balanced. This preserves distribution while greatly reducing the failure blast radius.
- **Evenly route writer traffic to memory nodes:** Memory-only contributor instances (D, role=memory) receive almost no writes. The distribution table routes writer data to memory nodes according to weights (for example, in a 3P1D deployment, it can route pressure from a failed P side to an idle D side), balancing load across instances.
- **Randomize within a backendId:** Shuffle all ranks in the target backendId on every allocation to avoid hot spots caused by a fixed order.

## 1.2 Motivation

Current issue: ALLOC_RANDOM distributes keys across all ranks, balancing load but ignoring the dependency chain between KV blocks and therefore creating a large failure blast radius. An earlier WIP implementation of ALLOC_LOCAL_FIRST (sequential ring failover across peer nodes) had three shortcomings:

1. **No randomization:** Rank order within a node was fixed (based on unordered_set iteration), concentrating hits on the first rank and creating a hot spot.
2. **Failover is not guided:** Cross-node failover followed backendId order, so in a 3P1D deployment (P writes and D is nearly idle) it could not actively route write pressure to the idle D side.
3. **Identity is coupled to preferences:** Meta inferred the caller from preferredRank_[0]; an explicit preferred rank from the caller could break local-direction detection.

## 1.3 Goals

**Goals:**

- Node affinity: Distribute writer data to target backendIds by weight, keeping the largest share local and routing the remainder evenly. Reduce the failure blast radius from the entire cluster to the set of target nodes.
- Even routing to memory: Writers use the distribution table to write some data evenly to nodes with role=memory, filling the D side while balancing write pressure across writer instances.
- Randomize within a group: Randomly shuffle the ranks in the target backendId on each allocation (thread_local mt19937) to prevent hot spots.
- Make identity explicit: Include the caller rank in the RPC; allocation no longer depends on preferredRank_ contents.
- Fallback: If allocation on the target backendId fails, fall back to global random allocation to preserve availability. Expose fallback frequency through counters and logs.
- Backward compatibility: Without the new fields, behavior remains the same as the original ALLOC_RANDOM. If role is omitted, default to writer so existing deployments continue to work without changes.

**Out of scope:**

- Multiple replicas (replicaNum > 1) are outside this design.
- Data migration and recovery are not included; recovery after a failure is an HA capability.
- Public API semantics for put/get/remove are unchanged; this enhances internal scheduling without a breaking API change.

# 2. Use-Case Analysis (Requirements)

| Use case | Priority | Functional and performance goal | Notes |
| --- | --- | --- | --- |
| Local-first allocation | P0 | Place writer data on target backendIds according to the distribution weights, favoring the local node and shuffling ranks within a group | ALLOC_LOCAL_FIRST; std::shuffle within ExpandLocalFirstRanks |
| Evenly route writers to memory on non-writer nodes | P0 | In 3P1D, route some P-side data to D according to weights, filling D and balancing writes across writer instances | Greedy ledger in BuildBackendDistribution plus local-affinity exchange |
| Automatically rebuild the distribution table | P0 | Recompute from the current set of backendIds after LocalService registration or deregistration, without manual configuration | RebuildWriteDistributionTable, write lock, ptracer instrumentation |
| Fallback | P0 | If allocation on every rank of the target backendId fails, fall back to global random allocation without failing the write | Retry with ALLOC_RANDOM in InnerAlloc; count local_first_fallback |
| Make caller identity explicit | P1 | Carry the caller rank in the RPC instead of inferring it from preferredRank_ | Add rankId_ to AllocRequest/BatchAllocRequest |
| Observable and measurable | P2 | Add fallback count to metric (/metrics/summary) and instrument distribution-table build time with ptracer | local_first_fallback=N, TP_MMC_META_BUILD_DIST_TABLE |

# 3. Design

## 3.1 Overall Design

The allocation strategy runs on the MetaService side and is transparent to the Client (it reuses the existing alloc_strategy flag upload path). It has three layers:

```
┌─ Report during LocalService registration ─────────────────┐
│  BmRegister(rank, mediaType, bm, ..., backendId, role)     │
│    backendId = node connection IP/hostname (shared by     │
│                multiple instances on the same machine)    │
│    role      = "writer"(P) / "memory"(D)                  │
└──────────────────────────┬────────────────────────────────┘
                           ▼
┌─ MetaService topology and distribution table ─────────────┐
│  backendIdRoleMap_       (backendId -> role)               │
│  writeDistributionTable_ (writer backendId -> target      │
│                           weight table)                   │
│  Register/deregister -> RebuildWriteDistributionTable()   │
│  Many reads, few writes -> shared reads, exclusive writes │
└──────────────────────────┬────────────────────────────────┘
                           ▼
┌─ Allocation path ─────────────────────────────────────────┐
│  Put/BatchPut -> HandleAlloc/HandleBatchAlloc              │
│  -> ExpandLocalFirstRanks:                                 │
│      SelectTargetBackendForRank(callerRank)  # weighted   │
│      -> GetRanksForBackendId(target) -> shuffle (within)  │
│  -> BatchAlloc -> InnerAlloc(ALLOC_LOCAL_FIRST)            │
│     -> LocalFirstAssign (try preferredRank_ in order)     │
│     -> all fail -> ALLOC_RANDOM fallback + counter        │
└───────────────────────────────────────────────────────────┘
```

## 3.2 Technology Choices

**Target selection:**

| Approach | Advantage | Disadvantage | Decision |
| --- | --- | --- | --- |
| Fixed local target + ring failover in backendId order | Simple; reuses the original WIP | No randomization; cannot guide failures to the D side | Rejected |
| Weight table (BuildBackendDistribution) | Represents “high local weight + even routing to memory” and is self-balancing | Must be recomputed on registration/deregistration | **Selected** |

**Building the distribution table:** Two-stage greedy algorithm. Stage one uses a ledger to conserve the total share assigned to each backendId and give each writer the same quota (quota = number of backendIds). Stage two exchanges shares to improve local affinity so each writer writes as much as possible to itself. An O(n) traversal approaches the optimum, where n is the number of backendIds.

**Random source within a group:** thread_local std::mt19937 (an independently and randomly seeded generator per thread) avoids lock contention on the hot path and changes the allocation order on every call. RanksToString supports debug output of the expanded result.

**Identity propagation:** Add a uint32_t rankId_ field to AllocRequest/BatchAllocRequest. The Client fills it with its process's RankId() and Meta uses it directly instead of reading preferredRank_[0].

## 3.3 Functional and Performance Design

### 3.3.1 Distribution-Table Algorithm (BuildBackendDistribution)

Input: allBackendIds (all nodes), writeBackendIds (writer-role nodes), and memoryBackendIds (memory-role nodes). Output: std::vector<BackendDistribution>, with a weighted target table {backendId, weight}[] for each writer backendId.

**Stage one: greedy ledger.** Each backendId has a total share of perBackendShare = number of writers; each writer has a quota of quota = number of backendIds. A writer starts consuming from **its own backendId** (local first), then takes remaining shares in a circular traversal of allBackendIds:

Example with 2 writers + 1 memory node (backendId: b1, b2 = “P”; b3 = “D”):

| Writer | Allocation process | Result |
| --- | --- | --- |
| b1 | Its quota is 3. b1 has 2 remaining, so take 2; b2 has 2 remaining, so take 1. | b1 -> {b1:2, b2:1} |
| b2 | Start at b2, which has 1 remaining, so take 1; b3 has 2 remaining, so take 2. | b2 -> {b2:1, b3:2} |

**Stage two: local-affinity exchange.** Traverse each writer. If its highest-weight target is not local, find the other writer that holds the largest share on this writer's local node, then exchange equal amounts up to min(the two shares), conserving total consumption per backendId:

| Step | Action | Result |
| --- | --- | --- |
| 1 | b1's largest target is its local b1 (2); skip it. | b1 -> {b1:2, b2:1} |
| 2 | b2's largest target is b3 (2), not local. b1 holds 1 share on b2, so exchange 1. | b2 -> {b2:2, b3:1}; b1 -> {b1:2, b3:1} (remove b2, whose share is now zero) |

**Final distribution** (2P1D scenario): b1 -> {b1:67%, D:33%} and b2 -> {b2:67%, D:33%}. Each writer is **2/3 local + 1/3 evenly routed to D**. Both writers feed D equally, balancing write pressure across writer instances.

### 3.3.2 Allocation Path

Core function: ExpandLocalFirstRanks (a MmcMetaService member). It looks up the caller rank's distribution weights, randomly selects a target backendId in proportion to its weight, shuffles all ranks in that backendId with std::shuffle, and writes them to options.preferredRank_. The existing LocalFirstAssign then tries each rank in order.

```
Put/BatchPut (client fills rankId_ locally)
  |
  v
HandleAlloc / HandleBatchAlloc (MetaNetServer)
  +-- (options.flags_ & 0xFF == ALLOC_LOCAL_FIRST)
       v
ExpandLocalFirstRanks(options, callerRank)
  +-- SelectTargetBackendForRank(callerRank) // lock, look up table, weighted random choice
  +-- targetBackendId empty -> set flags_ to ALLOC_ARRANGE as fallback
  +-- GetRanksForBackendId(targetBackendId) // all ranks on the target node
  +-- std::shuffle(ranks, thread_local mt19937) // randomize within group
       +-- options.preferredRank_ = shuffledRanks
       v
InnerAlloc (MmcGlobalAllocator, ALLOC_LOCAL_FIRST)
  +-- LocalFirstAssign: traverse preferredRank_; allocate on the first available allocator
       +-- all fail -> Free(blobs) -> retry with ALLOC_RANDOM
       |    +-- increment local_first_fallback (metric)
       |    +-- WARN log (callerRank + targetBackendId)
       +-- success -> return blob
```

### 3.3.3 Concurrency and Performance

- **Distribution-table reads:** SelectTargetBackendForRank traverses writeDistributionTable_ and shares rankBackendIdMapLock_ (std::shared_mutex) with RebuildWriteDistributionTable (registration/deregistration). Readers share the lock and writers hold it exclusively. The table is rebuilt in full (vector assignment) under the write lock; readers must hold the lock or use a copy to avoid dangling references.
- **Shuffle within a group:** thread_local mt19937 avoids locks. Each allocation is O(number of ranks in the group), typically the number of cards (single digits), so the cost is negligible.
- **Distribution-table rebuild cost:** O(n²), where n is the number of backendIds, due to two traversals. It runs only during registration/deregistration, which is infrequent.
- **Batch allocation:** HandleBatchAlloc calls ExpandLocalFirstRanks independently for each key's options. Each key gets an independent weighted choice and within-group shuffle, naturally spreading allocations. Batch reuse of the distribution table can be optimized later; the table is the same for every key, and only the random choices differ.

### 3.3.4 Security, Privacy, and DFX

**Privacy:** This feature does not collect or report user data. backendId and role are cluster-topology configuration and present no privacy risk.

**Compatibility:**

- If role is an empty string, treat the instance as a writer. Older LocalService versions without a role setting can still participate in allocation; log an ERROR asking operators to fill in the setting.
- If the target backendId is empty (not configured or not a writer), fall back to ALLOC_ARRANGE / ALLOC_RANDOM without changing the previous behavior.
- If alloc_strategy is omitted, it defaults to random and behavior remains unchanged.

**Reliability:**

- Explicit caller identity fully decouples user-specified preferred ranks from node-affinity decisions.
- Clear preferredRank_ on the fallback path before using ALLOC_RANDOM, preventing the old list from affecting allocation.

**Observability:**

- **Metric:** local_first_fallback counts fallbacks to global random allocation. It appears in /metrics/summary and supports the Prometheus counter memcache_local_first_fallback_total.
- **Ptracer:** Instrument distribution-table rebuild time with TP_MMC_META_BUILD_DIST_TABLE.
- **Debug logs:** ExpandLocalFirstRanks logs caller rank, target backendId, and the expanded preferredRank_ list (RanksToString); LocalFirstAssign logs the rank selected for each blob; fallback logs caller rank and target backendId.

## 3.4 Programming and Call Design

### 3.4.1 Configuration

**No new MetaService configuration is required**; all inputs come from LocalService registration reports. **LocalService** adds two settings (which can also be overridden through the Python API):

| Setting | Type | Required | Default | Meaning |
| --- | --- | --- | --- | --- |
| ock.mmc.local_service.backend_id | string | Recommended | Empty | Node connection IP/hostname. Multiple LocalServices on the same machine should share a backendId. |
| ock.mmc.local_service.role | string | Optional | writer | writer (P, writer) / memory (D, memory-only contributor) |

The corresponding Python API fields are LocalConfig.backend_id / LocalConfig.role (default "writer"). Set LocalConfig.alloc_strategy = 8 to enable this strategy (ALLOC_LOCAL_FIRST).

### 3.4.2 Added or Changed Interfaces

| Location | Interface | Description |
| --- | --- | --- |
| common/mmc_locality_distribution | BuildBackendDistribution(all, write, memory) | Pure function that builds the writer-to-target weight table with the two-stage greedy algorithm |
| common/mmc_locality_distribution | PickTargetByWeight(dist, rng) | Idempotently selects a target backendId according to weights |
| meta_service/mmc_meta_service | RebuildWriteDistributionTable() | Rebuilds the table after registration/deregistration (ptracer instrumentation) |
| meta_service/mmc_meta_service | SelectTargetBackendForRank(callerRank) | Locks and looks up the table, then selects a weighted target; returns an empty string if unconfigured or not a writer |
| meta_service/mmc_meta_service | ExpandLocalFirstRanks(options, callerRank) | Expands node affinity by shuffling all ranks on the target backendId into the preferred list |
| proto/mmc_msg_client_meta | AllocRequest/BatchAllocRequest::rankId_ | Explicit caller-rank field, populated by the Client and sent over RPC |
| meta_service/mmc_meta_net_server | HandleAlloc/HandleBatchAlloc | Calls ExpandLocalFirstRanks in place of the original ExpandRanks |
| meta_service/mmc_global_allocator | InnerAlloc ALLOC_LOCAL_FIRST branch | If LocalFirstAssign fails, retry with ALLOC_RANDOM and increment the fallback counter |

### 3.4.3 File Change List

**Core implementation:**

- src/memcache/csrc/common/mmc_locality_distribution.h/.cpp (new): distribution-table construction and weighted selection pure functions
- src/memcache/csrc/meta_service/mmc_meta_service.h/.cpp: backendIdRoleMap_, writeDistributionTable_, RebuildWriteDistributionTable, SelectTargetBackendForRank, and ExpandLocalFirstRanks; upgrade lock to shared_mutex
- src/memcache/csrc/meta_service/mmc_meta_net_server.cpp: switch HandleAlloc/HandleBatchAlloc to the new interface and update comments
- src/memcache/csrc/meta_service/mmc_global_allocator.h: fall back from LOCAL_FIRST to RANDOM and count local_first_fallback
- src/memcache/csrc/meta_service/mmc_locality_strategy.h: retain LocalFirstAssign (the order of preferredRank_ is the expansion order)
- src/memcache/csrc/proto/mmc_msg_client_meta.h: add rankId_ to allocation requests
- src/memcache/csrc/client/mmc_client_default.cpp: set rankId_ when constructing allocation requests

**Configuration/API:**

- src/memcache/include/mmc.h, mmc_def.h: backend_id and role fields
- src/memcache/csrc/config/mmc_config_const.h, mmc_configuration.h: parse the two new settings
- src/memcache/csrc/local_service/mmc_local_service_default.cpp: report backendId and role during registration
- src/memcache/csrc/mmc.cpp: defaults and Python API overrides
- src/memcache/csrc/python_wrapper/pymmc.cpp: expose backend_id and role

**DFX:**

- src/memcache/csrc/meta_service/mmc_meta_metric_manager.*: local_first_fallback counter
- src/memcache/csrc/meta_service/mmc_rest_api_facade.cpp: include it in summary output
- src/memcache/csrc/common/mmc_ptracer.h: TP_MMC_META_BUILD_DIST_TABLE

# 4. Drawbacks and Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Writer-local shares still use geometric equal distribution by backendId. With extreme counts (large n and few writers), the share sent to memory is not perfectly linear with the number of new writers. | Routing ratio is not strictly proportional to capacity. | The table uses geometric equal distribution with quota = number of backendIds; the error is negligible when the number of backendIds is small. Capacity-weighted distribution can be added later. |
| SelectTargetBackendForRank must hold a shared lock to read the table. | One additional read/write-lock operation on the high-frequency allocation path. | An uncontended shared_mutex read lock costs on the order of nanoseconds. A lock-free snapshot or pointer copy (replaced during rebuild) can be considered later. |
| Missing role configuration (empty string). | Old configurations are treated as writers (a conservative fallback), so memory nodes may not be recognized. | Defaulting to writer preserves write capability; an ERROR log prompts operators to complete the configuration. |
| The thread_local mt19937 seed depends on std::random_device, which may initialize slowly on a very cold start. | The first allocation on the first thread may incur a few microseconds of overhead. | One-time cost, incurred only on first access by the first thread; later calls reuse the generator. |
| When the target backendId is full, fallback to global random may place data on a “failed node.” | Slightly conflicts with the intent of ALLOC_LOCAL_FIRST. | Global random fallback only preserves availability. BmUnregister removes a failed node's allocator, so allocation will not select an invalid location. |

# 5. Open Questions

1. HandleBatchAlloc currently makes a separate weighted random choice and within-group shuffle for each key. Evaluate whether batch requests should share the table and use only one random starting point to reduce repeated traversal.
2. Evaluate whether equal quota per backendId should become capacity- or weight-based to support heterogeneous backendId capacity.
3. Evaluate whether shared-lock reads in SelectTargetBackendForRank should use a lock-free snapshot pointer; the current nanosecond-scale cost is acceptable.

***

Appendix

- **References:**
  - Source: src/memcache/csrc/common/mmc_locality_distribution.*, meta_service/mmc_meta_service.*, meta_service/mmc_meta_net_server.cpp, meta_service/mmc_global_allocator.h, proto/mmc_msg_client_meta.h
  - Verification script: /home/qzx/develop_repo/verify_local_first.py
- **Glossary:**
  - backendId: Node connection IP/hostname, shared by multiple LocalServices on the same machine (node-affinity grouping dimension)
  - role: writer (writer, P) / memory (memory-only contributor, D)
  - Distribution table (BackendDistribution): writer backendId -> target backendId weight table, rebuilt on registration/deregistration
  - preferredRank_: Preferred rank list in AllocOptions; under local_first, expanded to all ranks in the target backendId (after shuffling)
  - local_first_fallback: Counter for fallback to ALLOC_RANDOM after allocation on the target backendId fails
- **Documentation update plan:** Update docs/zh/memcache_config.md with the backend_id and role settings.

