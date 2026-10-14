# Control Plane Components

## Introduction

The control plane is the brain of your Kubernetes cluster. It makes all the decisions about what runs where, detects failures and responds to them, and maintains the complete desired state of your infrastructure. Without the control plane, worker nodes would have no instructions and your cluster would do nothing.

The control plane runs four core components: **API Server**, **etcd**, **Scheduler**, and **Controller Manager**. This file goes deep into each one — what it does, how it works, and how to observe it in a running cluster.

---

## Component 1: API Server (`kube-apiserver`)

### What It Is

The API Server is the front door of Kubernetes. Every single operation — whether from `kubectl`, from a controller, from the scheduler, or from a kubelet on a node — goes through the API Server. It is the only component that directly reads from and writes to etcd.

### What It Does

The API Server does four things for every request it receives:

**1. Authentication** — Who are you?
It verifies the identity of the requester using certificates, bearer tokens, or IAM (in EKS). An unauthenticated request is immediately rejected.

**2. Authorization** — Are you allowed to do this?
After verifying identity, it checks RBAC (Role-Based Access Control) rules to confirm the authenticated user has permission for the requested action on the requested resource.

**3. Admission Control** — Should this request go through?
Admission controllers are plugins that can modify or reject requests. For example, the `LimitRanger` admission controller enforces resource limits. The `PodSecurity` controller rejects Pods that violate security policies.

**4. Validation and Persistence**
After all checks pass, the API Server validates the object schema and writes it to etcd.

### The API Server is Stateless

The API Server itself stores nothing. All state lives in etcd. This means you can run multiple API Server replicas (AWS EKS does this automatically) for high availability — if one API Server dies, others immediately take over.

### How to Interact With It

Every `kubectl` command is an HTTP call to the API Server:

```bash
# kubectl get pods is actually:
# GET https://<api-server-endpoint>/api/v1/namespaces/default/pods

# You can make the same call with curl (using your kubeconfig credentials):
kubectl get pods -v=8

# -v=8 shows the full HTTP request and response — useful for debugging
# Output includes:
# GET https://ABC123.gr7.us-east-1.eks.amazonaws.com/api/v1/namespaces/default/pods
# Response Status: 200 OK
```

```bash
# See all API resources the server supports
kubectl api-resources

# Output (partial):
# NAME                    SHORTNAMES  APIVERSION  NAMESPACED  KIND
# pods                    po          v1          true        Pod
# services                svc         v1          true        Service
# deployments             deploy      apps/v1     true        Deployment
# nodes                               v1          false       Node
# namespaces              ns          v1          false       Namespace

# See all API versions the server supports
kubectl api-versions
```

### API Groups

Kubernetes APIs are organized into groups:

| API Group | Resources |
|-----------|-----------|
| `v1` (core) | Pods, Services, ConfigMaps, Secrets, Namespaces, Nodes |
| `apps/v1` | Deployments, StatefulSets, DaemonSets, ReplicaSets |
| `batch/v1` | Jobs, CronJobs |
| `networking.k8s.io/v1` | Ingress, NetworkPolicy |
| `autoscaling/v2` | HorizontalPodAutoscaler |
| `rbac.authorization.k8s.io/v1` | Roles, ClusterRoles, RoleBindings |

When you write YAML, the `apiVersion` field tells Kubernetes which API group and version to use:

```yaml
apiVersion: apps/v1     # apps group, version v1
kind: Deployment
```

```yaml
apiVersion: v1          # core group (no prefix)
kind: Pod
```

---

## Component 2: etcd

### What It Is

**etcd** is a distributed, consistent key-value store. It is the database of Kubernetes — the only place where the complete state of the cluster is stored.

Every object in your cluster — every Pod, Deployment, Service, ConfigMap, Secret, Node registration — exists as an entry in etcd. When you run `kubectl get pods`, the API server reads from etcd and returns the data.

### Why etcd Matters So Much

If etcd loses data, the cluster loses its memory of what should exist. The control plane would no longer know what Deployments to maintain, what Pods should be running, or how the network should be configured. Worker nodes would continue running whatever containers they have, but the control plane could not make any new decisions.

This is why etcd backup is one of the most critical operational tasks in self-managed Kubernetes. AWS EKS handles etcd backups automatically — this is one of the major benefits of using a managed service.

### The Data Model

etcd stores data as key-value pairs under a hierarchical path structure:

```
/registry/pods/default/my-web-pod
/registry/pods/production/api-server-pod
/registry/deployments/default/web-deployment
/registry/services/endpoints/default/kubernetes
/registry/secrets/default/db-password
/registry/namespaces/production
```

The values are serialized Kubernetes objects (protobuf-encoded). You never interact with etcd directly — always through the API server.

### etcd Consensus — How It Stays Consistent

etcd uses the **Raft consensus algorithm** to stay consistent across multiple etcd instances. For a production cluster, etcd runs as an odd-numbered cluster (3 or 5 nodes):

```
         etcd node 1 (Leader)
        /        |         \
       /         |          \
etcd node 2   etcd node 3   etcd node 4 (Followers)
    (Follower)  (Follower)
```

- Writes go to the leader, which replicates to followers
- A **majority quorum** must acknowledge a write before it is committed
- For 3 nodes: 2 must agree. Tolerates 1 node failure.
- For 5 nodes: 3 must agree. Tolerates 2 node failures.
- If the leader dies, followers elect a new leader automatically (within seconds)

AWS EKS runs etcd across 3 Availability Zones, so the loss of an entire AZ does not bring down etcd.

### Checking etcd Health on EKS

On EKS, you cannot access etcd directly (it is fully managed). But you can check cluster health which reflects etcd status:

```bash
# Check component status (older clusters)
kubectl get componentstatus

# Check that the API server (which depends on etcd) is responsive
kubectl cluster-info

# Check etcd-related events in the kube-system namespace
kubectl get events -n kube-system --sort-by='.lastTimestamp'
```

---

## Component 3: Scheduler (`kube-scheduler`)

### What It Is

The Scheduler watches for new Pods that have no node assigned yet, and picks the best node for each Pod. It does not start the Pod — it only decides where it should run. The kubelet on the chosen node does the actual starting.

### The Scheduling Process — Step by Step

When you create a Pod, here is exactly what happens:

```
You run: kubectl apply -f pod.yaml
              │
              ▼
    API Server validates and saves Pod to etcd
    (Pod status: Pending, nodeName: "")
              │
              ▼
    Scheduler watches etcd (via API server) and sees the unscheduled Pod
              │
              ▼
    Scheduler runs Filtering + Scoring:
      Step 1 - Filter: Remove nodes that CANNOT run this Pod
      Step 2 - Score: Rank the remaining nodes
      Step 3 - Bind: Assign the Pod to the highest-scoring node
              │
              ▼
    Scheduler writes the node assignment to etcd via API server
    (Pod status: Pending, nodeName: "ip-10-0-1-100.ec2.internal")
              │
              ▼
    kubelet on ip-10-0-1-100.ec2.internal sees the Pod is assigned to it
              │
              ▼
    kubelet starts the containers via containerd
    (Pod status: Running)
```

### Filtering — Nodes That Cannot Run the Pod

The scheduler first eliminates nodes that do not meet the Pod's requirements:

```yaml
# A Pod that MUST run on nodes with a GPU
spec:
  containers:
  - name: ml-trainer
    image: ml-trainer:v1
    resources:
      limits:
        nvidia.com/gpu: 1    # Requires a GPU
  
  # NodeSelector: only schedule on nodes labeled with gpu=true
  nodeSelector:
    gpu: "true"

  # Tolerations: allow scheduling on nodes with a taint
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

Filters applied during scheduling:
- **Resource fit** — Does the node have enough CPU and memory for this Pod?
- **NodeSelector** — Does the node have the required labels?
- **Node affinity** — Does the node match the affinity rules?
- **Taints and Tolerations** — Does the Pod tolerate the node's taints?
- **Volume availability** — Can required PersistentVolumes be mounted on this node?
- **Pod affinity/anti-affinity** — Are required or conflicting Pods on this node?

### Scoring — Ranking the Remaining Nodes

After filtering, the scheduler scores each remaining node:

- **Least requested** — Prefer nodes with more free CPU/memory (spread workload evenly)
- **Image locality** — Prefer nodes that already have the container image cached
- **Pod topology spread** — Spread Pods across zones and nodes for availability
- **Affinity weights** — Give preference to nodes matching preferred (not required) affinity rules

The node with the highest total score gets the Pod.

### Pod Affinity and Anti-Affinity

Control which nodes and which other Pods a Pod lands near:

```yaml
spec:
  affinity:
    # Pod anti-affinity: spread web Pods across different nodes
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname   # "hostname" = different physical nodes
    
    # Node affinity: prefer nodes in us-east-1a
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-east-1a
```

`requiredDuringScheduling` = a hard rule (Pod will not schedule if not met)
`preferredDuringScheduling` = a soft rule (scheduler tries, but not required)

---

## Component 4: Controller Manager (`kube-controller-manager`)

### What It Is

The Controller Manager runs multiple controllers in a single binary process. Each controller is a loop that watches the cluster state and works to make actual state match desired state.

Think of each controller as a dedicated robot watching one specific thing:

| Controller | What It Watches | What It Does |
|-----------|-----------------|-------------|
| **Deployment Controller** | Deployments | Ensures the correct number of ReplicaSets exist |
| **ReplicaSet Controller** | ReplicaSets | Ensures the correct number of Pods exist |
| **Node Controller** | Nodes | Marks nodes as unhealthy after 40 seconds of no heartbeat |
| **Job Controller** | Jobs | Creates Pods for Jobs, tracks completion |
| **CronJob Controller** | CronJobs | Creates Jobs on schedule |
| **Namespace Controller** | Namespaces | Cleans up resources when namespace is deleted |
| **ServiceAccount Controller** | ServiceAccounts | Creates default ServiceAccounts in new namespaces |
| **Endpoints Controller** | Services + Pods | Keeps Service endpoints up to date with healthy Pods |

### How a Controller Works — The Deployment Controller in Detail

When you create a Deployment with `replicas: 3`:

```
You apply deployment.yaml (replicas: 3)
              │
              ▼
API Server stores Deployment in etcd
              │
              ▼
Deployment Controller sees the new Deployment
Creates a ReplicaSet with selector matching the Deployment template
              │
              ▼
ReplicaSet Controller sees the new ReplicaSet (replicas: 3, current: 0)
Creates 3 Pod objects in the API Server
              │
              ▼
Scheduler sees 3 unscheduled Pods
Assigns each to a node
              │
              ▼
kubelet on each node sees Pods assigned to it
Starts containers via containerd
              │
              ▼
Pods are Running — actual state matches desired state
```

Now imagine one Pod crashes:

```
Pod crashes → actual running Pods: 2
              │
              ▼
ReplicaSet Controller sees desired: 3, actual: 2
Creates a new Pod object in the API Server
              │
              ▼
Scheduler assigns it to a node
kubelet starts it
              │
              ▼
Pods running: 3 → desired state restored
```

This all happens automatically, usually within seconds. You do not need to do anything.

### The Node Controller — Handling Node Failures

The Node Controller uses a heartbeat mechanism. Every node's kubelet sends a heartbeat to the API Server every few seconds. The Node Controller watches these heartbeats:

```
Normal state: kubelet sends heartbeat every 10 seconds

Node goes offline (network issue, power failure, etc.)
              │
              ▼
40 seconds pass with no heartbeat
              │
              ▼
Node Controller marks Node as "NotReady"
              │
              ▼
5 minutes pass still no heartbeat (configurable)
              │
              ▼
Node Controller evicts all Pods from the dead node
(Sets Pod status to "Unknown", then deletes them)
              │
              ▼
ReplicaSet Controller sees missing Pods
Creates replacement Pods on healthy nodes
```

On AWS EKS with auto-scaling, the dead node is also terminated and replaced by the Auto Scaling Group automatically.

---

## Summary

- The **API Server** is the single entry point for all Kubernetes operations — it authenticates, authorizes, validates, and persists every request to etcd
- **etcd** is the cluster's distributed database — it stores complete cluster state and uses the Raft algorithm to stay consistent across multiple replicas
- The **Scheduler** picks the best node for each unscheduled Pod by first filtering out ineligible nodes, then scoring remaining nodes based on resource availability, topology, and affinity rules
- The **Controller Manager** runs multiple control loops — each loop continuously compares desired state with actual state and takes action to reconcile them
- The Deployment Controller → ReplicaSet Controller → Scheduler → kubelet chain is how a `kubectl apply` eventually becomes running containers
- The Node Controller detects failed nodes after 40 seconds and evicts their Pods so ReplicaSet controllers can reschedule them on healthy nodes
- On AWS EKS, all four control plane components are managed by AWS — you never need to maintain, upgrade, or back them up

In the next file, `03-worker-node-components.md`, you will go deep into the worker node — the kubelet, kube-proxy, and containerd — and understand exactly how containers go from a Pod spec to a running process.
