[TOC]

# Case Study: Shared MemCache Pool for Multiple Inference Instances

## 1. Scenario

Online inference deployments often run multiple instances, while many requests share the same prefixes, such as system prompts, RAG templates, and few-shot examples.

If each instance computes independently, it repeatedly pre-fills the same prefix. This wastes compute and increases TTFT.

## 2. Approach

- **Connect multiple inference instances to one MemCache pool, managed by the same MetaService and backed by the LocalService instances intended for sharing.**
- MetaService manages metadata and pool space.
- LocalService provides DRAM/HBM memory pools.
- For sharing on the same host, use `host_shm`. For cross-host sharing, use `device_rdma`, `device_sdma`, or `host_rdma`.

The key idea is to make `KVCache` a reusable resource for the entire instance pool instead of giving each instance its own private cache.

```text
                +-------------------+
Request 1 -----> Inference instance A |
Request 2 -----> Inference instance B |-----> Shared MemCache pool
Request 3 -----> Inference instance C |       (KV blocks)
                +-------------------+

Miss:  The instance runs prefill and writes the result to the pool.
Hit:   Another instance reads the shared KV and skips duplicate prefill.
```

## 3. Request Flow

1. The instance generates a KV key from the model, prefix, and version.
2. On a MemCache hit, it reads the shared KV and continues incremental computation.
3. On a miss, the instance runs prefill and writes the KV to the shared pool.
4. Later requests with the same prefix can reuse that KV from any instance.

## 4. Results

- Multiple instances share one pool instead of each holding a copy of the hot KV data.
- Shared data turns state that was private to one instance into a resource available to the whole instance pool.
- TTFT decreases because the expensive prefill work can be reused.

KV sharing is useful when the workload has stable prefix reuse. If every request has a different prefix, the benefit will be limited.
