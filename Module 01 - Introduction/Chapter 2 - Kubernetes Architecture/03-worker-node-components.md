# Worker Node Components

## Introduction

Worker nodes are where your application containers actually run. Every Pod in your cluster lives on a worker node. While the control plane makes decisions, worker nodes do the real work — pulling container images, starting containers, managing network rules, and reporting status back to the control plane.

Each worker node runs exactly three components: **kubelet**, **kube-proxy**, and a **container runtime** (containerd). This file explains each one in depth and shows you how to observe and debug them in a live EKS cluster.

---

## What a Worker Node Looks Like

When you SSH into an EKS worker node (an EC2 instance), here is what you find running:

```bash
# SSH into a worker node (via AWS Systems Manager Session Manager)
aws ssm start-session --target i-0abc123def456789

# Check what Kubernetes components are running as system services
systemctl list-units --type=service | grep -E "kubelet|containerd"

# Output:
# containerd.service    loaded active running containerd container runtime
# kubelet.service       loaded active running kubelet: The Kubernetes Node Agent

# kube-proxy runs as a DaemonSet Pod inside the cluster, not a system service
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Output:
# NAME              READY   STATUS    RESTARTS   AGE
# kube-proxy-4xkqp  1/1     Running   0          5d
# kube-proxy-7bmpv  1/1     Running   0          5d
# kube-proxy-9ztrl  1/1     Running   0          5d
```

---

## Component 1: kubelet

### What It Is

The kubelet is the primary node agent. It runs as a system service on every worker node. Its job is to ensure that the containers described in Pod specifications are running and healthy on that specific node.

The kubelet is the only Kubernetes component that runs directly on the node as a system service (not as a container).

### What the kubelet Does

**Watches for Pod assignments**
The kubelet watches the API Server for Pods assigned to its node. When a new Pod lands on its node (Scheduler sets `nodeName`), the kubelet picks it up and starts working on it.

**Starts containers via the container runtime**
The kubelet does not start containers itself. It tells the container runtime (containerd) to start them using a standard interface called **CRI (Container Runtime Interface)**:

```
kubelet → CRI (gRPC) → containerd → runc → Linux containers
```

**Manages Pod lifecycle**
- Pulls container images if not already cached on the node
- Creates container networking and storage mounts
- Starts containers in the correct order
- Restarts containers that fail (based on the Pod's `restartPolicy`)

**Runs health checks**
The kubelet executes the liveness, readiness, and startup probes you define in your Pod spec. Based on probe results, it restarts unhealthy containers or removes Pods from Service endpoints.

**Reports status back to the API Server**
The kubelet regularly sends the status of all Pods on its node back to the API Server (which stores it in etcd). This is how `kubectl get pods` shows you real-time status.

**Sends heartbeats**
Every 10 seconds, the kubelet sends a heartbeat (Node lease update) to the API Server. The Node Controller uses this to detect dead nodes.

### kubelet Configuration

The kubelet is configured with a file at `/var/lib/kubelet/config.yaml` on EKS nodes:

```bash
# View kubelet configuration on a worker node
cat /var/lib/kubelet/config.yaml
```

```yaml
# Partial kubelet config
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
clusterDNS:
- 172.20.0.10          # CoreDNS service IP — kubelet tells containers to use this DNS
clusterDomain: cluster.local
containerRuntimeEndpoint: unix:///run/containerd/containerd.sock   # socket to containerd
evictionHard:
  memory.available: "100Mi"    # Evict Pods when node memory drops below 100MB
  nodefs.available: "10%"      # Evict Pods when disk drops below 10%
maxPods: 110                   # Maximum Pods this node will run
```

### kubelet and Resource Eviction

When a worker node runs low on memory or disk, the kubelet evicts Pods to recover resources. It evicts Pods in this priority order:

1. **BestEffort** Pods (no resource requests/limits set) — evicted first
2. **Burstable** Pods (requests set but below limits) — evicted next
3. **Guaranteed** Pods (requests equal limits) — evicted last

This is why setting resource requests and limits in your Pod spec is critical for production workloads.

```yaml
containers:
- name: web
  image: nginx:1.25
  resources:
    requests:          # Guaranteed minimum — scheduler uses this for placement
      cpu: "250m"      # 0.25 CPU cores
      memory: "256Mi"  # 256 megabytes
    limits:            # Hard maximum — container killed if it exceeds this
      cpu: "500m"
      memory: "512Mi"
```

A Pod with both requests and limits equal (`requests == limits`) gets **Guaranteed** QoS class — the highest priority. It is the last to be evicted in a resource pressure situation.

### Debugging the kubelet

```bash
# Check kubelet status on a node
systemctl status kubelet

# View kubelet logs (most useful for debugging Pod start failures)
journalctl -u kubelet -f

# Recent kubelet logs
journalctl -u kubelet --since "30 minutes ago"

# Common errors you will see in kubelet logs:
# "Failed to pull image" → image name wrong or ECR permissions issue
# "Back-off restarting failed container" → container keeps crashing (check application logs)
# "Evicted" → node ran out of memory or disk
```

---

## Component 2: kube-proxy

### What It Is

kube-proxy runs on every worker node and manages the network rules that allow Pods and Services to communicate. It implements the Kubernetes **Service** abstraction at the network level.

On EKS, kube-proxy runs as a **DaemonSet** — Kubernetes automatically runs one kube-proxy Pod on every worker node.

### The Problem kube-proxy Solves

When you create a Service in Kubernetes, the Service gets a stable virtual IP address called a **ClusterIP**. But this IP does not belong to any real machine — it only exists in kube-proxy's network rules.

```bash
# Create a Service
kubectl apply -f service.yaml

# Check its ClusterIP
kubectl get service web-service

# Output:
# NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# web-service   ClusterIP   172.20.45.123   <none>        80/TCP    2m
```

`172.20.45.123` is a virtual IP. When any Pod sends traffic to this IP, kube-proxy intercepts it and routes it to one of the healthy Pods backing the Service. No traffic actually goes to `172.20.45.123` — kube-proxy rewrites the destination IP to a real Pod IP using iptables or IPVS rules.

### How kube-proxy Works — iptables Mode

The default kube-proxy mode on EKS uses iptables. When you create a Service with 3 Pod endpoints, kube-proxy creates iptables rules like this:

```
Traffic to 172.20.45.123:80
         │
         ▼
iptables rule: randomly select one of:
  - 10.0.1.5:8080  (Pod 1 on node 1)   — 33% chance
  - 10.0.1.6:8080  (Pod 2 on node 2)   — 33% chance
  - 10.0.2.7:8080  (Pod 3 on node 3)   — 33% chance
         │
         ▼
Traffic delivered to selected Pod
```

When a Pod becomes unhealthy (fails readiness probe), the Endpoints Controller removes it from the Service endpoints list, and kube-proxy updates its iptables rules — traffic no longer goes to that Pod.

### How kube-proxy Works — IPVS Mode (High Performance)

For large clusters with thousands of Services, iptables can be slow because it does linear rule traversal. IPVS (IP Virtual Server) mode uses a hash table for O(1) lookups:

```bash
# Check which mode kube-proxy is using
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# To use IPVS mode (requires kernel module):
# Edit the kube-proxy ConfigMap to set mode: "ipvs"
```

### What kube-proxy Handles

| Traffic Type | kube-proxy Role |
|-------------|----------------|
| **Pod to Pod** (same node) | Not involved — handled by CNI directly |
| **Pod to Pod** (different node) | Not involved — handled by CNI (overlay network) |
| **Pod to Service** | Routes traffic to a healthy Pod endpoint |
| **External to NodePort** | Forwards external traffic on node port to Service |
| **External to LoadBalancer** | Works with cloud LB to route external traffic |

### Checking kube-proxy

```bash
# Check kube-proxy pods are running on all nodes
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide

# Output:
# NAME              READY   STATUS    NODE
# kube-proxy-4xkqp  1/1     Running   ip-10-0-1-100.ec2.internal
# kube-proxy-7bmpv  1/1     Running   ip-10-0-1-101.ec2.internal

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# On a node: see the iptables rules kube-proxy creates
iptables -t nat -L KUBE-SERVICES
```

---

## Component 3: Container Runtime (containerd)

### What It Is

The container runtime is the software that actually creates and runs containers on the node. The kubelet does not start containers directly — it sends instructions to the container runtime via the **CRI (Container Runtime Interface)** using gRPC.

The standard container runtime in modern Kubernetes (1.24+) is **containerd**.

### Why Not Docker?

Docker was the original container runtime in Kubernetes. But Docker is a large suite of tools (Docker CLI, Docker daemon, Docker API, Docker Compose, containerd) — and Kubernetes only needs the container runtime part, not the rest.

Kubernetes removed Docker as a supported runtime in v1.24 and moved to containerd directly. containerd was already the runtime inside Docker — so nothing actually changed for containers. Docker images still run perfectly on containerd. The Docker image format is the OCI (Open Container Initiative) standard and is supported by all runtimes.

```
Before Kubernetes 1.24:
kubectl → kubelet → dockershim → Docker daemon → containerd → runc → container

After Kubernetes 1.24:
kubectl → kubelet → CRI → containerd → runc → container
```

The Docker daemon is simply removed from the chain. Containers work identically.

### What containerd Does

When the kubelet tells containerd to start a container, containerd:

1. **Pulls the image** from the container registry (ECR, Docker Hub, etc.)
2. **Unpacks the image** into a filesystem snapshot
3. **Configures namespaces** — network, PID, mount, UTS (isolates the container from the host)
4. **Configures cgroups** — enforces CPU and memory limits
5. **Runs the container process** via **runc** (the low-level container runner)

### Interacting with containerd Directly

On EKS worker nodes, you can interact with containerd using the `crictl` tool (the CRI command-line tool):

```bash
# List all running containers on the node (not kubectl — this is at the node level)
crictl ps

# Output:
# CONTAINER      IMAGE          CREATED         STATE     NAME
# a1b2c3d4e5f6   abc123def456   5 minutes ago   Running   web
# b2c3d4e5f6a1   def456abc123   5 minutes ago   Running   nginx

# List all images cached on this node
crictl images

# Inspect a specific container
crictl inspect a1b2c3d4e5f6

# Pull an image manually
crictl pull 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
```

`crictl` is most useful when a Pod is stuck in `ContainerCreating` state — you can see exactly what containerd is doing at the node level.

### Image Caching and Pull Policies

containerd caches pulled images on the node disk. When a new Pod needs the same image, containerd uses the cached version instead of pulling from ECR again. This is why scheduling Pods to nodes that already have the image is faster.

Control image pull behavior with `imagePullPolicy`:

```yaml
containers:
- name: web
  image: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
  imagePullPolicy: IfNotPresent   # Use cached image if available (recommended for tagged images)
  # imagePullPolicy: Always       # Always pull from registry (use with 'latest' tag)
  # imagePullPolicy: Never        # Never pull — fail if not cached (air-gapped environments)
```

> ⚠️ **Warning:** Never use `imagePullPolicy: Always` in production with high Pod churn — it creates unnecessary load on your container registry and slows down Pod starts.

---

## The Full Pod Start-Up Flow

Putting all three components together, here is the exact sequence from `kubectl apply` to a running container:

```
Step 1: You run kubectl apply -f pod.yaml
              │
              ▼
Step 2: API Server receives the request
        Authenticates, authorizes, validates
        Writes Pod object to etcd (status: Pending, no node)
              │
              ▼
Step 3: Scheduler sees unscheduled Pod
        Filters and scores nodes
        Writes node assignment to API Server → etcd
        (status: Pending, nodeName: ip-10-0-1-100.ec2.internal)
              │
              ▼
Step 4: kubelet on ip-10-0-1-100 sees the Pod assigned to it
        Reads the full Pod spec from API Server
              │
              ▼
Step 5: kubelet tells containerd via CRI:
        "Start container with image X, ports Y, resources Z"
              │
              ▼
Step 6: containerd pulls image from ECR (if not cached)
        Sets up container filesystem, namespace, cgroups
        Starts container via runc
              │
              ▼
Step 7: kubelet starts running health probes
        Reports Pod status: Running to API Server
              │
              ▼
Step 8: kube-proxy sees the Pod's IP added to Service endpoints
        Updates iptables rules — Pod is now receiving traffic

Total time from kubectl apply to serving traffic: typically 5–30 seconds
```

---

## Common Worker Node Debugging Commands

```bash
# Check node resource usage (requires metrics-server installed)
kubectl top nodes

# Output:
# NAME                         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# ip-10-0-1-100.ec2.internal   234m         11%    2150Mi          67%

# Describe a node to see conditions and events
kubectl describe node ip-10-0-1-100.ec2.internal

# Look for these conditions in the output:
# MemoryPressure: False   → node is not low on memory
# DiskPressure: False     → node has adequate disk
# PIDPressure: False      → node is not running out of process IDs
# Ready: True             → kubelet is healthy and connected

# See all Pods running on a specific node
kubectl get pods --all-namespaces --field-selector spec.nodeName=ip-10-0-1-100.ec2.internal

# Check node events (recent issues)
kubectl get events --field-selector involvedObject.name=ip-10-0-1-100.ec2.internal
```

---

## Summary

- Worker nodes run three components: **kubelet** (node agent), **kube-proxy** (network rules), and **containerd** (container runtime)
- The **kubelet** watches for Pods assigned to its node, starts them via containerd, runs health probes, and reports status back to the API Server every few seconds
- kubelet evicts Pods under resource pressure — BestEffort Pods are evicted first, Guaranteed Pods last — which is why setting resource requests and limits is critical
- **kube-proxy** creates iptables or IPVS rules that implement the Service abstraction — routing traffic from ClusterIP to actual Pod IPs
- **containerd** pulls images, sets up container isolation (namespaces, cgroups), and runs container processes via runc — it replaced Docker as the runtime in Kubernetes 1.24+
- The pod start sequence is: API Server → etcd → Scheduler → kubelet → containerd → runc → running container
- Debug node-level issues with `crictl` (containerd), `journalctl -u kubelet`, and `kubectl describe node`

In the next file, `04-etcd-overview.md`, you will go deep into etcd — how it stores cluster state, the Raft consensus algorithm, backup strategies, and why it is the single most critical component to protect in a self-managed cluster.
