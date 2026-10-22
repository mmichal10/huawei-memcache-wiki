# Scheduled-Compaction-and-Forced-Eviction-Design

**Authors:** @logn/zhaoliang
**Created:** 2026-09-20
**Updated:** 2026-09-20

***

# 1. Overview

## 1.1 Introduction

`MmcBlobAllocator` manages memory with an address tree (`addressTree_`, which records free blocks by offset) and a size tree (`sizeTree_`), and uses first-fit allocation. After long-running workloads, keys on intermediate media are released in a scattered pattern, creating many small, nonadjacent holes. Total free space may be sufficient, but no single contiguous block satisfies a request, causing allocation failure.

This proposal addresses address-contiguity management for intermediate media (HBM/DRAM) with two complementary mechanisms:

- **Scheduled task:** Periodically perform asynchronous defragmentation (`DefragEvict`) to continuously merge holes during idle periods and reduce fragmentation.
- **Emergency fallback:** On allocation failure, synchronously force eviction (`AllocWithForceEvict`) to immediately release cold keys when fragmentation caused the failure and allow the current allocation to succeed.

SSD is managed by UBS IO and is outside this proposal.

## 1.2 Motivation

Current state: Watermark eviction (`CheckAndEvict` → `MultiLevelElimination`) considers only total media utilization and cannot address contiguity. When total free space is sufficient but fragmented, evicting more cold data does not necessarily create a contiguous block large enough for an allocation.

Problem: After long-running workloads, intermediate media may contain many small, nonadjacent holes. Large puts fail because no contiguous space is available, regardless of utilization watermarks. Simply lowering the eviction watermark does not address the issue and causes unnecessary data loss.

Solution: Manage contiguity independently of watermark eviction, which manages total capacity. The scheduled task proactively merges holes to reduce fragmentation; the allocation-failure path synchronously evicts cold keys as a fallback. Single-flight guards prevent conflicting concurrent work.

Fragmentation criterion (`MmcGlobalAllocator::IsFragmentation`): fragmentation is present when **total free space is at least the aligned request size** (so capacity is not exhausted), **but the largest contiguous free block is smaller than the request** (the space is fragmented). `MmcBlobAllocator::GetMaxContiguousFreeSize()` obtains the largest block from the last element of `sizeTree_`, which is sorted by size in ascending order.

## 1.3 Goals

**Goals:**

- Periodically merge holes in intermediate media to reduce fragmentation at its source without blocking business paths.
- If an allocation fails because of fragmentation, synchronously evict and retry within the same call so that the allocation succeeds.
- Keep responsibilities separate from watermark eviction: watermarks manage total usage; this proposal manages contiguity.
- Give the scheduled task and watermark eviction independent single-flight guards so neither blocks the other.
- Make release media-aware: release only the selected-media blob and preserve copies on other media (such as persistent SSD copies) and healthy ranks.
- Make the defragmentation trigger threshold configurable and hot-reloadable, independently of the high and low eviction watermarks.
- Provide counters and logs for defragmentation and eviction for observability.

**Out of scope:**

- Eviction for capacity-exhaustion scenarios remains the responsibility of watermark eviction.
- SSD is managed by UBS IO and is outside this proposal.
- Hot-data reordering and data-movement-based compaction are out of scope. The primary method is to release ranges by evicting cold keys (for HBM, an asynchronous thread pool moves data down when lower-level space is available).
- Allocation is not guaranteed to succeed if a fragmented rank cannot yield a contiguous range; in that case, return the original failure and let the caller retry.

# 2. Use-Case Analysis (Requirements)

| Use case | Priority | Functional and performance goal | Notes |
| ------------ | --- | ------------------------------- | ------------------------------------------------------------ |
| Periodic defragmentation | P0 | Merge holes during idle periods to reduce fragmentation without degrading business performance | `DefragEvict` runs asynchronously in a thread pool with its own `defragCheck_` single-flight guard |
| Allocation-failure fallback (forced eviction) | P0 | If fragmentation causes `Alloc` to fail, synchronously evict and retry within the same call | `AllocWithForceEvict` shares the `evictCheck_` single-flight guard with `CheckAndEvict` |
| Media-aware release | P0 | Release only the selected-media blob; preserve dual-write and replicated data | Retain SSD and healthy-rank copies; `ForceReleaseKey` removes only the target rank |
| Configurable, hot-reloadable threshold | P1 | Change the trigger threshold without restart; decouple it from eviction thresholds | `ock.mmc.defrag_evict_threshold`, default 70, range `[1, 99]` |
| Observability | P2 | Observe defragmentation and eviction, attributing metrics to the rank actually released | `IncrementEvictCounter` counts the rank of the matched blob |

# 3. Design

## 3.1 Overall Design

Fragmentation management is an overall solution consisting of two complementary mechanisms:

| Mechanism | Purpose | Trigger | Execution |
| ------------------------------------------- | --------- | ----------- | -------------- |
| **Scheduled task:** periodic asynchronous defragmentation (`DefragEvict`) | Proactive, addresses root cause | Periodic task (after watermark eviction) | Asynchronous; does not block business paths |
| **Emergency fallback:** synchronous forced eviction on allocation failure (`AllocWithForceEvict`) | Fallback for immediate recovery | Allocation fails and is classified as fragmentation | Synchronous; return memory immediately, then retry |

- The scheduled task continuously merges holes during idle periods, **reducing fragmentation** and preventing allocation failures at their source.
- If fragmentation still causes allocation failure after scheduled compaction, the emergency fallback **synchronously evicts immediately** so the current allocation can succeed.
- Existing watermark eviction manages total usage; this proposal manages contiguity. Their responsibilities are independent.

Interaction between the two mechanisms:

```
┌──────────── Scheduled task (proactive, addresses root cause) ────────────┐
│                                                              │
│  evict_check cycle -> CheckAndEvict (watermark eviction, total usage)    │
│                    -> DefragEvict (defragmentation, contiguity)          │
│                       Asynchronously release the longest contiguous      │
│                       cold-key ranges to merge holes and reduce         │
│                       fragmentation                                     │
└──────────────────────────┬───────────────────────────────────┘
                           │ Lower fragmentation probability
                           ▼
┌──────────── Emergency fallback (reactive, on allocation failure) ────────┐
│                                                              │
│  Alloc fails --IsFragmentation--> AllocWithForceEvict                     │
│                  (synchronously evict 5% from LRU tail, retry allocation) │
└──────────────────────────────────────────────────────────────┘
```

Key design constraints:

- **Asynchronous and nonblocking:** The scheduled task is submitted through `threadPool_->Enqueue` and returns immediately without blocking the periodic task or business paths.
- **Separate single-flight guards:** The scheduled task's `defragCheck_` is independent of watermark eviction's `evictCheck_`, so they can run concurrently. The emergency fallback shares `evictCheck_` with watermark eviction, allowing only one eviction at a time.
- **Media-aware release:** Release only the blob on the selected medium and preserve copies on other media and healthy ranks.
- **Consistent lock order:** All paths acquire `metaLock` before the `objMeta` lock (or LRU lock), matching existing paths and avoiding AB-BA deadlocks.

## 3.2 Technology Choices

**Single-flight flag for defragmentation:**

| Approach | Advantage | Disadvantage | Decision |
| ----------------- | ------------ | ------------------------------------------------------ | ------ |
| Reuse `evictCheck_` | Reuses an existing flag; no new flag needed | `CheckAndEvict` acquires `evictCheck_` first in the periodic task, so defragmentation is always skipped during that window | Rejected |
| Separate `defragCheck_` | Runs alongside watermark eviction without blocking it | Requires a new atomic flag | **Selected** |

**Cold-key selection:**

| Approach | Advantage | Disadvantage | Decision |
| --------------------------- | -------------------- | -------------------------------------------------------------------- | ------ |
| Prefer adjacent keys (`IsKeyAdjacentToFree`) | Each release is adjacent to a hole | Checking adjacency per key requires an allocator spinlock and tree lookup for each blob, adding synchronous allocation-thread overhead; adjacency on only one side does not guarantee a contiguous block large enough for a request | Rejected |
| Coldest 20% of LRU tail + greedy 5% across windows | Simple and reliable; trades the coldest data for the greatest benefit | May release cold keys that are not adjacent | **Selected** |

**Release method:**

| Scenario | Approach | Advantage | Disadvantage | Decision |
| ---- | ----------------------------------------------------------------------------------- | ---------------------- | -------------------------- | ------ |
| Scheduled task | Reuse `EvictCallBackFunction`; hold the container write lock per key to move data down, then release/move blobs asynchronously through `removeThreadPool_` | Does not block the defragmentation thread | Release is delayed within the cycle | **Selected** |
| Emergency fallback | Hold the `objMeta` lock and synchronously delete the source-layer blob with `DoRemoveBlobs` | Memory is freed immediately, allowing the `Alloc` retry to succeed | Removes the source-layer copy (relies on dual-write in Put and an existing SSD copy) | **Selected** |

**Defragmentation granularity:**

| Approach | Advantage | Disadvantage | Decision |
| ----------------------- | --------------------------------------------- | ---------------------------------------- | ------ |
| Aggregate check per medium | Simple to evaluate | With multiple ranks, always reports fragmentation and still triggers an O(n) full scan even if no rank is fragmented | Rejected |
| Defragment each allocator by `(rank, media)` | GVA address spaces are independent per rank; check free-segment count and fragmentation waste rate per rank to avoid unnecessary scans | Must visit every mounted location | **Selected** |

## 3.3 Functional and Performance Design

### 3.3.1 Mechanism 1: Scheduled Task — Periodic Asynchronous Defragmentation

#### 3.3.1.1 Call Logic and Trigger Sequence

Defragmentation is attached to the existing `evict_check` periodic task (the interval is configured by `evictCheckIntervalSeconds`, default 5s) and **runs after watermark eviction**. The two operations are chained within one cycle but run independently:

1. The periodic task first calls `CheckAndEvict(MEDIA_NONE, 0)` to perform watermark eviction (the existing mechanism for total usage).
2. It then calls `DefragEvict()` to trigger defragmentation (the new mechanism for contiguity).
3. `DefragEvict` uses a **single-flight flag, `defragCheck_`, independent of `evictCheck_`**. If they shared a flag, `CheckAndEvict` would acquire `evictCheck_` first during the periodic task, causing defragmentation to be skipped for that entire window. With independent flags, the two operations can run concurrently; the container's `metaLock` write lock arbitrates concurrent release of the same key.
4. `DefragEvict` submits an **asynchronous task** through `threadPool_->Enqueue` and returns immediately without blocking the periodic task. The task processes media from higher to lower levels using `DefragEvictForMedia` (SSD is managed by UBS IO and is excluded), then resets `defragCheck_`.

Entry points:

```cpp
void MmcMetaManager::DefragEvict();
void MmcMetaManager::DefragEvictForMedia(MediaType media);
```

#### 3.3.1.2 Call Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant T as evict_check periodic task
    participant MM as MetaManager
    participant GA as GlobalAllocator
    participant CT as MetaContainer(LRU)
    participant SC as DefragWindowScanner
    participant RTP as RemoveThreadPool

    T->>MM: CheckAndEvict(MEDIA_NONE, 0)
    Note over MM: Watermark eviction (total usage), existing mechanism
    T->>MM: DefragEvict()
    MM->>MM: defragCheck_ CAS(true), single-flight
    alt Flag already held
        MM-->>T: Skip defragmentation this cycle
    else Single-flight acquired
        MM->>RTP: Enqueue (asynchronous defragmentation task)
        RTP-->>T: Return immediately (does not block periodic task)
        Note over RTP: The following work runs asynchronously
        loop Each medium, media = HBM → … (skip SSD/NONE)
            RTP->>MM: DefragEvictForMedia(media)
            MM->>GA: GetFreeSpace(media)
            alt Total free space == 0
                Note over MM: Capacity exhaustion belongs to watermark eviction; skip
            else Free space available
                MM->>CT: GetKeyCount(media) / GetEvictCandidateKeys(media, last 20%)
                Note over CT: Under LRU read lock, take the 20% coldest tail keys (per medium, reused across ranks)
                Note over MM: releaseTarget = total LRU keys for this medium * 5%
                MM->>GA: GetMediaLocations(media)
                loop Each (rank, location)
                    MM->>GA: GetFreeSegments(loc)
                    alt freeSegments.size() <= 1
                        Note over MM: Free space on this rank is contiguous; no fragmentation
                    else Fragmentation waste rate < threshold
                        Note over MM: Unusable-free-space ratio is below threshold; no compaction needed
                    else Multiple free segments and waste rate >= threshold (fragmentation present)
                        MM->>CT: IterateIf (index-building callback)
                        CT->>CT: Build blob index under metaLock read lock (hot keys are barriers; copy cold keys under lock to avoid dangling references)
                        MM->>SC: PickDefragWindow(freeSegments, blobIdx, 5% target)
                        SC->>SC: Process / Settle (merge two inputs and split into windows)
                        SC->>SC: Select(target) (greedily reach 5% across windows)
                        SC-->>MM: Selected key indexes
                        Note over MM: Restore targets from coldKeyCopies (contiguous cold-key list)
                        MM->>CT: EvictDefragTargets(targets, moveFunc, media)
                        loop Each target key (acquire metaLock write lock per key)
                            CT->>CT: metaMap_.find(key); skip if not found (deleted concurrently)
                            CT->>MM: moveFunc = EvictCallBackFunction(key, objMeta, media)
                            Note over MM: Hold objMeta lock and move down; release only the selected-medium blob
                            MM->>RTP: PushRemoveList / DispatchMoveBlob (asynchronous release/move)
                            MM-->>CT: EvictResult（REMOVE / MOVE_DOWN）
                            alt REMOVE (no blobs remain on other media)
                                CT->>CT: Remove from LRU and erase from metaMap_ (delete whole key)
                            else MOVE_DOWN (blob remains on another medium such as SSD)
                                CT->>CT: Remove from LRU only; retain the entry (demoted residency)
                            end
                        end
                        Note over CT: Release the lock and yield once every yieldInterval keys
                    end
                end
            end
        end
        MM->>MM: defragCheck_.store(false)
    end
```

#### 3.3.1.3 Four Algorithm Steps (Design-to-Implementation Mapping)

| Step | Design | Implementation |
| -- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | Build the relationship between blobs and GVAs | In `DefragEvictForMedia`, traverse `metaMap` with `IterateIf`; record `(gva, 4K-aligned size, keyIdx, releasable)` in ascending GVA order |
| 2 | Combine blob/GVA relationships with free ranges from `addressTree` to build segments | `GetFreeSegments` allocator snapshot + `PickDefragWindow` merge into a complete range map of free space, allocated blobs, and ownership |
| 3 | Use ranges and the LRU tail's coldest 20% of keys to greedily select 5% for release across windows | `CollectDefragColdKeys` (tail 20%) + `PickDefragWindow` window selection; `Select(target)` fills 5% across windows |
| 4 | Move down and release the selected keys | Container-level `EvictDefragTargets`: hold the container write lock and call `EvictCallBackFunction` (move-down, reusing `CheckAndEvict`'s `moveFunc`), then erase or remove from LRU under lock (ABA-free), matching `EvictOneLeastRecentlyUsed`; yield after every `yieldInterval` keys |

#### 3.3.1.4 Fragmentation Precheck (Per Rank)

Different ranks have independent GVA address spaces and cannot be merged across ranks, so each allocator is defragmented by `(rank, media)`. **The fragmentation precheck must run per rank:** at most one free segment means free addresses on that rank are contiguous and there is nothing to defragment. (An aggregate check per medium would always report fragmentation with multiple ranks, causing an O(n) full scan even when no rank is fragmented.) In addition to counting segments, **use the fragmentation waste rate to decide whether defragmentation is worthwhile**: when there are multiple free segments, `IsDefragWorthy` measures the percentage of total free space occupied by blocks smaller than the usable-block threshold. Defragment a rank only when that percentage reaches `defragEvictThreshold_` (setting `ock.mmc.defrag_evict_threshold`, default 70); otherwise skip it because a large put may still fit and compaction would provide no benefit.

`DefragEvictForMedia` checks and iterates as follows:

- Return immediately if the medium is `MEDIA_NONE` or `MEDIA_SSD`.
- Return if `GetFreeSpace(media)` is 0. With no free space, there are no holes to merge; capacity exhaustion is handled by watermark eviction.
- For each location from `GetMediaLocations(media)`, get a `GetFreeSegments(loc)` snapshot. Skip the rank if it has at most one segment.
- If it has multiple segments, evaluate the fragmentation waste rate (share of unusable free space). Skip the rank if it is below `defragEvictThreshold_`; a low waste rate means a large allocation may still fit, so defragmentation would not help.
- Build a blob index under the `metaLock` read lock using `IterateIf`, greedily select contiguous cold keys with `PickDefragWindow`, then release them one at a time by moving them down through `EvictDefragTargets`.

Defragmentation eviction constants (`MmcMetaManager` anonymous namespace):

| Constant | Value | Meaning |
| -------------------------------- | --- | ---------------------- |
| `DEFRAG_EVICT_CANDIDATE_PERCENT` | 20 | Take the 20% least-recently-used keys from the LRU tail as candidates |
| `DEFRAG_EVICT_RELEASE_PERCENT` | 5 | Limit each release to 5% of the medium's total LRU keys |
| `DEFRAG_EVICT_PERCENT_DENOM` | 100 | Percentage denominator |

#### 3.3.1.5 Greedy Window-Selection Algorithm

**Problem to solve**

After a memory allocator runs for a long time, free segments and allocated blobs become interleaved in the address space, creating fragmentation. Evicting one cold key alone may not free a large contiguous range because a hot blob belonging to another key may be next to it.

The algorithm aims to **find a group of adjacent, releasable cold keys and evict them together, joining the space they occupy with neighboring free holes to create one large free block.**

**Inputs**

`PickDefragWindow` accepts two data sets sorted by address (GVA) in ascending order:

| Parameter | Meaning | Example |
| -------------- | ----------------------------------------- | ------------------------------------------ |
| `freeSegments` | **Free address ranges** in the allocator snapshot | `[0,100)`, `[500,600)`, `[750,800)` |
| `blobIdx` | **All allocated blobs** in the address space, each marked with `releasable_` | `[100,200)` cold k1, releasable; `[200,300)` hot k5, not releasable |

Key point: `blobIdx` **includes blobs belonging to hot keys**. They are not release targets; they act as “obstacles” that stop a window during scanning and ensure that the addresses within a window are truly contiguous (not interrupted by an overlooked hot blob).

For `releasable_` to be true, both conditions must hold: the key is in the coldest 20% at the LRU tail, and the blob state is `READABLE`.

**Three algorithm steps**

Step 1: Merge two sorted inputs into one sorted stream

As in merge sort, repeatedly take the next item with the lower GVA and emit a range, recording its start and end addresses, whether it is free, whether it is releasable, and its key:

```
Free:    [0,100)              [500,600)          [750,800)
Blobs:        [100,200) cold k1 [200,300) hot k5 [300,400) cold k2 [400,500) cold k3 [600,700) cold k4 [700,750) hot k6

Merged → [0,100) free → [100,200) cold k1 → [200,300) hot k5 → [300,400) cold k2 → [400,500) cold k3 → [500,600) free → [600,700) cold k4 → [700,750) hot k6 → [750,800) free
```

Step 2: Greedy window scan — accumulate “contiguous releasable ranges” along the merged stream

Traverse the merged stream and accumulate a **window** (a contiguous address range made entirely of free or releasable ranges). **Settle** (end the current window) in either of these cases:

1. **An unreleasable blob is encountered** (hot key / not `READABLE`): it is a wall, so the window ends here.
2. **An address gap is encountered** (the next range starts after the current window's end): continuity is broken, so the window ends here.

Store the key for each releasable blob in the window in `winKeys_` (a vector whose insertion order preserves ascending GVA order), and deduplicate with `winSeen_` (`unordered_set`). **If one key has multiple blobs, record it only on first occurrence (lowest GVA).** Each settled window is kept in the `windows_` list for cross-window selection in step 3.

Step 3: Greedy selection across windows — reach the 5% target

`DefragEvictForMedia` calculates the 5% release target as `releaseTarget = max(total LRU keys on this medium * 5 / 100, 1)` (the cold-key candidates and target are calculated once per medium, outside the rank loop). `Select(target)` reaches the target across windows as follows:

1. Sort windows by **descending merged-space size** (`mergedSize_`, the contiguous free space available after release; for equal sizes, prefer the window with more keys), then iterate from the start.
2. Take keys from the sorted windows in order. Keys within a window are already in ascending GVA order, guaranteed by insertion order, so take the first N directly.
3. If a window contains more keys than the remaining target, truncate it and take the keys with the lowest GVAs.
4. Stop early (`break`) when the accumulated count reaches `target`; there is no need to scan every window.

**The primary criterion is maximum merged space:** The direct goal of defragmentation is to reclaim contiguous free space. Prefer windows with more merged space and reach the 5% target using as few windows as possible to maximize the benefit of each release.

**Complete worked example**

Consider the following address space (`FREE` = free, `COLD` = cold and releasable, `HOT` = hot and unreleasable):

```
Address: 0     100    200    300    400    500    600    700    750    800
         |------|------|------|------|------|------|------|------|------|
         FREE   COLD   HOT    COLD   COLD   FREE   COLD   HOT    FREE
         (free) k1     k5     k2     k3     (free) k4     k6     (free)
         0-100  100-200 200-300 300-400 400-500 500-600 600-700 700-750 750-800
```

Merge scan:

| Step | Range | Type | Action | Window state |
| -- | ----------- | ---- | ------------ | ---------------------------- |
| 1 | `[0,100)` | Free | Open window | `[0,100)` keys={} |
| 2 | `[100,200)` | Cold k1 | Extend window | `[0,200)` keys={k1} |
| 3 | `[200,300)` | Hot k5 | **Wall! Settle window 1** | Window 1: keys={k1}, size=200; reset |
| 4 | `[300,400)` | Cold k2 | Open window | `[300,400)` keys={k2} |
| 5 | `[400,500)` | Cold k3 | Extend window | `[300,500)` keys={k2,k3} |
| 6 | `[500,600)` | Free | Extend window | `[300,600)` keys={k2,k3} |
| 7 | `[600,700)` | Cold k4 | Extend window | `[300,700)` keys={k2,k3,k4} |
| 8 | `[700,750)` | Hot k6 | **Wall! Settle window 2** | Window 2: keys={k2,k3,k4}, size=400 |
| 9 | `[750,800)` | Free | Open window | `[750,800)` keys={} |
| 10 | End of scan | — | **Settle window 3** | No keys; do not add to list |

**Final result: `{k2, k3, k4}`**

After evicting these three cold keys, the entire `[300,700)` range is free (the blobs for k2, k3, and k4 are released, along with the already-free `[500,600)` range), forming one contiguous free block of 400 bytes.

By comparison, evicting only k1 yields just 200 bytes of contiguous space (`[0,200)`) and releases only one key. `Select` sorts by merged-space size (400 > 200), chooses window 2, and takes `{k2, k3, k4}` to reach the target in one pass.

**Core Interface and Key Snippets**

The algorithm is encapsulated in the `DefragWindowScanner` struct (`meta_service/mmc_defrag_evict.h`). After collecting all windows, `Select(target)` greedily selects across them; `PickDefragWindow` only drives the merge. Core interface signatures:

```cpp
struct DefragWindowScanner {
    void Process(uint64_t start, uint64_t end, bool isFree, bool releasable, uint32_t key); // Feed one range from the merged stream
    void Settle();                                                                            // Settle the current window and add it to the list
    std::vector<uint32_t> Select(size_t target);                                             // Greedily reach target across windows
};

// Contract: freeSegments and blobIdx must both be sorted by ascending GVA (guaranteed by the caller)
inline std::vector<uint32_t> PickDefragWindow(const std::vector<SpaceRange> &freeSegments,
                                              const std::vector<DefragBlobEntry> &blobIdx, size_t releaseTarget);
```

Core decision logic in `Process` (key snippet): settle when encountering a “wall” or address gap; otherwise extend the window. Record and deduplicate only releasable blob keys:

```cpp
void Process(uint64_t start, uint64_t end, bool isFree, bool releasable, uint32_t key) {
    if (!isFree && !releasable) { Settle(); return; }        // Wall: unreleasable blob; settle
    if (hasWin_ && start > wEnd_) { Settle(); }               // Address gap; settle
    if (!hasWin_) { hasWin_ = true; wStart_ = start; }
    wEnd_ = std::max(wEnd_, end);                              // Extend window
    if (!isFree && winSeen_.insert(key).second) {
        winKeys_.push_back(key);                              // Insertion order is ascending GVA; record a key only on first occurrence
    }
}
```

`DefragEvictForMedia` calculates the cold-key candidates and release target outside the rank loop (reusing the container's `GetKeyCount` / `GetEvictCandidateKeys` primitives), then builds a blob index under the `metaLock` read lock with `IterateIf`:

```cpp
const size_t totalKeys = metaContainer_->GetKeyCount(media);
const size_t coldCount =
    std::max(totalKeys * DEFRAG_EVICT_CANDIDATE_PERCENT / DEFRAG_EVICT_PERCENT_DENOM, static_cast<size_t>(1));
const auto coldKeyVec = metaContainer_->GetEvictCandidateKeys(media, coldCount);
const std::unordered_set<std::string> coldKeys(coldKeyVec.begin(), coldKeyVec.end());
const size_t releaseTarget =
    std::max(totalKeys * DEFRAG_EVICT_RELEASE_PERCENT / DEFRAG_EVICT_PERCENT_DENOM, static_cast<size_t>(1));
// In the IterateIf callback: collect (GVA, aligned size, keyIdx, releasable) in blobIdx; copy cold keys into coldKeyCopies to prevent dangling references
const auto bestKeyIdx = PickDefragWindow(freeSegments, blobIdx, releaseTarget);
```

**Key Design Points**

1. **`blobIdx` includes hot keys:** This is not to release them, but to make them “obstacles” that stop windows during scanning and ensure ranges within a window are truly contiguous.
2. **5% release target filled across windows:** The target for each release is 5% of the total (`DEFRAG_EVICT_RELEASE_PERCENT`), rather than selecting only the best window. Collect all windows, sort by descending merged space, and take keys until reaching 5%. If one window is insufficient, continue to another; if it has too many keys, truncate at the lowest GVA.
3. **Deduplicate with vector + unordered_set:** `winKeys_` (vector) preserves insertion order, which is ascending GVA, and `winSeen_` (unordered_set) deduplicates in O(1). For multiple blobs belonging to the same key, record only the first occurrence (lowest GVA). **`Select` does not need to sort keys**; taking the first N gives the lowest GVAs.
4. **Meaning of `wEnd_ - wStart_`:** A window contains only free ranges and releasable blobs, and is contiguous (a gap splits it), so this value equals the contiguous free block size available after release.
5. **Sort windows by merged space and stop early:** `Select` fully sorts `windows_` with `std::sort` by descending `mergedSize_`, preferring more keys when sizes tie. It takes keys until reaching `target`, then breaks. The window count w is usually around 1–10, so sorting cost is negligible; the merge scan O(n+m) dominates.
6. **Complexity:** O(n+m) merge scan + O(w log w) window sort + O(k) key selection, where w is the number of windows (usually 1–10) and k ≤ 5% of total keys. Overall complexity is **O(n+m)**.
7. **Inputs must be sorted:** The algorithm expects `freeSegments` and `blobIdx` in ascending GVA order. `GetFreeSegments` guarantees an ordered snapshot; the caller sorts `blobIdx` with `std::sort` after collecting it in the `IterateIf` callback (`metaMap_` is an `unordered_map` with arbitrary iteration order).

#### 3.3.1.6 Asynchronous Release (Algorithm Step 4)

Release is **media-aware**: only blobs on the selected medium (HBM/DRAM) are released, matching the filter semantics of the emergency fallback `ForceReleaseKey`. Blobs for the key on other media (such as a persistent SSD copy that coexists with a DRAM blob after a Get warms it up) are preserved to avoid data loss beyond what defragmentation requires.

`DefragEvictForMedia` constructs a `moveFunc` analogous to `CheckAndEvict` (forwarding to `EvictCallBackFunction`) and passes it to the container-level `EvictDefragTargets` for per-key execution. This is analogous to `EvictOneLeastRecentlyUsed`: arbitration occurs entirely under the `metaLock` write lock, with no ABA window.

1. Hold the container write lock (`metaLock` → `lruLock`) and call `metaMap_.find(key)`. If a concurrent path such as `Remove` has already deleted the key, skip it.
2. Under the lock, call `moveFunc` (`EvictCallBackFunction`, which holds the `objMeta` lock while moving data down). `IncrementEvictCounter` counts the rank of the blob matched by `filter=(UINT32_MAX, media, READABLE)`, fixing the attribution issue where `blobs[0]` from the full list might refer to an SSD blob. Then release only the selected-medium blob using one of these branches:
   - If there is no `READABLE` blob (only `ALLOCATED`, because the write did not finish or its result was lost), enqueue `PushRemoveList` with `filter=(UINT32_MAX, media, NONE)` to release blobs in all states on this medium.
   - If the lower medium does not have enough free space (DRAM's lower medium is SSD, managed by UBS IO, and `GetFreeSpace(MEDIA_SSD)` is always 0, so DRAM defragmentation always takes this path), enqueue `PushRemoveList` with `filter=(UINT32_MAX, media, READABLE)` to release only `READABLE` blobs on the selected medium. `triggerSsdPreFree=false` prevents remote SSD data deletion. If a lease is active, `removeThreadPool_` waits for the lease to expire before releasing, without blocking the defragmentation thread.
   - Otherwise, `DispatchMoveBlob` asynchronously moves the blob to the lower medium and releases the source blob after the move succeeds. This branch is reached only for HBM defragmentation when DRAM has sufficient free space.
   - Return `MOVE_DOWN` if the key still has blobs on other media (such as a persistent SSD copy); otherwise return `REMOVE`.
3. The container atomically finalizes according to the result **under the same write lock**: `REMOVE` removes the key from LRU and erases it from `metaMap_` (deleting the whole key, analogous to `Remove`); `MOVE_DOWN` removes it from LRU only (`mediaType_ = MEDIA_NONE`) and retains the container entry, equivalent to demoted residency after `MOVE_DOWN`. For `DispatchMoveBlob`, after the move completes, `MoveBlob` reinserts it into the lower-medium LRU with `InsertLru(key, dst)`.
4. After every `yieldInterval` keys, release the lock and sleep for `yieldDurationUs` microseconds so blocked Get/Put/Exist operations have an opportunity to acquire `metaLock_`.

Entry points:

```cpp
size_t MmcMetaContainer::EvictDefragTargets(const std::vector<Key> &keys,
    std::function<EvictResult(const Key &, const Value &, MediaType)> moveFunc,
    MediaType mediaType, uint32_t yieldInterval, uint32_t yieldDurationUs);
```

#### 3.3.1.7 Concurrency Safety

| Scenario | Guarantee |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Concurrent release of the same key by watermark eviction | `EvictDefragTargets` and `EvictOneLeastRecentlyUsed` arbitrate under the `metaLock` write lock. The first path completes `REMOVE` (deletes the key, so the later `metaMap_.find` misses and skips) or `MOVE_DOWN` (removes it from LRU, so watermark eviction cannot find it there). The `objMeta`-lock filter prevents double release of blobs; the later path gets an empty detach. |
| Mutual blocking with `CheckAndEvict` | `defragCheck_` and `evictCheck_` use independent single-flight guards and do not block each other. |
| Key disappears while the index is built | Build under the `metaLock_` read lock; copy cold keys into `coldKeyCopies` (`unordered_map<idx, string>`) while locked. After `IterateIf` returns and unlocks, no map pointer is dereferenced, preventing dangling references. |
| `freeSegments` and `blobIdx` are snapshots from different times | All race outcomes are conservative (a phantom hole causes an early split; a newly allocated blob is at the LRU head and therefore unreleasable, blocking the range). This can miss a candidate but cannot select the wrong one. Release uses locked live state, and allocator Release merges the truly adjacent free blocks at release time. The next cycle can recover. |
| Lock order | Every path follows `metaLock` → `objMeta` lock (or LRU lock), matching existing paths and avoiding AB-BA deadlocks. |

### 3.3.2 Mechanism 2: Emergency Fallback — Synchronous Forced Eviction on Allocation Failure

#### 3.3.2.1 Call Logic and Trigger Sequence

The emergency fallback runs on the business `Alloc` failure path and executes **synchronously** so the current allocation can succeed:

1. After `MmcMetaManager::Alloc` fails to call `globalAllocator_->Alloc`, first call `Free(blobs)` to clean up partial allocations.
2. Get the medium from `allocOpt.mediaType_` (use the top-level medium when it is `MEDIA_NONE`) and check `media != MEDIA_NONE && media != MEDIA_SSD && IsFragmentation(media, blobSize_)`.
3. If `IsFragmentation` returns true (capacity is not exhausted, but free space is fragmented), call `GetFragmentedRanks(media, blobSize_)` to check each rank and return only ranks with sufficient total free space but an insufficient largest contiguous block. Allocation through `ArrangeLocality` / `RandomAssign` may fail on any rank, so `preferredRank_` does not identify the rank that actually failed; check every mounted rank. Ranks have independent GVA address spaces, so freeing memory on another rank cannot create a contiguous range on the failed rank. Forced eviction therefore targets only fragmented ranks.
4. If `fragmentedRanks` is nonempty, enter `AllocWithForceEvict`; otherwise do not force eviction because no rank can gain contiguous memory through eviction.
5. `AllocWithForceEvict` **shares the `evictCheck_` single-flight guard** with `CheckAndEvict`, so only one eviction runs at a time. If regular eviction is already running, return the original failure and let the caller retry. Use the RAII `EvictFlagGuard` to reset the flag on every exit path, including exceptions.
6. Enter a loop (initial attempt plus at most `FORCE_EVICT_MAX_RETRY_COUNT` retries): synchronously release a batch of cold keys with `ForceEvictForFragmentation` (it returns the number released; stop if this is zero), then retry `globalAllocator_->Alloc`. Exit on success or when the retry limit is reached, avoiding pointless data loss from emptying the entire media cache over repeated iterations.

Entry points:

```cpp
Result MmcMetaManager::AllocWithForceEvict(const AllocOptions &allocOpt, MediaType media,
                                            const std::vector<uint32_t> &fragmentedRanks,
                                            Result originRet, std::vector<MmcMemBlobPtr> &blobs);
size_t MmcMetaManager::ForceEvictForFragmentation(MediaType media,
                                                   const std::vector<uint32_t> &fragmentedRanks,
                                                   uint64_t wantAllocSize);
bool MmcMetaManager::ForceReleaseKey(const std::string &key, MediaType media,
                                     const std::vector<uint32_t> &targetRanks);
```

#### 3.3.2.2 Call Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant C as Business thread (Alloc)
    participant MM as MetaManager
    participant GA as GlobalAllocator
    participant CT as MetaContainer(LRU)

    C->>MM: Alloc(key, allocOpt)
    MM->>GA: Alloc(allocOpt, blobs)
    GA-->>MM: Failure
    MM->>GA: Free(blobs) (clean up partial allocations)
    MM->>GA: IsFragmentation(media, size)?
    alt Not fragmented (capacity exhausted)
        GA-->>MM: false
        MM-->>C: Return original failure
    else Fragmentation detected
        GA-->>MM: true
        MM->>GA: GetFragmentedRanks(media, size)
        GA-->>MM: fragmentedRanks (nonempty, proceed to eviction)
        MM->>MM: AllocWithForceEvict(allocOpt, media, fragmentedRanks, ret)
        MM->>MM: evictCheck_ CAS(true), shared single-flight with CheckAndEvict
        alt Flag held (regular eviction in progress)
            MM-->>C: Return original failure; caller retries
        else Single-flight acquired
            loop Retry retry = 0..MAX
                MM->>MM: ForceEvictForFragmentation(media, fragmentedRanks, size)
                MM->>CT: GetKeyCount(media)
                MM->>CT: GetEvictCandidateKeys(media, 2x 5% candidates)
                loop Candidate keys (until 5% released)
                    MM->>MM: ForceReleaseKey(key, media, fragmentedRanks)
                    MM->>CT: Get(key, objMeta)
                    MM->>MM: Hold objMeta lock; GetBlobsDesc and select the first READABLE blob on a target rank
                    alt No matching blob / blob has an active lease
                        Note over MM: Skip; leave LRU unchanged and continue farther down to meet target
                    else Releasable blob found
                        MM->>MM: Synchronously delete the blob on that rank with DoRemoveBlobs (rank filter)
                        alt No blobs remain
                            MM->>CT: Erase(key) (delete entire key)
                        else Blobs remain on another rank of this medium
                            Note over MM: Leave LRU unchanged; key remains eligible for eviction in place
                        else Only blobs on another medium remain (e.g. SSD)
                            MM->>CT: InsertLru(key, MEDIA_NONE) (remove from this medium's LRU)
                        end
                    end
                end
                MM-->>MM: releasedCount (stop retrying if zero)
                MM->>GA: Alloc(allocOpt, blobs) (retry)
                alt Success
                    GA-->>MM: OK -> exit retry loop
                end
            end
            MM-->>C: Final result
        end
    end
```

#### 3.3.2.3 Release Strategy

Force-release 5% from the LRU tail (`FORCE_EVICT_RELEASE_PERCENT`) and fetch twice that number of candidates (`FORCE_EVICT_FETCH_MULTIPLIER`). Skip keys that cannot be released, such as those with active leases, and continue farther down the list to meet the target. Stop after all candidates have been checked:

- `releaseTarget = max(totalKeys * 5 / 100, 1)`；
- Set `fetchCount = min(releaseTarget * 2, totalKeys)` and call `GetEvictCandidateKeys(media, fetchCount)` to fetch candidates.
- Iterate over candidate keys and call `ForceReleaseKey(key, media, fragmentedRanks)` for each. The key-level function releases the first `READABLE` blob on a target rank. On success, increment `releasedCount` and continue to the next candidate; stop when `releaseTarget` is reached. Release blobs only on fragmented ranks to avoid freeing space on the wrong rank or losing data unnecessarily.

#### 3.3.2.4 Synchronously Release One Key

`ForceReleaseKey` executes synchronously, unlike the scheduled task's asynchronous release, and receives a list of target ranks (`targetRanks`, i.e. `fragmentedRanks`). Its overall structure uses a **single lock section**, analogous to `EvictCallBackFunction` but synchronous:

- **Rank filtering (select under lock):** Hold the `objMeta` lock and enumerate `READABLE` blobs on the medium with `(UINT32_MAX, media, READABLE)`. Select the **first blob on a rank in `targetRanks`** to determine which rank to release. If none matches, return false and skip the key without changing LRU. All subsequent operations are filtered to this rank:
  - Check active leases with `(rank, media, READABLE)`. Leases are blob-level, so skip only if the blob to be released has a lease; a lease on a replica on another rank does not block this release.
  - Delete with `DoRemoveBlobs((rank, media, READABLE))`. Return memory only on this rank, preserving blobs for the key on other ranks of the same medium (such as replicas on healthy ranks) and other media (such as a persistent SSD copy).
  - Attribute the eviction metric `IncrementEvictCounter` to the rank actually released.
- **Skip keys with active leases:** Return false without changing LRU, leaving the key in its current position where it can be selected again later. This matches the FAIL semantics of `EvictOneLeastRecentlyUsed`.
- **Dual-write protects data; no move-down is needed:** Put's dual-write already ensures there is a copy on the target medium (SSD), so synchronously deleting the source-layer blob does not lose data. This avoids the synchronous RPC copy overhead and space-check branches of `MoveBlob`, reducing the operation to direct deletion under one lock section.
- **Synchronous deletion:** Under the lock, call `DoRemoveBlobs` to synchronously return memory to the allocator (**the memory becomes available immediately**, allowing the `Alloc` retry to succeed) and trigger the removed callback.
- **Three finalization branches** (called after releasing the `objMeta` lock, preserving the `metaLock` → `objMeta` lock order and matching `REMOVE`/`MOVE_DOWN` in `EvictOneLeastRecentlyUsed`):
  - No blobs remain -> container `Erase(key)` deletes the entire key (the usual single-replica path).
  - Blobs remain on another rank of this medium -> **leave LRU unchanged** and keep the key in its current position so it remains eligible for eviction (same as `EvictOneLeastRecentlyUsed` leaving LRU unchanged on FAIL).
  - Only blobs on another medium, such as SSD, remain -> `InsertLru(key, MEDIA_NONE)` removes the key from this medium's LRU and demotes its residency (same semantics as defragmentation `MOVE_DOWN`).

### 3.3.3 Performance and Memory Overhead

#### 3.3.3.1 `windows_` Memory Overhead (Extreme 1 TB Scenario)

`DefragWindowScanner::windows_` is a `vector<Window>`, and each `Window` contains a `vector<uint32_t> keys_` and a `uint64_t mergedSize_`. The number of windows w equals the number of contiguous releasable ranges separated by unreleasable blobs or address gaps.

**Worst case:** 1 TB of memory, all blobs allocated at 4 MB or 3 MB, with cold and hot blobs strictly alternating (cold-hot-cold-hot…):

| Dimension | Estimate |
| -------------- | -------------------------------------------------------- |
| Total blobs n | 1 TB / 3 MB ≈ 333k |
| Number of windows w | With alternating hot and cold blobs, each hot blob splits a window -> w ≈ n/2 ≈ 166k |
| Per-window overhead | One `uint32_t` key + `uint64_t` size + vector overhead ≈ 40 bytes |
| Total `windows_` memory | 166k × 40B ≈ **6.6 MB** |

**Real workloads are far below this upper bound:** Fragmentation develops gradually, and hot and cold blobs do not strictly alternate, so w is usually around 1–10. Even under extreme alternation, 6.6 MB is only **0.0006%** of a 1 TB capacity and is acceptable.

#### 3.3.3.2 Algorithm Complexity Summary

| Stage | Complexity | Description |
| ------ | ------------ | ------------------------------------------------------- |
| Merge scan | O(n + m) | n = number of blobs, m = number of free segments; merge both inputs in one pass |
| Window sort | O(w log w) | `std::sort` all windows by descending merged space (`mergedSize_`); w is usually 1–10, so the cost is negligible |
| Key selection | O(k) | Take the first N keys in ascending GVA order without sorting; stop early |
| **Total** | **O(n + m)** | The sort is dominated by n+m; both w and k are much smaller than n |

#### 3.3.3.3 Key Design Decisions

| Decision | Approach | Benefit |
| ------------ | ------------------------------------------ | ---------------------------------- |
| Key selection | Keep insertion order in `vector` (ascending GVA) and take the first N | Avoid sorting candidate keys, reducing O(k log k) to **O(k)** |
| Window collection | Collect all windows and greedily reach 5% across windows | Meets the “select 5% from 20%” requirement; automatically continues across windows if one is insufficient |
| Multiple blobs per key | Deduplicate with `vector` + `unordered_set`; record only the first (lowest GVA) | `Select` needs no further sorting; the first N are the lowest GVAs |
| Single-flight isolation | `defragCheck_` is independent of `evictCheck_` | Watermark eviction in the periodic task does not block defragmentation; both can run concurrently |

## 3.4 Security, Privacy, and DFX Design

**Compatibility:**

- Existing public C/Python API semantics for `put`/`get`/`remove` are unchanged. The new capability consists of internal scheduling and container methods, with no breaking API change.
- Defragmentation is added only to the existing `evict_check` periodic task. If its setting is absent, it uses the default threshold 70 and does not affect existing configuration or behavior.
- The emergency fallback runs only when `Alloc` fails and the failure is classified as fragmentation; non-fragmentation failures such as capacity exhaustion still return the original error.

**Reliability:**

- Single-flight guards (`defragCheck_` / `evictCheck_`) prevent concurrent eviction conflicts between the scheduled task, emergency fallback, and watermark eviction. RAII `EvictFlagGuard` resets the flag even on exception paths.
- All paths preserve the `metaLock` → `objMeta` lock order and avoid AB-BA deadlocks.
- Media-aware release frees blobs only on the selected medium and target rank, preserving persistent SSD copies and healthy-rank copies. `ForceReleaseKey` skips a key without changing LRU if there is no match or an active lease.
- Snapshot-race outcomes are conservative (they can miss a candidate but cannot select the wrong one), and the next cycle recovers. Release safety does not depend on the snapshot.

**Observability:**

- `IncrementEvictCounter` counts the rank of the matched blob, accurately attributing the metric to the rank actually released and avoiding errors when `blobs[0]` from the full list refers to another medium such as SSD.
- Defragmentation and eviction paths have counters and logs to observe compaction benefit and forced-eviction triggers.

**Testability:**

- The repository pipeline runs unit tests and tracks coverage (line coverage ≥ 70%, branch coverage ≥ 40%).
- Defragmentation eviction strategy unit tests are in `defrag_evict_test.cpp` and registered as `test_defrag_evict`. Metadata-manager tests are in `test_meta_manager.cpp` and cover these scenarios:
  - `DefragEvictReleasesConsecutiveColdKeys`: Allocate 60 32K blobs in order and release every third blob (20 blobs: k1/k4/k7/.../k58), creating nonadjacent 32K holes. With 40 keys remaining, the LRU tail's 20% contains 8 cold keys (k0/k2/k3/k5/k6/k8/k9/k11), whose addresses are joined into one window by the holes. The 5% release cap is `max(40 * 5 / 100, 1) = 2`; truncate in ascending GVA order and release the lowest-address keys k0 and k2. Assert that k0/k2 are fully deleted (for DRAM-only keys, `EvictCallBackFunction` returns REMOVE, removes them from LRU and `metaMap_` under lock, then asynchronously releases the DRAM blobs), while k3/k5 truncated from the 5% window are unaffected. The test uses the `DefragIdle` bridge to wait for the asynchronous main task, then polls until release takes effect.
  - `DefragEvictKeepsSsdBlobOfDualMediaKey`: Use the same fragmented layout and add a persistent SSD copy to k0 to simulate the steady state after Get warms a key, where DRAM and SSD coexist. Assert that after defragmentation k0's container entry remains with only its SSD blob (the medium filter releases DRAM without triggering remote deletion of the SSD copy), k0 is removed from the DRAM LRU (demoted to SSD residency), and the DRAM-only k2 is fully deleted as usual.
  - `DefragEvictSkipsWhenMaxContiguousFreeSufficient`: With 1500 MB DRAM, allocate only 60 32K blobs and create 32K holes (multiple free segments), while the tail free segment remains about 1500 MB. The unusable-free-space ratio is well below `defragEvictThreshold_`, so the rank is skipped. Assert that the cold keys in the window that would otherwise be selected (k0/k2/k3) remain and the LRU count is unchanged.
  - `AllocFragmentationFailure_TriggerForceEvict`: Create a 32K hole among ten 32K keys, then fail to allocate 96K. This triggers synchronous forced eviction of LRU-tail keys and a successful retry within the same `Alloc` call; non-tail keys remain.
  - `ForceEvict_MultiRankOnlyReleasesFragmentedRank`: In a multi-rank setup (rank0 fragmented, rank1 healthy), the first LRU-tail candidate has a blob only on rank1. `ForceReleaseKey` skips it through rank filtering under lock (returns false and continues), then releases the rank0 key in second position; rank1 data remains intact.
  - `ForceEvict_MultiReplicaReleasesOnlyTargetRankBlob`: A key has replicas on rank0 (fragmented) and rank1 (healthy). Forced eviction releases only the rank0 blob; the key entry remains and is reinserted into the DRAM LRU for future eviction, while the rank1 blob is preserved.
  - `test_global_allocator.cpp` and `test_meta_container_lru.cpp`: allocator and container unit tests, including `GetMaxContiguousFreeSize`, `GetFreeSegments`, and `GetEvictCandidateKeys`.
- Run with: `bash script/run_ut.sh meta_manager`.

**Why 60 keys?** The 5% target is `max(n * 5 / 100, 1)`. At n=10, the target is only 1, so truncation can release just one key and cannot verify that selection truncates multiple keys in a window to the lowest GVA. With n=40 remaining keys, the target is 2; the test can assert that k0/k2 (lowest GVAs) are released while k3/k5 are retained by truncation, verifying cross-window selection and ascending-GVA truncation.

## 3.5 Programming and Invocation Design

### 3.5.1 Basic Programming Model Design

**Development environment:** Same as the MemCache main project (C++17, CMake >= 3.12, Ninja/Make).

**Entry-point signatures:**

```cpp
void MmcMetaManager::DefragEvict();
void MmcMetaManager::DefragEvictForMedia(MediaType media);

Result MmcMetaManager::AllocWithForceEvict(const AllocOptions &allocOpt, MediaType media,
                                            const std::vector<uint32_t> &fragmentedRanks,
                                            Result originRet, std::vector<MmcMemBlobPtr> &blobs);
size_t MmcMetaManager::ForceEvictForFragmentation(MediaType media,
                                                   const std::vector<uint32_t> &fragmentedRanks,
                                                   uint64_t wantAllocSize);
bool MmcMetaManager::ForceReleaseKey(const std::string &key, MediaType media,
                                     const std::vector<uint32_t> &targetRanks);

size_t MmcMetaContainer::EvictDefragTargets(const std::vector<Key> &keys,
    std::function<EvictResult(const Key &, const Value &, MediaType)> moveFunc,
    MediaType mediaType, uint32_t yieldInterval, uint32_t yieldDurationUs);
```

**New Interface Summary:**

| Location | Interface | Description |
| ---------------------- | ----------------------------------------------------- | ------------------------------------------------------- |
| `mmc_blob_allocator` | `GetFreeSegments()` | Snapshot of free `addressTree` segments (absolute GVAs, ascending order) |
| `mmc_global_allocator` | `GetFreeSegments(loc)` | Snapshot of free segments in the allocator for the specified location |
| `mmc_global_allocator` | `GetMediaLocations(type)` | Enumerate mounted locations for the specified medium (ascending rank order) |
| `mmc_meta_container` | `GetEvictCandidateKeys(media, count)` | Get `count` keys from the LRU tail (least recently used); reused by defragmentation to get the coldest 20% |
| `mmc_meta_container` | `GetKeyCount(media)` | Number of keys in the medium's LRU; reused to calculate the 5% release target |
| `mmc_meta_container` | `IterateIf(matchFunc, matchedValues)` | Traverse under `metaLock` read lock; reused to build the blob index and copy cold keys under lock to prevent dangling references |
| `mmc_meta_manager` | `DefragEvict()` / `DefragEvictForMedia(media)` | Scheduling entry point / defragmentation per medium |
| `mmc_meta_container` | `EvictDefragTargets(keys, moveFunc, media, yield...)` | Release keys by moving down under the write lock; analogous to `EvictOneLeastRecentlyUsed` |

**Acceptance Criteria:**

- Functional: Create a fragmented layout and enable the periodic task; verify that cold keys are fully evicted and holes merge. On allocation failure, retry successfully within the same `Alloc` call.
- Performance: Asynchronous defragmentation does not block business operations. `AllocWithForceEvict` triggers only on allocation failures caused by fragmentation and is bounded by the maximum retry count.
- Observability: Eviction counts are attributed to the rank actually released, and hot reload of the configured threshold takes effect.

### 3.5.2 Configuration

The defragmentation trigger threshold (share of unusable free space) is configurable in MetaService and defaults to 70. It is independent of the high and low eviction watermarks (`ock.mmc.evict_threshold_high` / `ock.mmc.evict_threshold_low`), with separate maintenance and hot reload.

| Setting | Type | Required | Default | Range | Meaning |
| -------------------------------- | --- | -------- | --- | -------------- | ------------------------------------------------------------------------------------------------------------------ |
| `ock.mmc.defrag_evict_threshold` | int | optional | 70 | `1 <= n <= 99` | Defragmentation trigger threshold: percentage of total free space occupied by unusable free blocks (small free blocks under 3 MB). Defragmentation starts when this value is reached. This is a MetaService-private setting and supports dynamic hot reload (effective when `ock.mmc.dynamic_config.enable=true`). |

- **Definition of unusable free space:** A free block smaller than `DEFRAG_EVICT_MIN_USABLE_BLOCK_SIZE` (3 MB) cannot satisfy the allocation needs of most cache blobs and is therefore treated as “wasted free space.”
- **Fragmentation criterion:** The higher the share of unusable free space, the more severe the fragmentation. When the share reaches the threshold, enough space has been wasted through fragmentation to warrant defragmentation and merging into larger free blocks.
- **Tuning:** A lower threshold triggers defragmentation more often, reducing fragmentation but increasing release overhead. A higher threshold triggers it less often, lowering overhead but slightly increasing allocation-failure probability.
- **Zero-value fallback:** If the configuration struct is zero-initialized, the field is 0 (outside the valid range). `MmcMetaService::Start` replaces it with the default 70 to avoid startup failure.
- `minUsableBlockSize` (usable-block threshold) is adaptive: use `GetMaxAlignedSize()` from the global allocator when it is nonzero; otherwise use `DEFRAG_EVICT_MIN_USABLE_BLOCK_SIZE` (3 MB).

Other hard-coded constants (not configurable yet):

| Constant | Value | Meaning |
| -------------------------------------- | --- | --------------------- |
| `DEFRAG_EVICT_MIN_USABLE_BLOCK_SIZE` | 3 MB | Free blocks smaller than this are considered unusable free space |
| `DEFRAG_EVICT_WASTE_PERCENT_THRESHOLD` | 70 | Default algorithm threshold, used as fallback when not configured |
| `DEFRAG_EVICT_CANDIDATE_PERCENT` | 20 | Take candidate cold keys from the LRU tail's 20% |
| `DEFRAG_EVICT_RELEASE_PERCENT` | 5 | Release at most 5% of total keys in one defragmentation pass |

### 3.5.3 Programming Guide

Add the `defrag_evict_threshold` setting to the existing `docs/zh/memcache_config.md`; refer to this design document for other details. This feature adds no public C/C++/Python API, so `memcache_c_api.md`, `memcache_c++_api.md`, and `memcache_python_api.md` do not need changes.

# 4. Drawbacks and Risks

| Risk | Impact | Mitigation |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| Snapshot race: `freeSegments` and `blobIdx` are captured at different times | Memory freed in the gap appears as a phantom “address gap,” settling a window early and missing a candidate in this pass | Race outcomes are conservative (may miss a candidate but cannot select the wrong one); release safety does not depend on the snapshot; the next cycle can recover |
| Index-build cost: `BuildDefragBlobIndex` scans all entries in O(n) and takes the `objMeta` lock per key | A very large number of keys can occupy the defragmentation thread for a long time | Early return when the medium has no free space and per-rank free-segment precheck avoid unnecessary scans when there is no fragmentation; evaluate batched/incremental scans later |
| Thread-pool shutdown edge case: `DefragEvict` enqueues a task but Stop occurs before it runs | `defragCheck_` remains true, so later defragmentation is skipped | Occurs only during Stop; reset the flag in the Stop path if this must be fully prevented |
| Small residual window around `Erase` outside the `ForceReleaseKey` lock | If another path concurrently deletes and recreates the same key after `DoRemoveBlobs` finishes under the lock but before `Erase(key)` outside it, an unconditional erase may delete the new entry (ABA) | The window is nanoseconds and requires precise delete/recreate interleaving, so it is currently accepted. The `evictCheck_` single-flight guard prevents concurrency with `EvictOneLeastRecentlyUsed`, further reducing the window. |
| Retry limit: `AllocWithForceEvict` is bounded by `FORCE_EVICT_MAX_RETRY_COUNT` | Allocation may still fail under extreme fragmentation | Avoid pointless data loss from clearing the whole medium on each iteration; return the original error and let the caller retry |
| Metric attribution (fixed) | Previously, `blobs[0]` from the full list could refer to another medium, such as SSD, skewing attribution | `IncrementEvictCounter` in `EvictCallBackFunction` now counts the rank of the blob matched by `srcFilter` |
| Race in media-aware release (fixed) | Earlier implementations had an ABA window around erase/LRU removal | `EvictDefragTargets` matches `EvictOneLeastRecentlyUsed` and completes atomically under the `metaLock` write lock; yield after every `yieldInterval` keys |

# 5. Related Work

| Project/community | Design | What this proposal borrows | Difference |
| ----------------------- | --------------------------------------- | -------------------------- | ------------------------------------------------------------------ |
| Linux memory compaction | Migrates movable pages to merge scattered free pages into large contiguous blocks; can be triggered on allocation failure or periodically | Uses a similar two-level trigger: periodic compaction plus an allocation-failure fallback | The kernel must migrate physical pages with page-table/writeback coordination. This proposal primarily releases ranges by evicting cold keys in an application-level allocator and does not move hot data. |
| Redis activedefrag | Incrementally scans fragmented data in the background and moves it to new allocations without blocking the main thread | Asynchronous background and incremental compaction that does not block the main path | Redis compacts by moving data; this proposal mainly releases space by evicting cold LRU data and moving data down. |
| jemalloc / tcmalloc | Reduce internal fragmentation with size-class buckets and return memory through decay | No direct borrowing | This allocator manages contiguous addresses by offset. Its central issue is address contiguity rather than size-class internal fragmentation, and it manages large KV blobs. |
| Existing watermark eviction (`CheckAndEvict`) | Evicts LRU cold data according to total medium utilization | Reuses its `moveFunc`, single-flight guard, and LRU container primitives | Watermark eviction manages total usage; this proposal manages contiguity. Their responsibilities are independent. |

# 6. Open Questions

1. `defragCheck_` may remain true if the thread pool stops after the task is enqueued but before it runs. Decide whether to reset it proactively in the Stop path.
2. Evaluate batched or incremental scanning for the O(n) full-table scan in `BuildDefragBlobIndex` when the number of keys is very large.
3. Evaluate whether to move the `Erase` in `ForceReleaseKey` inside the lock to eliminate its very small ABA window.
4. Window selection currently sorts by descending merged space. Evaluate whether to add weighted strategies such as released-key-count-to-benefit ratio.

***

Appendix

- **References:**
  - Source: `src/memcache/csrc/meta_service/mmc_defrag_evict.h`, `mmc_meta_manager`, `mmc_meta_container`, `mmc_blob_allocator`, `mmc_global_allocator`
  - Unit tests: `defrag_evict_test.cpp`, `test_meta_manager.cpp`, `test_global_allocator.cpp`, `test_meta_container_lru.cpp`
  - Run with: `bash script/run_ut.sh meta_manager`
- **Glossary:**
  - GVA: Global Virtual Address, a memory address managed by the allocator
  - LRU: Least Recently Used eviction
  - first-fit: Select the first free block that is large enough during allocation
  - single-flight: Allow only one task of a given type to run at a time
  - move-down: Move or demote data from a higher-level medium to a lower-level medium
  - waste percent: Percentage of total free space that is unusable (small blocks below the usable-block threshold), used as a measure of fragmentation
- **Documentation update plan:** Update `docs/zh/memcache_config.md` with the `defrag_evict_threshold` setting.

