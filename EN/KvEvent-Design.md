**Authors:** @Ascend/Memcache
**Created:** 2026-07-13
**Updated:** 2026-07-13
**Related Issues/PRs:** 
<https://gitcode.com/Ascend/memcache/pull/334>
<https://gitcode.com/Ascend/memcache/pull/363>

***

# 1. Overview

## 1.1 Introduction

KvEvent broadcasts write-operation events. When metadata is stored or deleted, events are sent to subscribers through ZMQ PUB. This feature integrates with PyMotor's kv-conductor to calculate KV affinity and enable cache-aware prefill scheduling.

## 1.2 Motivation

Current situation: Agentic AI multi-turn conversations can easily reach hundreds of thousands of tokens, and requests share many of the same prefixes (system prompts, conversation history, and RAG knowledge bases).

Problem: Random scheduling across requests prevents KV Cache hits. Each turn repeats prefill for the full context, resulting in high TTFT and substantial wasted compute.

Solution: Cache-aware scheduling precisely identifies the longest prefix match and intelligently routes requests to the optimal node to maximize KV reuse.

Cross-request KV reuse: Store computed KV in pooled MemCache storage so subsequent requests can hit the cache directly and avoid repeated prefill.

## 1.3 Goals

Goal: Enable PyMotor to observe KV pool state and perform cache-aware prefill scheduling, maximizing KV reuse and reducing TTFT.

Non-goal: KvEvent only broadcasts events; it does not perform scheduling or orchestration. Performance gains depend on PyMotor scheduling requests according to KV reuse.

# 2. Use Case Analysis (Requirements Analysis)

| Use Case | Priority | Performance / Functional Goal | Notes |
| ------ | --- | ------------------------------------ | ------------------------------------------------------ |
| Cache state awareness | P0 | Enabling KvEvent broadcasts must not degrade MemCache performance | ZMQ uses an asynchronous queue and DONTWAIT so the main path is not blocked |
| HA primary/standby switch | P1 | Events must not be lost or reordered during failover | `SetActive(false)` stops queue writes; `SetActive(true)` resumes them. In HA mode, the default is inactive |
| Status information | P2 | Provide a RESTful API for event sent/dropped statistics to improve observability | `KvEventStats` covers published/dropped/skipped dimensions |

# 3. Solution Design

## 3.1 Overall Design

After a metadata store or delete operation succeeds, a KV event is broadcast asynchronously through a ZMQ PUB socket. The overall architecture is:

```
Write succeeds → MmcKvEventRuntime.OnMetaStored/OnMetaRemoved
               ↓
         KvEventPublisher.Enqueue (asynchronous queue, protected by a mutex)
               ↓
         WorkerLoop thread → dequeue in batches → BuildEvent + SerializeEventBatch (msgpack)
               ↓
         IKvEventTransport.Send (ZMQ PUB, ZMQ_DONTWAIT)
               ↓
         ZMQ subscriber (kv-conductor) ← three frames: topic | seq(BE64) | payload(msgpack)
```

Key design constraints:

- **Non-blocking:** The caller thread enqueues the event and returns immediately. ZMQ sends use `ZMQ_DONTWAIT` to prevent the worker thread from blocking.
- **Priority:** REMOVED/CLEARED are high-priority events. When the queue is full, they can take the place of STORED events to prevent deletion events from being lost.
- **HA support:** Standby nodes default to `active=false`; publishing is dynamically enabled or disabled through `SetActive` during failover.

## 3.2 Technology Selection

| Option | Advantages | Disadvantages | Decision |
| ----------- | -------------------------------- | --------------- | ------ |
| ZMQ PUB/SUB | Mature and stable, zero-copy sends, native multi-subscriber support, fast shutdown with LINGER=0 | No built-in retries; SNDHWM must be managed | **Selected** |

### Encryption Options

| Option | Advantages | Disadvantages | Decision |
| ------------ | ---------------------------------------- | --------------------------- | ------ |
| TLS over TCP | Enterprise security standard and mature certificate ecosystem | Requires separate certificate management, has significant handshake overhead, and does not suit embedded ZMQ scenarios | Rejected |
| Application-layer encryption | Flexible and controllable | Complex to implement; key negotiation must be managed manually | Rejected |
| ZMQ CURVE | Officially recommended by ZMQ, built into libzmq, no extra dependencies, lightweight NaCl cryptography | Supports encryption and authentication only, without complex access control | **Selected** |

**ZMQ CURVE encrypted communication:**

CURVE is ZMQ's recommended encryption mechanism. It is based on the NaCl cryptographic library and provides end-to-end encryption with optional client authentication.

- **Encryption algorithms:** Curve25519 (key exchange) + XSalsa20-Poly1305 (symmetric encryption)
- **Key format:** A 40-character Z85-encoded string (32-byte binary). Key pairs must be generated through the native ZMQ API (`zmq_curve_keypair()`) or the underlying libsodium library. These interfaces internally call the operating system's cryptographically secure pseudorandom number generator (CSPRNG), ensuring keys have unpredictable, high entropy.
Offline generation principle: Generate keys in an offline environment before deployment. Never generate them dynamically at runtime or use a weak random source.
- **Authentication modes:** Server-only authentication (encryption only) or mutual server/client authentication (requires the ZAP protocol)
- **Official documentation:** <https://libzmq.readthedocs.io/en/latest/zmq_setsockopt.html>

This project uses server-only authentication: communication is encrypted without client authentication. If encryption is enabled but the key configuration is invalid, the system automatically falls back to unencrypted mode and logs an ERROR.

## 3.3 Functional and Performance Design

### Event Types

| Event | KvEventType | Meaning | Priority |
| ------- | ----------- | ------------- | --- |
| STORED | `STORED` | KV block stored | Normal |
| REMOVED | `REMOVED` | KV block removed | High |
| CLEARED | `CLEARED` | All data cleared for the specified nd/rank | High |

### Core Data Flow

1. **Enqueue:** The caller thread locks the queue and enqueues the event. STORED is dropped when the queue reaches `queueCapacity`. REMOVED/CLEARED events can replace STORED events until the queue reaches `2×queueCapacity` (`HardQueueCapacity`); if it is still full, the event is dropped and counted in `droppedHighPriorityEvents`.
2. **Send:** The worker dequeues batches (up to `maxBatchSize`), calls BuildEvent for each event (extracting `blockHash` and populating metadata), serializes with msgpack, and sends through ZMQ. Send failures are counted in `droppedEvents`/`droppedStoredEvents`/`droppedHighPriorityEvents`.
3. **ZMQ frame format:** Three-frame multipart message — `topic | seq(BE uint64) | payload(msgpack map array)`.

### KvEvent Fields

| Field | Type | Applicable event types | Description |
| ---------------- | --------------------- | ---------------- | ---------------------------------- |
| event\_id | uint64 | common | Monotonically increasing per (medium, backendId) |
| timestamp | uint64 | common | Unix timestamp in milliseconds |
| event\_type      | string                | common           | "stored"/"removed"/"cleared"       |
| model\_name | string? | common | Model name |
| block\_size | uint32? | common | KV block size |
| additional\_salt | string? | common | Additional salt |
| lora\_name | string? | common | LoRA name |
| tenant\_id | string | common | Tenant ID; defaults to "default" |
| medium           | string?               | common           | "xpu"(HBM)/"cpu"(DRAM)/"disk"(SSD) |
| backend\_id | string | common | Target backend identifier |
| dp\_rank | uint32? | common | Compute DP rank (currently left empty) |
| seq\_hashes | uint64\[] or string\[] | stored + removed | List of block hashes parsed from object keys |
| base\_block\_idx | nil | stored + removed | Placeholder field; currently emitted as nil |
| parent\_hash | nil | stored only | Placeholder field; currently emitted as nil |
| token\_ids | nil | stored only | Placeholder field; currently emitted as nil |

> **Applicable event types:** common = present for STORED/REMOVED/CLEARED; stored + removed = only STORED and REMOVED (CLEARED returns immediately after `seq\_hashes`); stored only = only STORED.

### Performance Constraints

- Enqueue: O(1) locked enqueue that does not affect the write hot path.
- ZMQ send: Non-blocking `ZMQ_DONTWAIT`; SNDHWM=10000. When full, it returns EAGAIN and increments the dropped-event statistics.
- Safe destruction: `ZMQ_DONTWAIT` prevents the worker from blocking on send, and the `stop_` flag plus `join()` ensures reliable shutdown. `ZMQ_LINGER=0` discards unsent frames when the socket closes.

## 3.4 Security, Privacy, and DFX Design

Compatibility: The module is modular, with KvEvent code kept separate; broadcast calls are added only after successful writes. The `mmc_kv_events_config_t` configuration is a substructure of `mmc_meta_service_config_t` and defaults to `enable=false`, so existing configurations are unaffected.

Maintainability: Transport is implemented as the `IKvEventTransport` abstract base class, exposing only the `Send(seq, payload)` interface to simplify adding other broadcast mechanisms in the future (such as HTTP or gRPC).

Testability: The repository pipeline runs unit tests and tracks coverage (line coverage >=70%, branch coverage >=40%). Unit tests use a mock transport to verify enqueueing, dropped events, batch sends, HA failover, and other scenarios.

Reliability:

- Events dropped during enqueue are counted (`droppedEvents`/`droppedStoredEvents`/`droppedHighPriorityEvents`).
- Send failures are counted in the same statistics when Send returns false.
- Reads and writes of `MmcMetaChangeCallbacks` are protected by `std::mutex`, eliminating undefined behavior from callback write races during `Stop()`.
- ZMQ `DONTWAIT` prevents destruction from hanging if the worker blocks.

## 3.5 Programming and Invocation Design

### 3.5.1 Basic Programming Model

\*\*Development environment:\*\* Same as the main MemCache project (C++17, CMake >= 3.12, Ninja/Make).

**Development constraints:**

- The kv\_event module is for internal use only and is not exposed as a public C/Python API.
- The `MMC_ENABLE_KV_EVENTS` compile-time macro controls feature compilation. If undefined, `MmcKvEventRuntime` is entirely a no-op.

**Acceptance criteria:**

- Functional: Start the meta service with `kvEvents.enable=true` and verify that subscribers receive STORED/REMOVED/CLEARED events.
- Performance: MemCache benchmark performance with kv\_events enabled must be no worse than with it disabled (O(1) enqueue; ZMQ DONTWAIT is non-blocking).
- Observability: Retrieve `KvEventStats` through the RESTful API and verify that published/dropped counts are correct.

### 3.5.2 Configuration

KV event broadcasting is enabled through the MetaService configuration file and supports the following options:

| Option | Type | Default | Description |
| --------------------------------------------- | ------ | --------- | -------------------------------------------------------------------- |
| ock.mmc.kv\_events.enable | bool | false | Whether KV event broadcasting is enabled |
| ock.mmc.kv\_events.endpoint | string | "" | ZMQ PUB bind address, for example `tcp://127.0.0.1:5557` |
| ock.mmc.kv\_events.model\_name | string | "" | Model name used to build the kv\_event envelope |
| ock.mmc.kv\_events.tenant\_id | string | "default" | Tenant ID used to build the kv\_event envelope |
| ock.mmc.kv\_events.block\_size | uint32 | 0 | KV block size used to build the kv\_event envelope |
| ock.mmc.kv\_events.queue\_capacity | uint32 | 65536 | Internal queue soft capacity; hard capacity is twice this value |
| ock.mmc.kv\_events.hash\_as\_int | bool | true | `seq\_hashes` encoding: `true` emits a uint64 array; `false` emits an array of hex strings |
| ock.mmc.kv\_events.curve\_enable | bool | false | (In development) Whether to enable ZMQ CURVE encryption. When enabled, `server\_public\_key` and `server\_secret\_key` must be configured |
| ock.mmc.kv\_events.curve\_server\_public\_key | string | "" | (In development) ZMQ CURVE server public key (40-character Z85 encoding). Required when encryption is enabled |
| ock.mmc.kv\_events.curve\_server\_secret\_key | string | "" | (In development) ZMQ CURVE server secret key (40-character Z85 encoding). Required when encryption is enabled; sensitive information |

**Encrypted communication:**

- When `curve_enable=true`, the ZMQ PUB socket uses the CURVE encryption protocol and all event data is encrypted in transit.
- If encryption is enabled but the public/private key is missing or malformed, the system automatically falls back to unencrypted mode and logs an ERROR.
- Subscribers (such as kv-conductor) must be configured with the server's public key to decrypt and receive events. Distribute the public key to subscribers offline through a secure channel.
- Generate keys in advance with the `zmq_curve_keypair()` function or the `zmqcert` tool to create a Z85-encoded key pair.

### 3.5.3 Programming Manual

Add the `kv_events` configuration options to the existing `doc/memcache_config.md` and document the `/stats/kv_events` query endpoint in `doc/memcache_restful_api.md`. Do not create a separate manual.

# 4. Limitations and Risks

| Risk | Impact | Mitigation |
| -------------- | -------------------------------------- | ----------------------------------------------------------------------- |
| ZMQ subscriber disconnects or consumes slowly | Events are dropped when SNDHWM is full, and the kv-conductor affinity table may be temporarily incomplete | DONTWAIT and dropped-event statistics provide visibility; lower SNDHWM to reduce stale backlog |
| HA standby does not publish | Events are not broadcast while the node is standby; affinity must be rebuilt after failover | Publishing resumes immediately after `SetActive(true)`; kv-conductor should tolerate a brief gap |
| Incompatible objectKey format | Failed `seq\_hashes` parsing increases `skippedUnparsedKeys` | Monitor `skippedUnparsedKeys`; keep `ExtractBlockHashFromObjectKey` synchronized with the upstream objectKey format |

# 5. Related Technologies

| Project / Community | Design | Adopted in this project | Differences |
| ----------------------- | ---------------------------------------- | ------------------------ | ------------------------------------------------ |
| vLLM kv event publisher | Broadcasts KV block additions/removals using ZMQ PUB, msgpack encoding, and the `seq\_hashes` field | Event data model, msgpack serialization format, and field names | vLLM publishes directly within one process and has no encryption; this project spans MetaService processes and adds an asynchronous queue, batching, priorities, HA, and CURVE encryption |
| Redis Pub/Sub | Channel-based publish/subscribe supporting multiple consumers | One-to-many broadcast model | Redis requires a separate deployment; this project embeds ZMQ with no extra dependencies |
| Kafka | Distributed message queue with persistence and retries | None | Kafka is too heavyweight; persistence and replay are unnecessary, and ZMQ meets low-latency requirements |

# 6. Open Issues

1. Communication encryption is still in development.

***

Appendix

- **References:** vLLM kv-event implementation, ZMQ PUB/SUB documentation, and the msgpack specification
- **Glossary:**
  - kv-conductor: PyMotor-side KV affinity calculation service
  - TTFT：Time To First Token
  - SNDHWM: ZMQ Send High Water Mark, the maximum number of buffered messages at the sender
  - DONTWAIT: ZMQ non-blocking send flag
- **Documentation updates:** Also update `doc/memcache_config.md` (kv\_events options) and `doc/memcache_restful_api.md` (stats query endpoint)

