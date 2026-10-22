# Tag Topology Routing and Affinity-Based Allocation

## 1. Overview

In multi-node, multi-device deployments, NPUs within the same HCCS-reachable domain can transfer data at high speed over HCCS. Transfers between different HCCS-reachable domains require cross-domain methods such as Device RDMA.

MemCache provides topology awareness through these settings:

- `tag`: Identifies the transport-reachable domain to which the current rank belongs.
- `tag_op_info`: Describes the data-transfer methods available between tags.
- `tag_affinity_enabled`: Controls whether random allocation prefers ranks with the same tag.
- `tag_affinity_high_watermark`: Optional tag high watermark; after it is reached, random allocation prefers other tags.

These settings apply to two stages:

| Stage | Purpose |
|---|---|
| Data placement | MetaService selects the target rank based on the write API, tag affinity, and watermark. |
| Data transfer | After the target rank is selected, MemFabric chooses a transfer method based on the source and destination tags. |

For example, configure `DEVICE_SDMA` within a tag to transfer data over HCCS, and `DEVICE_RDMA` between different tags to transfer data across HCCS-reachable domains.


## 2. Basic Concepts

### 2.1 tag

`tag` represents a domain reachable through the same transport protocol. Ranks in the same transport-reachable domain should use the same tag; different domains should use different tags.

For example:

| Reachable domain | Ranks | Tag |
|---|---|---|
| Transport protocol 0, domain 0 | rank 0, rank 1 | `TAG_0` |
| Transport protocol 1, domain 1 | rank 2, rank 3 | `TAG_1` |

Tags support letters, digits, and underscores, and must be no longer than 30 bytes. When a tag is empty, MemFabric uses an internal default tag for compatibility with existing single-transport deployments.

### 2.2 tag_op_info

`tag_op_info` describes available transfer methods between tags. Each route uses this format:

```text
tag1:transport_method:tag2
```

Separate multiple routes with commas. For example:

```text
TAG_0:DEVICE_SDMA:TAG_0,TAG_0:DEVICE_RDMA:TAG_1,TAG_1:DEVICE_RDMA:TAG_0,TAG_1:DEVICE_SDMA:TAG_1
```

The routes above mean:

| Tag pair | Transfer method | Description |
|---|---|---|
| `TAG_0` and `TAG_0` | `DEVICE_SDMA` | Transfers over HCCS within the TAG_0 domain. |
| `TAG_1` and `TAG_1` | `DEVICE_SDMA` | Transfers over HCCS within the TAG_1 domain. |
| `TAG_0` and `TAG_1` | `DEVICE_RDMA` | Cross-domain transfers between TAG_0 and TAG_1. |

`tag_op_info` is limited to 255 bytes. All ranks in the same BM should use the same complete route configuration.

The current implementation first looks up `tag1:tag2`, then tries `tag2:tag1` if no route is found. Therefore, symmetric transfers can reuse a route configured in one direction. For readability and easier operations, both directions may be listed explicitly as in the example. Do not rely on asymmetric semantics where the two directions use different transfer methods.

### 2.3 Tag Affinity

Tag affinity controls data-placement order during random allocation. When a write request does not explicitly specify a target rank through `preferredLocalServiceIDs`, the client sends the rank initiating the write to MetaService as the affinity rank.

During random allocation, candidate ranks are processed in this order:

1. Randomly shuffle the candidate ranks that satisfy the media-type requirements.
2. If tag affinity is enabled and the current rank has a nonempty tag, stably move candidates with the same tag ahead of candidates with other tags.
3. Try candidates in order. If a rank has insufficient space, continue to the next candidate.

The specific rank selected within a tag is therefore still random. Tag affinity is a preference, not a requirement; capacity, media type, exclusion lists, and rank availability still determine whether allocation succeeds.

### 2.4 Write APIs and Allocation Strategies

Regular writes and batch writes use different initial allocation strategies. This is the key distinction to understand when using tag affinity.

| Python API | Initial allocation strategy | Behavior for one replica when no target rank is specified |
|---|---|---|
| `put`, `put_from`, `put_from_layers` | Prefer this rank (`ArrangeLocality`) | Try the current rank first; return immediately on success. |
| `put_batch`, `batch_put_from`, `batch_put_from_layers` | Random allocation (`RandomAssign`) | Do not first pin the write to the current rank; choose candidate ranks in tag-affinity order. |

#### Regular `put`

The regular `put` APIs pass allocation flag 0, so MetaService uses a prefer-this-rank strategy:

1. The client sets the current rank as the default affinity rank.
2. MetaService first allocates on the target media of the current rank.
3. For a single replica, successful allocation returns immediately without entering `RandomAssign`.
4. Remaining allocations enter `RandomAssign` only if the current rank cannot allocate, or if additional replicas still need allocation.

Consequently, consecutive regular single-replica `put` calls continue to prefer the current rank as long as it has enough space. Seeing that results are on the same tag does not by itself show that tag affinity took effect, because the current rank is usually already in that tag.

#### batch put

The `put_batch` and `batch_put_from` APIs explicitly use `ALLOC_RANDOM`. Even a batch with a single key enters `RandomAssign` directly:

```python
results = store.batch_put_from(
    [key],
    [buffer_ptr],
    [size],
    direct=memcache_hybrid.L2G,
)
```

When tag affinity is enabled and the high watermark has not been reached, the object is preferentially allocated to any available rank in the current tag. It is not guaranteed to use the current rank or any particular rank in the same tag.

#### Explicitly Setting `preferredLocalServiceIDs`

Whenever `ReplicateConfig.preferredLocalServiceIDs` is nonempty, the client clears the original allocation strategy and sets `ALLOC_FORCE_BY_RANK`. This applies to both regular and batch writes:

```python
replicate_config.preferredLocalServiceIDs = [1]
```

For a single-replica request, allocation is forced to the specified rank and tag-affinity ordering is skipped. For multiple replicas, the explicitly specified ranks are assigned first. Any remaining unspecified replicas are allocated randomly, excluding ranks already assigned.

In other words, `batch_put_from` combined with `preferredLocalServiceIDs` does not mean “randomly allocate near the specified rank.”

### 2.5 Tag Affinity High Watermark

`tag_affinity_high_watermark` prevents random allocation from remaining concentrated in the current tag. Its range is `[0, 100]`:

- `0`: Default; disables proactive cross-tag allocation based on the watermark.
- `1` to `100`: Once the current tag reaches this utilization, prefer other tags that are below the watermark.

Watermark statistics and checks work as follows:

- Checked only during `RandomAssign`; the watermark is not checked when a regular single-replica `put` succeeds on the current rank.
- Statistics are collected separately for the media type requested; for example, an HBM request counts only HBM capacity and usage for each tag.
- A tag's capacity and usage are aggregated across all relevant ranks in that tag. The watermark does not need to be reached independently on every rank.
- After the current tag reaches the watermark, other tags below the watermark are preferred, followed by other nonempty tags, with the current tag placed last.

## 3. End-to-End Workflow

For example, assume rank 0 and rank 1 belong to `TAG_0`, and rank 2 belongs to `TAG_1`.

### 3.1 Service Startup and Topology Registration

1. When LocalService starts, it passes the current rank's `tag` and `tag_op_info` to MemFabric to create the BM.
2. When LocalService registers the rank with MetaService, it also reports the rank's tag.
3. MetaService stores the mapping between ranks and tags.

### 3.2 Regular Single-Replica `put`

1. Rank 0 calls `put` or `put_from` without specifying `preferredLocalServiceIDs`.
2. The client sends rank 0 to MetaService as the default affinity rank.
3. MetaService first attempts to allocate space on the target media of rank 0.
4. If rank 0 succeeds, the request ends; tag random ordering and the watermark do not affect this placement.
5. If rank 0 cannot allocate, the request enters random candidate selection. Below the watermark, ranks in `TAG_0` are preferred; at the watermark, tag order is adjusted according to the high-watermark rules.

### 3.3 Batch `put`

1. Rank 0 calls `put_batch` or `batch_put_from`.
2. The request enters random candidate selection directly, without first trying rank 0 specifically.
3. Below the watermark, rank 0 and rank 1 are preferred as same-tag candidates, and one is selected at random.
4. If rank 1 is selected, MemFabric uses `DEVICE_SDMA` according to the `TAG_0` to `TAG_0` route.
5. If the current tag reaches the watermark while `TAG_1` is below it, rank 2 is preferred. MemFabric uses `DEVICE_RDMA` according to the `TAG_0` to `TAG_1` route.

Tag topology routing and tag affinity address separate concerns:

| Capability | Concern addressed |
|---|---|
| Tag affinity and high watermark | Which rank to prefer during random allocation. |
| Tag topology routing | Which transfer method to use after the target rank is selected. |

## 4. Configuration

### 4.1 Configuration Files

Set each LocalService's tag in its existing configuration and provide the same complete route table on all services.

Ranks in domain 0:

```ini
ock.mmc.local_service.protocol = device_sdma
ock.mmc.local_service.tag = TAG_0
ock.mmc.local_service.tag_op_info = TAG_0:DEVICE_SDMA:TAG_0,TAG_0:DEVICE_RDMA:TAG_1,TAG_1:DEVICE_RDMA:TAG_0,TAG_1:DEVICE_SDMA:TAG_1
```

Ranks in domain 1:

```ini
ock.mmc.local_service.protocol = device_sdma
ock.mmc.local_service.tag = TAG_1
ock.mmc.local_service.tag_op_info = TAG_0:DEVICE_SDMA:TAG_0,TAG_0:DEVICE_RDMA:TAG_1,TAG_1:DEVICE_RDMA:TAG_0,TAG_1:DEVICE_SDMA:TAG_1
```

Control tag affinity and the high watermark on MetaService with:

```ini
ock.mmc.tag_affinity.enabled = true
ock.mmc.tag_affinity.high_watermark = 0
```

`ock.mmc.tag_affinity.enabled` defaults to `true`; `ock.mmc.tag_affinity.high_watermark` defaults to `0`.

### 4.2 Python API

MetaService configuration:

```python
from memcache_hybrid import MetaConfig

meta_config = MetaConfig()
meta_config.tag_affinity_enabled = True
meta_config.tag_affinity_high_watermark = 0
```

LocalService configuration:

```python
from memcache_hybrid import LocalConfig

tag_routes = (
    "TAG_0:DEVICE_SDMA:TAG_0,"
    "TAG_0:DEVICE_RDMA:TAG_1,"
    "TAG_1:DEVICE_RDMA:TAG_0,"
    "TAG_1:DEVICE_SDMA:TAG_1"
)

local_config = LocalConfig()
local_config.protocol = "device_sdma"
local_config.tag = "TAG_0"
local_config.tag_op_info = tag_routes
```

For different reachable domains, only change `local_config.tag`; keep `tag_op_info` consistent. Configure capacity, service addresses, HBM size, and other settings for the actual environment.

## 5. Conditions for Applying the Configuration

For tag topology routing and affinity allocation to work as expected:

- Ranks in the same transport-reachable domain use the same tag, and different domains use different tags.
- All ranks in a BM use the same `tag_op_info`.
- `tag_op_info` covers every tag pair that may actually communicate.
- New tag and route settings take effect after LocalService recreates the BM.

If all rank tags are empty, MemFabric uses its internal default tag and generates compatible same-tag routes from `protocol`, preserving the existing single-protocol behavior.

If multiple distinct, nonempty tags are configured, provide routes for all possible cross-tag transfers. Leaving `tag_op_info` empty does not guarantee that cross-tag transfers fall back to `protocol`.

## 6. Tag Affinity Activation Conditions

Tag affinity is applied only when all of the following conditions hold:

- MetaService has `tag_affinity_enabled` enabled.
- The allocation request has exactly one affinity rank. If no target is explicitly specified, the client uses the rank initiating the write.
- The current rank has registered a nonempty tag with MetaService.
- Allocation has entered `RandomAssign`.

Same-tag preference is not applied in these cases:

- `tag_affinity_enabled` is disabled.
- The current rank's tag is empty or MetaService has no corresponding tag.
- The caller explicitly specifies target ranks through `preferredLocalServiceIDs`; those replicas are forced to the specified ranks without same-tag ordering.
- The request does not have exactly one affinity rank.
- A regular single-replica `put` has already succeeded on the current rank and has not entered random allocation.

Disabling tag affinity affects rank ordering during random allocation only; it does not affect transfer routes configured by `tag_op_info`.

## 7. Compatibility

- `tag` defaults to empty.
- `tag_op_info` defaults to empty.
- `tag_affinity_enabled` defaults to enabled.
- `tag_affinity_high_watermark` defaults to `0`.
- When tags are empty, MemFabric uses its internal default tag. If all ranks use that default tag, tag grouping does not change allocation order.
- Existing single-protocol deployments without explicit tag topology continue to use same-tag compatible routes generated from `protocol`.

Existing deployments that do not use multiple tag topologies therefore require no configuration changes. When configuring multiple nonempty tags, also configure cross-tag routes.

## 8. Logs and Troubleshooting

Set the MetaService log level to `debug` to see a log like this when tag affinity succeeds during random allocation:

```text
Affinity allocate, currentRank=0, currentTag=TAG_0, selectedRank=1, selectedTag=TAG_0, sameTag=1, spillToOtherTag=0, tagUsed=4096, tagCapacity=2147483648, highWatermark=80
```

Key fields:

| Field | Meaning |
|---|---|
| `currentRank` | Affinity rank used for this random allocation. |
| `currentTag` | Tag of the affinity rank. |
| `selectedRank` | Rank that received the allocation. |
| `selectedTag` | Tag of the rank that received the allocation. |
| `sameTag` | Whether the selected rank shares the affinity rank's tag; `1` means yes, `0` means no. |
| `spillToOtherTag` | Whether the current tag reached the high watermark and cross-tag preference was enabled. |
| `tagUsed` | Aggregated usage for the current tag on the media requested, in bytes. |
| `tagCapacity` | Aggregated capacity for the current tag on the media requested, in bytes. |
| `highWatermark` | Configured tag high-watermark percentage. |

## 9. Configuration and Usage Notes

- A tag represents reachability under a common transport protocol. Define tags according to the actual hardware topology, not only by business role or node name.
- The same rank must use the same tag when mounting different media; otherwise, MetaService rejects the mount.
- Tag affinity does not replace capacity planning. If the same-tag domain lacks capacity, other tags are still tried.
- The high watermark changes candidate order in `RandomAssign` only; it does not move data that has already been allocated to another tag.
- A batch-write API uses random allocation even for a single key. “Batch” describes the API and request form, not the number of replicas.
- `replicaNum` determines the replica count and is independent of choosing regular or batch writes.
- `DEVICE_RDMA` requires the corresponding network and communication components to be available; otherwise, cross-domain transfers cannot complete normally.
- After changing `tag`, `tag_op_info`, or affinity settings, restart the relevant services as required.

For more settings, see [MemCache Configuration](./memcache_config.md) and the [Python API reference](./memcache_python_api.md).

## 10. Source-Code References

The key behaviors described here were checked against these implementations:

- Regular `put` passes allocation flag 0: `MmcacheStore::Put` and `MmcacheStore::PutFrom` in `src/memcache/csrc/mmcache_store.cpp`.
- Batch `put` passes `ALLOC_RANDOM`: `MmcacheStore::PutBatch` and `MmcacheStore::BatchPutFrom` in the same file.
- An explicit preferred rank overrides the original strategy and sets `ALLOC_FORCE_BY_RANK`: `PrepareAllocOpt` in `src/memcache/csrc/client/mmc_client_default.cpp`.
- Prefer-this-rank allocation and fallback to random allocation: `ArrangeLocality` in `src/memcache/csrc/meta_service/mmc_locality_strategy.h`.
- Random shuffling, same-tag ordering, high watermark, and affinity logs: `RandomAssign` and `ShouldSpillToOtherTag` in the same file.
- Default tag-affinity and high-watermark values: `src/memcache/csrc/config/mmc_config_const.h` and `src/memcache/csrc/mmc.cpp`.
- Tag and `tag_op_info` length limits: `src/memcache/include/mmc_def.h` and `src/memcache/csrc/config/mmc_configuration.h`.
- Tag format, route parsing, and reverse-route fallback: `memfabric_hybrid/src/hybm/csrc/entity/hybm_entity_tag_info.cpp` and `memfabric_hybrid/src/util/csrc/mf_ipv4_validator.h`.
