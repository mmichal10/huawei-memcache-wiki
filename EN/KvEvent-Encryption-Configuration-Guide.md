# MemCache KV Event CURVE Encryption Configuration Guide

Complete these three steps: **generate a key pair → configure the Meta service (server) → configure subscribers (clients)**.

---

## Step 1: Generate a Key Pair

```bash
cd /path/to/memcache
python ./test/python/memcache/dev_scripts/kv_event_generate_keys.py
```

The command outputs two keys (40 characters each, Z85-encoded):

```
public_key: <PUBLIC_KEY>
secret_key: <SECRET_KEY>
```

> Each run generates a different key pair. **`public_key` and `secret_key` must come from the same run.** Mixing keys causes the handshake to fail and makes troubleshooting difficult.

### Verify the Key Pair (Recommended)

Edit `kv_event_verify_key_pair.py` and set the `PUBLIC_KEY` / `SECRET_KEY` values at the top to the keys you just generated:

```python
PUBLIC_KEY = "<PUBLIC_KEY>"
SECRET_KEY = "<SECRET_KEY>"
```

Run:

```bash
python ./test/python/memcache/dev_scripts/kv_event_verify_key_pair.py
```

**The keys are valid if both tests report `result: OK` (Test 1 plaintext and Test 2 CURVE).** If Test 2 receives no message, the public and secret keys do not match; generate a new pair.

---

## Step 2: Configure the Meta Service (Server)

### Option A: Edit the Configuration File (when MetaService runs in C++)

Edit the configuration file (for example, `config/mmc-meta.conf`) and add:

```ini
# Enable KV event broadcasting
ock.mmc.kv_events.enable = true
# Transport: 0 = ZMQ (the only currently supported option)
ock.mmc.kv_events.transport = 0
# ZMQ bind address (subscribers connect to this address)
ock.mmc.kv_events.endpoint = tcp://0.0.0.0:5557

# Enable CURVE encryption
ock.mmc.zmq.enable_curve = true
# Server public key
ock.mmc.zmq.public_key = <PUBLIC_KEY>
# Server secret key (keep it confidential)
ock.mmc.zmq.private_key = <SECRET_KEY>
```

> Keep `secret_key` only on the server. **Distribute `public_key` to all subscribers through a secure channel.**

### Option B: Start MetaService with Python

See `test_mmc_eviction.py` for an example:

```python
config = MetaConfig()
...
config.kv_events_enable = True
config.kv_events_transport_type = 0
config.kv_events_endpoint = "tcp://127.0.0.1:5557"
config.zmq_enable_curve = True
config.zmq_public_key = "<PUBLIC_KEY>"
config.zmq_private_key = "<SECRET_KEY>"
```

After startup, the server log should contain:

```
kv_events: ZMQ CURVE encryption enabled (server-only auth)
```

> If you see `FALLBACK TO PLAINTEXT`, a key is missing, malformed, or mismatched. Return to Step 1.

---

## Step 3: Configure Subscribers (Clients)

**Each client must be configured with both of the following:**

1. **The server's `public_key`**, distributed by the server.
2. **A temporary key pair for the client**, generated when it connects. **Do not distribute this pair to the server.**

Reason: if a libzmq client socket has only the server's public key and no client key pair, the CURVE handshake **fails silently** (no error and no messages), making the issue difficult to diagnose. Before each connection, the client therefore calls `zmq.curve_keypair()` and configures the generated temporary pair on the socket.

To verify the setup with the provided subscriber script:

```bash
vim ./test/python/memcache/dev_scripts/kv_event_sub.py
```

Set `SERVER_PUB_KEY` to the server's public key:

```python
SERVER_PUB_KEY = "<PUBLIC_KEY>"
```

The script already calls `zmq.curve_keypair()` to generate a temporary client key pair for each connection, so no manual client-key configuration is needed.

Start the subscriber, adjusting the endpoint and topic to match the server configuration:

```bash
python ./test/python/memcache/dev_scripts/kv_event_sub.py \
    --endpoint tcp://127.0.0.1:5557 --topic "kv"
```

The setup is working when the output shows `curve=on`:

```
Subscribed to tcp://127.0.0.1:5557 (topic='kv', curve=on)
```

> If `model_name` is configured on the server, the topic becomes `kv@<model-name>`. Set `--topic` accordingly.

---

## Frequently Asked Questions

| Symptom | Cause | Resolution |
| --- | --- | --- |
| Server log shows `FALLBACK TO PLAINTEXT` | A key is missing, malformed, or mismatched. | Verify the keys with the script. Generate a new pair and update both server and client settings. |
| Client shows `curve=off` | The public key is missing or malformed. | Check that `SERVER_PUB_KEY` contains 40 characters. |
| Client shows `curve=on` but receives no messages | The client has no temporary key pair. | Confirm the script calls `zmq.curve_keypair()` (required by libzmq 4.3.5). |
| No messages and no error | The key does not match the server, the topic is wrong, or startup order is incorrect. | Verify the public key and topic. Start the server before the client. |

---

## Key Ownership at a Glance

| Key | Owner | Description |
| --- | --- | --- |
| Server `public_key` | Configured on the server and distributed to all clients | Used by clients to verify the server identity and complete the handshake. |
| Server `secret_key` | Held only by the server | Used for encryption; never disclose it. |
| Client key pair (public + secret) | Generated temporarily by each client for each connection | Used to complete the handshake. Do not distribute it; it expires when the connection ends. |

**The client never needs the server's `secret_key`, and the server does not need any client keys.**
