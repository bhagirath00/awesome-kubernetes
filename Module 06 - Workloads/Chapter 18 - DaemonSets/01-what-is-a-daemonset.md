# What is a DaemonSet

## Introduction

A **DaemonSet** ensures that exactly one copy of a Pod runs on every node in the cluster (or a subset of nodes). When a new node joins the cluster, the DaemonSet automatically schedules a Pod on it. When a node leaves, the Pod is garbage collected.

DaemonSets are the right workload for **node-level system components** — tasks that must run on every node, not just some nodes. Every major Kubernetes cluster feature (log collection, network plugins, monitoring, storage drivers) uses DaemonSets internally.

---

## When to Use a DaemonSet

Use a DaemonSet when:
- The task must run on every node (or a specific subset of nodes)
- The task needs access to node-level resources (`hostPath`, `/proc`, `/sys`, Docker socket)
- The task serves the node itself rather than the cluster workload

**Real examples in every EKS cluster:**

```bash
kubectl get daemonsets --all-namespaces

# NAMESPACE      NAME              DESIRED   CURRENT   READY
# kube-system    aws-node          3         3         3    ← VPC CNI — manages Pod IPs
# kube-system    kube-proxy        3         3         3    ← iptables rules for Services
# kube-system    ebs-csi-node      3         3         3    ← EBS CSI driver node component
# kube-system    nvidia-device-plugin  0     0         0    ← GPU device plugin (if GPU nodes)
# monitoring     node-exporter     3         3         3    ← Prometheus node metrics
# logging        fluentd           3         3         3    ← log collection from nodes
```

Every node in the cluster runs these Pods — automatically.

---

## DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  
  # Update strategy — how to update Pods during a DaemonSet update
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # update 1 node at a time
      maxSurge: 0          # cannot create extra Pods (only 1 per node allowed)
  
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      # Required for system DaemonSets — can tolerate all node conditions
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
        operator: Exists
      - key: node.kubernetes.io/not-ready
        effect: NoExecute
        operator: Exists
      - key: node.kubernetes.io/disk-pressure
        effect: NoSchedule
        operator: Exists
      
      # Run on the host network namespace (for network metrics)
      hostNetwork: true
      hostPID: true          # access host PID namespace (for process metrics)
      
      # ServiceAccount for Prometheus metrics scraping permissions
      serviceAccountName: node-exporter-sa
      
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.7.0
        args:
        - "--path.procfs=/host/proc"
        - "--path.sysfs=/host/sys"
        - "--path.rootfs=/rootfs"
        - "--collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+)($|/)"
        ports:
        - containerPort: 9100
          hostPort: 9100       # bind to the node's port 9100 directly
          name: metrics
        
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "250m"
            memory: "180Mi"
        
        securityContext:
          runAsNonRoot: true
          runAsUser: 65534
          readOnlyRootFilesystem: true
        
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /rootfs
          readOnly: true
      
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```

---

## The `tolerations` Field — Running on All Nodes

By default, Kubernetes applies taints to control-plane nodes and nodes under pressure (disk, memory, not-ready). DaemonSets for system components need to tolerate these taints to run everywhere:

```yaml
tolerations:
# Run on control-plane/master nodes
- key: node-role.kubernetes.io/control-plane
  effect: NoSchedule
  operator: Exists

# Run on nodes that are NotReady (recovering from failure)
- key: node.kubernetes.io/not-ready
  effect: NoExecute
  operator: Exists
  tolerationSeconds: 300   # tolerate NotReady for 5 minutes, then stop

# Run on nodes with disk pressure
- key: node.kubernetes.io/disk-pressure
  effect: NoSchedule
  operator: Exists

# Run on nodes with memory pressure
- key: node.kubernetes.io/memory-pressure
  effect: NoSchedule
  operator: Exists
```

Network plugins (VPC CNI) and node monitors must tolerate everything — they must run even on nodes that are not in a perfect state, because they are part of what makes the node healthy.

---

## Targeting a Subset of Nodes — nodeSelector

To run a DaemonSet only on specific nodes (e.g., only GPU nodes, only nodes in a specific role):

```yaml
spec:
  template:
    spec:
      # nodeSelector — simple label match
      nodeSelector:
        node-type: gpu             # only run on nodes labeled node-type=gpu

      # OR: nodeAffinity — more expressive
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: nvidia.com/gpu
                operator: Exists   # only on nodes that have the GPU label
```

```bash
# Label specific nodes for the DaemonSet
kubectl label node ip-10-0-1-200.ec2.internal node-type=gpu

# The DaemonSet Pod only runs on this node
kubectl get pods -n monitoring -l app=gpu-monitor -o wide
# NAME              READY   NODE
# gpu-monitor-xyz   1/1     ip-10-0-1-200.ec2.internal  ← only on labeled node
```

---

## `hostPort` vs `hostNetwork`

DaemonSet Pods often need to be reachable at a specific port on the node. There are two ways:

**`hostPort`** — bind a specific container port to the node's port:
```yaml
ports:
- containerPort: 9100
  hostPort: 9100    # accessible at <NodeIP>:9100
```

**`hostNetwork: true`** — the Pod shares the node's network namespace entirely:
```yaml
spec:
  hostNetwork: true    # Pod uses node's IP and all node ports
```

Prometheus uses `hostPort: 9100` to scrape Node Exporter from each node's IP address.

---

## Verifying DaemonSet Scheduling

```bash
# Check that all nodes have a DaemonSet Pod
kubectl get daemonset node-exporter -n monitoring

# NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# node-exporter   3         3         3       3            3           <none>

# List DaemonSet Pods and their nodes
kubectl get pods -n monitoring -l app=node-exporter -o wide

# NAME                  READY   NODE
# node-exporter-abc12   1/1     ip-10-0-1-100.ec2.internal
# node-exporter-def34   1/1     ip-10-0-2-100.ec2.internal
# node-exporter-ghi56   1/1     ip-10-0-3-100.ec2.internal

# If DESIRED != CURRENT, check why a node isn't getting a Pod
kubectl describe daemonset node-exporter -n monitoring | grep -A 5 "Events"
# Warning  FailedCreate  timed out waiting for a container to be created
# → check tolerations — likely a taint on the node that the DaemonSet doesn't tolerate
kubectl describe node ip-10-0-1-100.ec2.internal | grep -A 5 "Taints"
```

---

## Summary

- A DaemonSet runs exactly one Pod per node — automatically adding Pods when nodes join and removing them when nodes leave
- Use DaemonSets for node-level system components: log collectors (Fluentd, Fluent Bit), monitoring agents (Node Exporter), network plugins (VPC CNI), storage drivers (EBS CSI node component)
- System DaemonSets must tolerate node taints (`node-role.kubernetes.io/control-plane`, `node.kubernetes.io/not-ready`) to run everywhere
- Use `nodeSelector` or `nodeAffinity` to restrict a DaemonSet to a subset of nodes (e.g., only GPU nodes)
- `hostPort` exposes the DaemonSet Pod at a specific port on each node's IP; `hostNetwork: true` shares the entire node network namespace

In the next file, `02-daemonset-use-cases.md`, you will see the production use cases for DaemonSets in depth — log collection, monitoring, and security agent deployment patterns on EKS.
