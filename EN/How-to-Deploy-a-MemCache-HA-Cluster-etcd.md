# Verifying MetaService High Availability with the etcd Backend

This guide explains how to verify MetaService active-standby high availability with the etcd backend. It supports bare-metal process deployment and deploying MetaService as Pods in Kubernetes (see Section 4), and can serve as a production configuration reference.

Mechanism: MetaService leader election uses etcd leases and a distributed lock:
* Lease: Each MetaService connects to the etcd backend in server mode. The Leader registers its address under a key with a TTL of about 5 seconds and renews the lease automatically. If the Leader fails, the lease expires, the address key is deleted, and the cluster detects the failure.
* Distributed lock: After the lease expires, the remaining MetaService nodes compete for the election lock. The winner confirms that no Leader exists, then registers itself as the new Leader using a new port and address key.
* Dynamic discovery: LocalService clients do not connect to a fixed MetaService address. They read the current Leader address from etcd every 2 seconds and reconnect, automatically following a new Leader after failover.

Leader election and failover are implemented by MemFabric's HaConfigStore. To enable them, change `config_store_url` on the MemCache side to use the etcd prefix; no other code changes are required.

```
            ┌──────────────┐   Register/renew lease (TTL 5s)   ┌──────────────┐
            │ MetaService  │ ───────────────────────▶ │              │
            │  (Leader)    │ ◀─────────────────────── │     etcd     │
            └──────────────┘     Read Leader address (2s)     │   (backend)  │
            ┌──────────────┐                          │              │
            │ MetaService  │ ──── Compete for election lock ─▶ │              │
            │  (Standby)   │                          └──────────────┘
            └──────────────┘                                  ▲
            ┌──────────────┐                                  │
            │LocalService ×N│ ──── Read Leader address and reconnect ────┘
            └──────────────┘
```

### 1. Prepare the Environment

Two nodes are recommended for testing. This guide assumes two mutually reachable nodes with 910B or newer chips. You also need an etcd service (it can run on one of the nodes; a single instance is sufficient for testing).

Note

```text
Replace directory paths and IP addresses in the commands with those for your environment.
```

#### 1.1 Ensure Both Machines Have the Same Time
Run the following command on both machines:
```shell
sudo timedatectl set-time "2025-10-17 17:30:00"
```

#### 1.2 Deploy etcd

Testing can use a single etcd instance. Production must use an etcd cluster with at least three nodes to avoid a single point of failure in the coordination service.

```shell
# Install etcd (version 3.5 or later is required)
yum install -y etcd

# Start a single etcd instance; this example listens on 10.0.0.1:12335
nohup etcd --listen-client-urls http://10.0.0.1:12335 \
  --advertise-client-urls http://10.0.0.1:12335 \
  --listen-peer-urls http://10.0.0.1:12380 \
  --initial-advertise-peer-urls http://10.0.0.1:12380 \
  --initial-cluster default=http://10.0.0.1:12380 &
```

Verify that the etcd service is running:
```text
# etcdctl --endpoints=http://10.0.0.1:12335 endpoint health
10.0.0.1:12335 is healthy: successfully committed proposal: took = 2ms
```

For instructions on setting up a three-node etcd cluster in production, see the [official etcd documentation](https://etcd.io/docs/v3.5/op-guide/clustering/).

#### 1.3 Check the etcd Client Shared Library

This solution requires the `libetcd_client_v3.so` shared library, built by MemFabric with `--build_etcd_backend ON`. After installing the MemFabric run package, verify that the library exists:

```shell
source /usr/local/memfabric_hybrid/set_env.sh
ls /usr/local/memfabric_hybrid/latest/etcd/lib64/libetcd_client_v3.so
```

If the library is missing, rebuild and install MemFabric using the [etcd backend deployment guide](https://gitee.com/ascend/memfabric_hybrid).

### 2. Prepare Configuration Files, Log Directories, and Packages

Create the following directory structure on both nodes:

```text
/home/meta/
├── bin
├── config
│   ├── mmc-local.conf
│   └── mmc-meta.conf
├── lib
│   └── libhcom.so
├── logs
│   └── logs
└── memfabric_hybrid-1.0.0_linux_aarch64.run  # Replace with the actual path and filename
└── memcache_hybrid-1.0.0_linux_aarch64.run   # Replace with the actual path and filename
```

#### 2.1 Configuration Files

First copy configuration templates into the `config` directory. You can obtain them by installing the run packages:
```shell
bash memfabric_hybrid-1.0.0_linux_aarch64.run  # Replace with the actual path and filename
bash memcache_hybrid-1.0.0_linux_aarch64.run   # Replace with the actual path and filename
cp -rf /usr/local/memcache_hybrid/latest/config/* /home/meta/config
```

Make the following configuration changes:

1. **Set the etcd backend address (the key setting for this solution).** Change `config_store_url` from the `tcp://` prefix to `etcd://` and point it to the etcd service deployed in Section 1.2. Both MetaService nodes and all LocalService nodes must use the same etcd backend.

   `mmc-meta.conf` (on MetaService nodes)
   ```ini
   ock.mmc.meta_service.config_store_url = etcd://10.0.0.1:12335
   ```

   `mmc-local.conf` (on LocalService nodes)
   ```ini
   ock.mmc.local_service.config_store_url = etcd://10.0.0.1:12335
   ```

   Note: Use the URL format `etcd://<ip>:<port>`, where `ip` is the etcd listen address and `port` is required. When multiple clusters share one etcd instance, isolate them by appending `#instance-id` to the URL, for example `etcd://10.0.0.1:12335#prod` and `etcd://10.0.0.1:12335#test`.

2. **`meta_service_url` no longer needs an actual address.** In etcd mode, LocalService discovers the Leader address dynamically from etcd, so the default value can be kept.

   mmc-meta.conf / mmc-local.conf
   ```ini
   ock.mmc.meta_service_url = tcp://127.0.0.1:5000
   ```

3. Configure TLS certificates. If TLS is not used, the switches can be disabled (disabling TLS may create security risks and is not recommended in production).

   mmc-meta.conf
   ```ini
   ock.mmc.tls.enable = false
   ock.mmc.config_store.tls.enable = false
   ```

   mmc-local.conf
   ```ini
   ock.mmc.tls.enable = false
   ock.mmc.config_store.tls.enable = false
   ock.mmc.local_service.hcom.tls.enable = false
   ```

5. Set the data-transfer protocol. This example uses `host_rdma`; use `show_gids` to find the RDMA interface IP and update the IP in `hcom_url`.

   mmc-local.conf
   ```ini
   ock.mmc.local_service.hcom_url = tcp://X.X.X.X:7000
   ```

5. Set `world_size` to match the number of LocalService processes to start.

   mmc-local.conf
   ```ini
   ock.mmc.local_service.world_size = 16
   ```

6. Set the MetaService log path to a directory on mounted storage so service logs can be viewed directly.

   mmc-meta.conf
   ```ini
   ock.mmc.log_path = /home/meta/logs
   ```

#### 2.2 Dependency Library

When using the `host_rdma` data-transfer protocol, copy `libhcom.so` to `/home/meta/lib`.
See the [hcom project](https://atomgit.com/openeuler/ubs-comm/tree/br_dev_container_20260228) for the latest shared library.

#### 2.3 Packages

This deployment depends on [MemCache](https://gitcode.com/Ascend/memcache) and [MemFabric](https://gitcode.com/Ascend/memfabric_hybrid).
Build the packages according to the official instructions, enabling the etcd backend in MemFabric with `--build_etcd_backend ON`, and install them on all nodes.

### 3. Start the High-Availability Service Cluster

#### 3.1 Start MetaService (Two Processes: Active and Standby)

Start a MetaService process on each of the two nodes. The `config_store_url` values in both configuration files must be identical:

```shell
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/memfabric_hybrid/set_env.sh
source /usr/local/memcache_hybrid/set_env.sh
export MMC_META_CONFIG_PATH=/home/meta/config/mmc-meta.conf
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/meta/lib
mmc_meta_service &
```

After startup, inspect the logs. One node should become Leader and the other Standby:

```text
# Leader node log (/home/meta/logs/logs/mmc-meta.log)
Attempting to become leader
Became leader after 1 attempts
Registered MetaService address in backend: 10.0.0.1:32145
MetaNetServer binding on 10.0.0.1:32145

# Standby node log
became follower of 10.0.0.1:32145
```

Note: The process automatically selects a Leader port from the configured range (9000–65535 by default); no manual port configuration is needed. You can narrow the range with `MF_CONFIG_STORE_PORT_START` and `MF_CONFIG_STORE_PORT_END`. Ensure the firewall allows the selected ports.

#### 3.2 Verify the Election Result

Query etcd for the current Leader (the value is the Leader node's `ip:port`, without a protocol prefix):

```shell
# Get the current Leader's config store address
etcdctl --endpoints=http://10.0.0.1:12335 get /memfabric_hybrid/config_store/meta/leader

# Get the MetaService address used by LocalService
etcdctl --endpoints=http://10.0.0.1:12335 get /memcache/meta/service_addr

# List all related keys
etcdctl --endpoints=http://10.0.0.1:12335 get /memfabric_hybrid --prefix
etcdctl --endpoints=http://10.0.0.1:12335 get /memcache --prefix
```

#### 3.3 Start LocalService Clients

Start LocalService on the application nodes (this example starts it with Python):

```shell
source /usr/local/Ascend/ascend-toolkit/set_env.sh
source /usr/local/memfabric_hybrid/set_env.sh
source /usr/local/memcache_hybrid/set_env.sh
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/meta/lib
python3 /usr/local/memcache_hybrid/latest/aarch64-linux/script/ha/test-mmc-meta-ha.py > /home/meta/logs/mmc-local.log 2>&1 &
```

Finally, verify that everything started successfully (no errors in the LocalService log and a successful connection to MetaService).
```text
# tail -f /home/meta/logs/mmc-local.log
```

Notes:
* In etcd mode, the MetaService address and port are dynamic. No fixed port configuration or Kubernetes Service, ClusterIP, or Lease resources are needed.
* If the application still runs in Kubernetes, MetaService can be deployed as Pods; see Section 4.
* When multiple clusters share one etcd instance, use a different `#instance-id` for each to prevent data from mixing.

### 4. Deploy as Pods in a Kubernetes Cluster (Optional)

This section explains how to deploy MetaService as Pods when the application runs in Kubernetes. In etcd mode, leader election is handled entirely by HaConfigStore inside the MetaService process (leases and a distributed lock). Kubernetes only starts and maintains the processes and **does not participate in leader election**, so no extra Kubernetes resources (such as Services, RBAC permissions, or role labels) are needed for election.

Prerequisites:
* All MetaService and LocalService Pods can access the etcd endpoint.
* Clients can connect directly to the Leader's dynamic ports (the default range is 9000–65535; see Section 4.3 for firewall rules).
* Configuration files are prepared as described in Section 2 (`config_store_url` uses an `etcd://` address), and the image has been pushed to the registry.

#### 4.1 Where to Deploy etcd

etcd is the basis for leader election, so its availability directly determines the system's high availability. Choose one of these deployment options:

* Deploy separately outside the cluster (recommended for production): Separate etcd's failure domain from the MemCache cluster. Set up an etcd cluster with at least three nodes as described in Section 1.2, and ensure Kubernetes nodes can reach it.
* Deploy inside the cluster: Run etcd as a StatefulSet in Kubernetes (see the official etcd Helm or manifest deployment). Keep etcd replicas and MetaService Pods in separate failure domains.

In either case, put the etcd address in `config_store_url` in each node's configuration file (configured in Section 2). Kubernetes does not need an etcd Service for MetaService; Pods can connect directly to the etcd endpoint.

#### 4.2 Create Active and Standby MetaService Pods

The sample YAML fully defines the active and standby MetaService Pods. Replace placeholders such as image names and package values with actual values for your environment. **The sample is for reference only and may have security risks. Do not use it directly in production; adapt it to your environment.**

```yaml
# meta-pods-etcd-demo.yaml: active and standby MetaService Pods for etcd mode
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: meta-service-pod
  namespace: ns-memcache
spec:
  serviceName: meta-service
  replicas: 2
  selector:
    matchLabels:
      app: meta-service
  template:
    metadata:
      labels:
        app: meta-service   # Active/standby state is tracked in etcd, not by Pod labels
    spec:
      # The process does not access Kubernetes resources; serviceAccountName is not needed.
      terminationGracePeriodSeconds: 10
      hostNetwork: true     # The IP registered in etcd is the Pod IP, which is the host IP with hostNetwork; direct client connections are simpler.
      dnsPolicy: ClusterFirstWithHostNet
      topologySpreadConstraints:   # Keep the active and standby Pods on separate nodes.
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: meta-service
      containers:
        - name: meta-service
          image: 127.0.0.1:5000/memcache:1028   # Replace with the actual image.
          command: [ "/bin/bash", "-c" ]
          args:
            - |
              source /usr/local/Ascend/ascend-toolkit/set_env.sh && \
              bash /home/meta/memfabric_hybrid-1.0.0_linux_aarch64.run && \
              bash /home/meta/memcache_hybrid-1.0.0_linux_aarch64.run && \
              echo "install end." && \
              sleep 20 && \
              exec /usr/local/memcache_hybrid/latest/aarch64-linux/bin/mmc_meta_service
          startupProbe:
            exec:
              command: ["sh", "-c", "curl -fsS --max-time 2 http://${META_POD_IP}:8000/livez"]
            failureThreshold: 30
            periodSeconds: 10
            timeoutSeconds: 3
          livenessProbe:
            exec:
              command: ["sh", "-c", "curl -fsS --max-time 2 http://${META_POD_IP}:8000/livez"]
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          readinessProbe:
            exec:
              command: ["sh", "-c", "curl -fsS --max-time 2 http://${META_POD_IP}:8000/livez"]
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          env:
            - name: LD_LIBRARY_PATH
              value: "/usr/local/python3.11/lib:/usr/local/memfabric_hybrid/latest/aarch64-linux/lib64:/usr/local/memcache_hybrid/latest/aarch64-linux/lib64"
            - name: MMC_META_CONFIG_PATH
              value: /home/meta/config/mmc-meta.conf   # Already contains the etcd:// address; no change needed.
            - name: META_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - name: host-home
              mountPath: /home
            - name: ascend-dir
              mountPath: /usr/local/Ascend
            - name: host-ping
              mountPath: /usr/local/bin/ping
            - name: smi
              mountPath: /usr/local/bin/npu-smi
            - name: drive
              mountPath: /usr/local/Ascend/driver
            - name: firmware
              mountPath: /usr/local/Ascend/firmware
            # NPU device mounts: add the required devices for the node's NPU count (davinci0 through N, davinci_manager, etc.; type: CharDevice).
      initContainers:
        - name: change-pod-ip
          image: 127.0.0.1:5000/memcache:1028
          command: ["/bin/sh", "-c"]
          args:
            - |
              # In etcd mode, keep the static etcd:// address in config_store_url unchanged.
              # Only metrics_url needs to be updated to the Pod IP for probe access.
              sed -i '/^ock.mmc.meta_service.metrics_url/c ock.mmc.meta_service.metrics_url = http://metaServiceUrl:8000' /home/meta/config/mmc-meta.conf && \
              sed -i s/metaServiceUrl/$META_POD_IP/ /home/meta/config/mmc-meta.conf
          env:
            - name: META_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          volumeMounts:
            - name: host-home
              mountPath: /home
      volumes:
        - name: host-home
          hostPath:
            path: /home
            type: Directory
        # Add other volumes, such as hostPath mounts for NPU devices, for the actual node environment.
```

To deploy LocalService clients as Pods, use `local-pods-demo.yaml` in `test/k8s_deploy` as a reference. The only requirement is that `config_store_url` in the mounted `mmc-local.conf` has been changed to an `etcd://` address as described in Section 2.

Create the resources and verify them:
```shell
kubectl apply -f meta-pods-etcd-demo.yaml
kubectl get pod -n ns-memcache -o wide
```

Verify the election result (as with bare-metal deployment, etcd is authoritative):
```shell
etcdctl --endpoints=http://10.0.0.1:12335 get /memfabric_hybrid/config_store/meta/leader
etcdctl --endpoints=http://10.0.0.1:12335 get /memcache/meta/service_addr
```

#### 4.3 Firewall Rules and Network Policies

The Leader process automatically selects two available ports from the configured range (9000–65535 by default)—the config store port and the MetaService port—and registers them in etcd. Clients connect directly to these ports. In Kubernetes, note the following:

* If the cluster uses NetworkPolicy, such a large port range is not allowed by default. Explicitly allow it for meta-service and local-service Pods, or narrow the range with environment variables and allow the smaller range:
```shell
# Add these entries to env in meta-pods-etcd-demo.yaml (use the same range for both MetaService Pods).
- name: MF_CONFIG_STORE_PORT_START
  value: "9000"
- name: MF_CONFIG_STORE_PORT_END
  value: "9100"
```
* A range of at least several dozen ports is recommended. The Leader needs two available ports at once; a narrow range or ports occupied by stale processes can cause election to fail.
* In a multi-node deployment, different nodes can use the same port numbers because each binds to its own IP. The sample uses hostNetwork and spreads the two Pods across nodes with topologySpreadConstraints, so the ports do not conflict.

#### 4.4 Verify Failover in Kubernetes

First identify the Pod hosting the current Leader (the `ip:port` in etcd corresponds to a node's Pod):
```shell
etcdctl --endpoints=http://10.0.0.1:12335 get /memfabric_hybrid/config_store/meta/leader
kubectl get pod -n ns-memcache -o wide
```

Assume `meta-service-pod-0` is the Leader. Delete the Pod and observe the failover:
```shell
kubectl delete pod meta-service-pod-0 -n ns-memcache
```

Kubernetes starts a replacement Pod after one is deleted to maintain the desired Pod count. The active-standby failover itself does not depend on Kubernetes: after the original Leader is killed, its etcd lease expires in about 5 seconds, and the standby Pod detects this and elects itself within about 4 seconds. The failover timeline is the same as the bare-metal scenario in Section 5.3; overall recovery is usually under 1 minute. Verify as follows:

```shell
# Verify that etcd now lists the other node as Leader.
etcdctl --endpoints=http://10.0.0.1:12335 get /memfabric_hybrid/config_store/meta/leader

# Check that the new Leader Pod's log records its promotion.
kubectl logs meta-service-pod-1 -n ns-memcache -c meta-service | grep -E "Became leader|MetaNetServer binding"

# Verify that the application recovers and put/get operations work.
```

Notes:
* The probe (`/livez`) reports only whether the process is alive, not its active/standby role. A Standby Pod being alive is expected; use etcd query results and logs to determine the role.
* A Kubernetes node failure (power or network loss) has the same effect as killing the Leader process and recreating its Pod; the failover mechanism is the same.

### 5. Check Service Status and Verify Leader Election

#### 5.1 Check Service Status

Query the current Leader (the value in etcd is authoritative):
```shell
etcdctl --endpoints=http://10.0.0.1:12335 get /memfabric_hybrid/config_store/meta/leader
etcdctl --endpoints=http://10.0.0.1:12335 get /memcache/meta/service_addr
```

Query the cluster size and live nodes:
```shell
etcdctl --endpoints=http://10.0.0.1:12335 get /memfabric_hybrid/config_store/data/world_size
etcdctl --endpoints=http://10.0.0.1:12335 get /memfabric_hybrid/config_store/data/alive_rank_list
```

Note: In etcd mode, the active/standby role in MetaService HTTP endpoints such as `/health` and `/role` may show `unknown`. **Do not** use these fields to determine the role; use etcd query results and logs.

#### 5.2 Verify That MemCache Works
```shell
# Run an interactive test and enter put/get/remove commands to verify functionality.
python interactive_app.py
```

#### 5.3 Verify Active-Standby High Availability

Assume node 1 (`10.0.0.1`) is the current Leader. Kill its process and observe the failover:

```shell
# Run on the Leader node.
kill -9 <mmc_meta_service_pid>
```

Failover timeline (overall recovery is usually under 1 minute; observed recovery is about 10–35 seconds):
* After the Leader is killed, its etcd lease expires in about 5 seconds and the address key is deleted.
* The Standby detects the failure in about 4 seconds, competes for the election lock, becomes the new Leader, and registers its address.
* The new Leader waits about 5 seconds for recovery, then accepts LocalService registrations.
* LocalService rediscovers the Leader address every 2 seconds and reconnects automatically. put/get requests may fail and retry during this period; the application recovers after reconnection.

Verify the failover result:
```shell
# Confirm that node 2 (10.0.0.2) is now the Leader.
etcdctl --endpoints=http://10.0.0.1:12335 get /memfabric_hybrid/config_store/meta/leader

# Check that node 2's log records its promotion.
Attempting to become leader
Became leader after N attempts
MetaNetServer binding on 10.0.0.2:xxxxx

# Verify that the application recovers and put/get operations work.
```

Note: This solution also covers a hung Leader (for example, a process stopped with SIGSTOP or deadlocked while TCP remains connected). The standby periodically compares the Leader record in etcd and triggers Leader demotion and a new election if they disagree, without manual intervention.

### 6. FAQ

#### 6.1 MetaService Fails to Start with “Failed to Open Library [libetcd_client_v3.so]”
The etcd client shared library was not found. Verify that the etcd backend was enabled when building MemFabric (`--build_etcd_backend ON`) and that the environment variables were sourced or the library path was added to `LD_LIBRARY_PATH` manually.

#### 6.2 MetaService Repeatedly Fails to Become Leader
Check that enough ports are available in the configured range (9000–65535 by default). The Leader must bind two ports (for config store and MetaService); a range that is too narrow or ports occupied by stale processes can cause election to fail. Adjust the range with these environment variables:
```shell
export MF_CONFIG_STORE_PORT_START=9000
export MF_CONFIG_STORE_PORT_END=10000
```

#### 6.3 Troubleshooting
View the MetaService log:
```shell
tail -f /home/meta/logs/logs/mmc-meta.log
```
View the LocalService log:
```shell
tail -f /home/meta/logs/mmc-local.log
```
Useful log messages include `[ETCD] Connection initialized successfully` (connected to etcd), `Became leader` (promoted), `became follower` (demoted), and `MetaNetServer binding on` (service port).

#### 6.4 LocalService Cannot Connect to MetaService
Check the following in order:
* The etcd service is healthy (endpoint health).
* Both sides use identical `config_store_url` values, including the `#instance-id`.
* The LocalService node can reach the Leader's dynamic ports (query `service_addr` in etcd and verify with telnet).
* `world_size` matches the actual number of LocalService processes.

**Recommendation:**<br>Write service logs to a directory on mounted storage so you can monitor operation and troubleshoot issues.

### 7. Notes

1. Production requires an etcd cluster with at least three nodes. Connections between the process and etcd do not currently support username/password authentication or TLS. TLS settings for other links (config store, MetaNet, HCOM, and others) remain effective. Secure etcd access through network isolation, such as a dedicated etcd cluster or security-group allowlists.
2. During failover (usually under 1 minute), put/get requests may fail and retry. The client's retry budget must cover this window; check the retry limit in custom clients.
3. After the Standby becomes Leader, metadata is recovered from backup records and LocalService re-registration. Asynchronous backup data not yet written to disk may be lost if the Leader is killed; set `asyncFlushIntervalMs` according to the sensitivity of the business data.
4. When multiple clusters share one etcd instance, isolate them with distinct instance IDs in the URL.
5. Follow the official etcd documentation to harden the etcd cluster.
