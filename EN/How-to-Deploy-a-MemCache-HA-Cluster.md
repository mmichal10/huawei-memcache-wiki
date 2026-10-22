# Verifying MetaService High Availability on Kubernetes
This guide explains how to verify MetaService active-standby high availability in a Kubernetes cluster. The configuration can also serve as a reference for setting up MetaService high availability in production.

Mechanism: MetaService leader election uses Kubernetes Service (`type=ClusterIP`) and Lease resources:
* Service (`type=ClusterIP`): Provides all LocalService clients with a unified MetaService endpoint and routes requests to the active MetaService (the Pod with `role=master`).
* Lease: Provides a distributed lock and records the active service Pod name and lease expiration time.
Each MetaService periodically checks whether the Lease has expired. If so, it attempts to update the Lease and become leader.
The leader renews its lease periodically. If the active node fails and the lease expires, another MetaService competes to become leader and updates its label to `role=master`.

![metaservice-ha.png](https://raw.gitcode.com/Ascend/memcache/raw/master/docs/zh/source/memcache_metaservice_ha.png)

### 1. Prepare the Environment

Two nodes are recommended for testing. This guide assumes two mutually reachable nodes with 910B or newer chips.

Note

```text
Replace directory paths and IP addresses in the commands with the paths and addresses for your environment.
```

#### 1.1 Ensure Both Machines Have the Same Time
Run the following command on both machines:
```shell
sudo timedatectl set-time "2025-10-17 17:30:00"
```

#### 1.2 Install Kubernetes

Install the environment by following the Kubernetes installation guide.
See the [Kubernetes documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

```shell
# Enable Kubernetes command completion
sudo apt install -y bash-completion
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo "alias k=kubectl" >> ~/.bashrc
echo "complete -F __start_kubectl k" >> ~/.bashrc
source ~/.bashrc
```

Replace `master1` with your control-plane node name (for example, `k8s-master`) so workloads can also be scheduled on it.
```shell
kubectl taint node master1 node-role.kubernetes.io/control-plane:NoSchedule-
```

```text
k8s_deploy# k describe node | grep Taints
Taints:             <none>
Taints:             <none>
```

Verify that Kubernetes services are running:
```text
k8s_deploy# k get all -A -o wide
NAMESPACE     NAME                                          READY   STATUS    RESTARTS       AGE     IP              NODE      NOMINATED NODE   READINESS GATES
kube-system   pod/calico-kube-controllers-d864bb84b-jsq5n   1/1     Running   0              29h     127.0.0.1   master1   <none>           <none>
kube-system   pod/calico-node-4k7t6                         0/1     Running   29 (27h ago)   29h     127.0.0.1     work1     <none>           <none>
kube-system   pod/calico-node-gqtb6                         0/1     Running   0              29h     127.0.0.1     master1   <none>           <none>
kube-system   pod/coredns-88f77f6b5-g2qhc                   1/1     Running   0              29h     127.0.0.1   master1   <none>           <none>
kube-system   pod/coredns-88f77f6b5-q76xv                   1/1     Running   0              29h     127.0.0.1   master1   <none>           <none>
kube-system   pod/etcd-master1                              1/1     Running   1              29h     127.0.0.1     master1   <none>           <none>
kube-system   pod/kube-apiserver-master1                    1/1     Running   1              29h     127.0.0.1     master1   <none>           <none>
kube-system   pod/kube-controller-manager-master1           1/1     Running   1              29h     127.0.0.1     master1   <none>           <none>
kube-system   pod/kube-proxy-b855z                          1/1     Running   0              29h     127.0.0.1     master1   <none>           <none>
kube-system   pod/kube-proxy-fr29g                          1/1     Running   0              29h     127.0.0.1     work1     <none>           <none>
kube-system   pod/kube-scheduler-master1                    1/1     Running   1              29h     127.0.0.1     master1   <none>           <none>

NAMESPACE     NAME                                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE     SELECTOR
default       service/kubernetes                    ClusterIP   127.0.0.1      <none>        443/TCP                  29h     <none>
kube-system   service/kube-dns                      ClusterIP   127.0.0.1     <none>        53/UDP,53/TCP,9153/TCP   29h     k8s-app=kube-dns

NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE   CONTAINERS    IMAGES                                 SELECTOR
kube-system   daemonset.apps/calico-node   2         2         0       2            0           kubernetes.io/os=linux   29h   calico-node   127.0.0.1:5000/calico/node:v3.28.0   k8s-app=calico-node
kube-system   daemonset.apps/kube-proxy    2         2         2       2            2           kubernetes.io/os=linux   29h   kube-proxy    127.0.0.1:5000/kube-proxy:v1.30.3    k8s-app=kube-proxy

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS                IMAGES                                             SELECTOR
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           29h   calico-kube-controllers   127.0.0.1:5000/calico/kube-controllers:v3.28.0   k8s-app=calico-kube-controllers
kube-system   deployment.apps/coredns                   2/2     2            2           29h   coredns                   127.0.0.1:5000/coredns:v1.11.1                   k8s-app=kube-dns

NAMESPACE     NAME                                                DESIRED   CURRENT   READY   AGE   CONTAINERS                IMAGES                                             SELECTOR
kube-system   replicaset.apps/calico-kube-controllers-d864bb84b   1         1         1       29h   calico-kube-controllers   127.0.0.1:5000/calico/kube-controllers:v3.28.0   k8s-app=calico-kube-controllers,pod-template-hash=d864bb84b
kube-system   replicaset.apps/coredns-88f77f6b5                   2         2         2       29h   coredns                   127.0.0.1:5000/coredns:v1.11.1                   k8s-app=kube-dns,pod-template-hash=88f77f6b5
```

#### 1.3 Start a Container Registry
Check whether a local container registry is available:
```shell
nerdctl ps -a | grep registry
nerdctl start <instance-id>
nerdctl ps
```

If a container such as `registry:2` is present, a local registry is running (port 5000 by default). Otherwise, start one locally with nerdctl:
```shell
nerdctl run -d --name registry -p 5000:5000 -v /tmp/registry:/var/lib/registry registry:2
nerdctl start registry
```

Verify that the registry is running:
```text
k8s_deploy# nerdctl ps -a
CONTAINER ID    IMAGE                           COMMAND                   CREATED        STATUS    PORTS    NAMES
98e7e3607d31    docker.io/library/registry:2    "/entrypoint.sh /etc…"    13 days ago    Up                 registry
```

### 2. Prepare Configuration, Log Directories, Packages, and Images
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
First copy configuration templates into the `config` directory. You can obtain them by installing the run package:
```shell
bash memcache_hybrid-1.0.0_linux_aarch64.run  # Replace with the actual path and filename
cp -rf /usr/local/memcache_hybrid/latest/config/* /home/meta/config
```

Make the following configuration changes:
1. Enable high availability.

   mmc-meta.conf
   ```ini
   ock.mmc.meta.ha.enable = true
   ```

2. Configure TLS certificates. If TLS is not used, the switches can be disabled (disabling TLS may create security risks and is not recommended in production).

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

3. Set the data-transfer protocol. This example uses `host_rdma`; use `show_gids` to find the RDMA interface IP and update the IP in `hcom_url`.

   mmc-local.conf
   ```ini
   ock.mmc.local_service.hcom_url = tcp://X.X.X.X:7000
   ```

4. Set `world_size` to match the number of LocalService processes to start.
   
   mmc-local.conf
   ```ini
   ock.mmc.local_service.world_size = 16
   ```

5. Set the MetaService log path to a directory on mounted storage so service logs can be viewed directly from the container.

   mmc-meta.conf
   ```ini
   ock.mmc.log_path = /home/meta/logs
   ```

#### 2.2 Dependency Library

When using the `host_rdma` data-transfer protocol, copy `libhcom.so` to `/home/meta/lib`.
See the [hcom project](https://atomgit.com/openeuler/ubs-comm/tree/br_dev_container_20260228) for the latest shared library.

#### 2.3 Packages
 
This deployment depends on [MemCache](https://gitcode.com/Ascend/memcache) and [MemFabric](https://gitcode.com/Ascend/memfabric_hybrid).
Build the installation packages according to the official instructions and place them in `/home/meta`. The Pod startup process automatically installs packages from that directory.

#### 2.4 Prepare the Image

##### 2.4.1 Prepare an Image That Can Run MemCache
Use an image whose operating-system environment matches the build and installation environment, and mount the `/home/meta` directory.
The official vllm_ascend [Dockerfile](https://github.com/vllm-project/vllm-ascend/blob/main/Dockerfile) is recommended for building the image.

##### 2.4.2 Import the Docker Image into nerdctl
```shell
# List available images
nerdctl images

# Export a Docker-format image and load it into nerdctl
docker save my-image:latest -o my-image.tar
nerdctl load -i my-image.tar
```

##### 2.4.3 Push the Image to the Registry
```shell
# Push the image
nerdctl push my-image:latest
# List images in the registry
curl http://127.0.0.1:5000/v2/_catalog
# List image tags in the registry
curl http://127.0.0.1:5000/v2/<image-name>/tags/list
```

### 3. Start the High-Availability Service Cluster

Before testing, delete any previously created resources:
```shell
kubectl delete ns ns-memcache
```
The repository's `/test/k8s_deploy` directory contains sample YAML files for verifying active-standby high availability.
Edit `meta-pods-demo.yaml` and `local-pods-demo.yaml`, setting **containers:image** and **InitContarners:image** to the image pushed in step 2.

**Note: Files under `./test` are examples only and may have security risks. Do not use them directly in production. Adapt them to your environment before use.**

```shell
# 1. Create a namespace, service account, and role, then bind and authorize the account and role
kubectl apply -f account-role-demo.yaml

# 2. Create the Lease
kubectl apply -f meta-lease-lock-demo.yaml

# 3. Create the ClusterIP Service; its targetPort is the MetaService access port
kubectl apply -f meta-cluster-ip-demo.yaml

# 4. Create the MetaService service
kubectl apply -f meta-pods-demo.yaml
```

Find the cluster IP. In this example, it is `x.x.x.x`:
```shell
k8s_deploy# k get service -n ns-memcache
NAME                          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)               AGE
service-cluster-ip-memcache   ClusterIP   x.x.x.x   <none>        18080/TCP,18090/TCP   24s
```
Then update `/home/meta/config/mmc-local.conf`:
```ini
ock.mmc.meta_service_url = tcp://x.x.x.x:18080
ock.mmc.local_service.config_store_url = tcp://x.x.x.x:18090
```

Create the LocalService clients:
```shell
kubectl apply -f local-pods-demo.yaml
```

Finally, verify that everything started successfully:
```shell
k8s_deploy# kubectl get pod -n ns-memcache
NAME                  READY   STATUS    RESTARTS   AGE
local-service-pod-0   1/1     Running   0          13s
local-service-pod-1   1/1     Running   0          11s
meta-service-pod-0    1/1     Running   0          164m
meta-service-pod-1    1/1     Running   0          164m
```

Notes:
* In `mmc-meta.conf`, `meta_service_url` is `<PodIP:targetPort>`.
* In `mmc-local.conf`, `meta_service_url` is `<ClusterIP:Port>`.
* For convenience, the sample YAML files start Pods with host networking. Change this for production, while ensuring client Pod IPs can reach each other.
* All Kubernetes resources must be in the same namespace.

### 4. Check Service Status and Verify Leader Election

#### 4.1 Check Service Status

After creation, inspect the Pods in the cluster:
```
# List all Pods
kubectl get pods -A -o wide 

# List Pods in the specified namespace
kubectl get pods -n ns-memcache -o wide 
```

After starting the Pods, inspect the MetaService Pods and use the `role=master` label to identify the active and standby nodes:
```
Inspect meta-service-pod-0:
kubectl describe pod meta-service-pod-0 -n ns-memcache

Inspect meta-service-pod-1:
kubectl describe pod meta-service-pod-1 -n ns-memcache
```

Inspect the ClusterIP Service. Its Endpoints show the active service Pod IP and port:
```
kubectl describe service service-cluster-ip-memcache -n ns-memcache
```

Inspect the Lease (Holder identifies the active node. You can also change the lease expiration time by modifying `LeaseDurationSeconds`.)
```
kubectl describe lease lease-memcache -n ns-memcache
```

#### 4.2 Verify That MemCache Works
```shell
# Here, local-service-pod-0 is the Pod name and local-service is the container name.
# Enter a running LocalService container.
kubectl exec -it local-service-pod-0 -n ns-memcache -c local-service -- bash

source /usr/local/Ascend/ascend-toolkit/set_env.sh;
source /usr/local/memfabric_hybrid/set_env.sh;
source /usr/local/memcache_hybrid/set_env.sh;
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/meta/lib
cd /usr/local/memcache_hybrid/latest/aarch64-linux/script/k8s_deploy;
# Run an interactive test and enter put/get/remove commands to verify functionality.
python interactive_app.py 
```

#### 4.3 Verify Active-Standby High Availability
```shell
# Get the leader Pod
kubectl get leases -n ns-memcache 
# Run on the leader node to inspect the log
cat /home/meta/logs/logs/mmc-meta.log
# Delete the leader and check whether the Lease changes and the service remains functional.
kubectl delete pod meta-service-pod-0 -n ns-memcache
```

Kubernetes automatically starts a replacement Pod after one is deleted to maintain the desired Pod count.
Therefore, deleting the leader Pod causes Kubernetes to recreate it. If the leader does not recover before the lease expires, a standby node competes to become the new leader.

Assume `meta-service-pod-0` is the current leader. Delete leader Pod 0:
`kubectl delete pod meta-service-pod-0 -n ns-memcache`

Use `kubectl describe pod meta-service-pod-1 -n ns-memcache` to check whether Pod 1 became the leader.
Also check whether the ClusterIP Endpoints and Lease details changed accordingly.


### 5. FAQ

#### 5.1 Troubleshooting
View the local-service-pod-0 log:
`kubectl logs local-service-pod-0 -n ns-memcache `

```shell
 kubectl get events -n default --sort-by=.metadata.creationTimestamp -A
 kubectl -n ns-memcache logs local-service-pod-0 -c change-pod-ip --previous
```
To enter a Pod container:
```
kubectl exec -it local-service-pod-0 -n ns-memcache -c local-service -- bash
kubectl exec -it local-service-pod-1 -n ns-memcache -c local-service -- bash
kubectl exec -it meta-service-pod-0 -n ns-memcache -c meta-service -- bash
Here, local-service-pod-0 is the Pod name and local-service is the container name.
```

**Recommendation:** When starting container services, write service logs to a host or shared directory to make monitoring and troubleshooting easier. Mount the host directory into the Pod.
<br>The sample YAML already mounts the home directory inside the container, so you can:
<br>Set the MetaService log path to `/home/memcache` to write its log to `/home/memcache/logs/mmc-meta.log`.
<br>Redirect LocalService test output to `/home/memcache/logs/mmc-local.log`.<br>
(Note the service startup command in `local-pods-demo.yaml`:
`exec python3 /usr/local/memfabric_hybrid/latest/aarch64-linux/script/ha/test-mmc-meta-ha.py > /home/memcache/logs/mmc-local.log 2>&1`）


### 6. Notes

1. Follow the official Kubernetes documentation to harden the cluster.
