# Cluster Scaling

> MemCache can dynamically add or remove nodes within the `world_size` limit without restarting MetaService.

---

## Core Concepts

| Concept | Description |
| --- | --- |
| **world_size** | Maximum cluster size, set in the configuration file and immutable at runtime (default 256, maximum 1024). |
| **Actual node count** | Any value up to `world_size`; can change at runtime. |
| **Rank ID** | Unique identifier assigned automatically to each node in the cluster. |

```
world_size = 32, initially 4 nodes:

Online:  R0  R1  R2  R3  R4  R5  ...  R31
Status:  ●   ●   ●   ●   ○   ○        ○
         └── deployed ──┘  └── can join dynamically at any time ──┘
```

---

## Scale Out

A new node discovers and joins the cluster when it starts. Existing node configurations do not need to change.

**Flow:**

```
New node → Connect to ConfigStore → Automatically assign rank ID
         → Establish P2P connections with other nodes
         → Register local memory resources with MetaService
         → Cluster detects the node; it can now receive data
```

**Prerequisite:** The current number of online nodes must be less than `world_size`.

---

## Scale In

### Graceful Shutdown

Before a node exits, the application should drain its cached data and then call the shutdown API.

```
1. Drain data — delete entries individually or wait for natural eviction
   Use Remove(key) or BatchRemove.

2. Call Stop() — notify the cluster that the node is exiting
   Current node → BmUnregister → MetaService cleans up metadata
                → Release P2P connections → Shut down
```

**Why drain the data?** `Unmount` checks that the target node has no allocated blobs (`allocatedSize_ == 0`). If data remains, shutdown is rejected to prevent accidental data loss.

### Failure Shutdown

If a node process crashes or its network connection is lost, MetaService detects the broken TCP connection and automatically clears the node's resource records. Its data is discarded.

```
Node crash → MetaService HandleLinkBroken
           → ClearResource → EraseIf force-deletes data
           → Remove node from the global allocator
```

### Notes

- **Graceful shutdown:** Drain data to leave the cluster in a clean state.
- **Failure shutdown:** No data handling is needed. KVCache can be recomputed, and later requests will prefill entries that were lost.
- **Current limitation:** There is no one-click drain script or API. The application must call `Remove` for each key.

---

## Configuration Recommendations

| Scenario | Recommendation |
| --- | --- |
| Small or medium cluster | Set `world_size = 64`, deploy the number of nodes needed, and reserve 20%–50% headroom. |
| Large cluster | Set `world_size = 256` (maximum 1024) to allow for growth in most scenarios. |
| LLM inference | Set `world_size` to the maximum expected node count. The GVA address space is calculated automatically. |
| Capacity | Configure the same `hbm/dram.size` on every node to satisfy the underlying linear address calculation. |

---

## Limitations

| Limitation | Reason |
| --- | --- |
| `world_size` cannot be changed at runtime | The GVA address space is preallocated based on `world_size`. |
| Data cannot be recovered | Cached data on a removed node is lost. In a KVCache workload, later requests recompute it through prefill. |
| Scaling in can reduce the hit rate | The departing node's cached data is lost, which may temporarily lower the hit rate while hot data is rebuilt on other nodes. |
| Per-node capacity limit | DRAM and HBM are each limited to 1 TB. |

---

## Implementation

Cluster scaling relies on cooperation between ConfigStore and GroupEngine.

**Scale out:**

The new node marks its rank ID in the shared ConfigStore bitmap. Online nodes watch for bitmap changes, exchange memory address information with the new node, establish P2P data-transfer connections, and register with MetaService to complete resource mounting.

```
New node → Set bitmap bit → Notify online nodes
         → Exchange addresses → Establish P2P connections
         → Register with MetaService → Cluster detects the new node
```

**Scale in:**

When a node exits, it clears its rank ID from the bitmap. Online nodes remove the corresponding P2P connections, and MetaService removes the node's memory from the global allocator.

```
Node exits → Clear bitmap bit → Notify online nodes
           → Disconnect P2P → Remove from global allocator
```

**Key dependencies:**

| Component | Role |
| --- | --- |
| ConfigStore | Shares the bitmap, coordinates CAS operations, and distributes watch events. |
| GroupEngine | Listens for ConfigStore events and invokes join/leave callbacks. |
| MetaService GlobalAllocator | Mounts and unmounts node memory resources. |
