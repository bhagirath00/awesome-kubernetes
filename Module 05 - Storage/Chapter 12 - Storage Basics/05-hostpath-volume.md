# hostPath Volume

## Introduction

A `hostPath` volume mounts a file or directory from the **worker node's filesystem** directly into a Pod. The data exists on a specific node's disk — it survives Pod restarts but is permanently tied to that one node. If the Pod is scheduled on a different node, it sees a completely different (or non-existent) directory.

`hostPath` is a powerful but dangerous volume type. Used correctly, it enables system-level operations that no other volume type can provide. Used incorrectly, it creates security vulnerabilities and operational nightmares.

---

## How hostPath Works

```
Worker Node (ip-10-0-1-100)
├── /var/log/pods/               ← node's actual directory
├── /var/run/docker.sock         ← node's Docker socket
├── /etc/kubernetes/             ← node's kubeconfig
└── /proc/                       ← kernel process info

Pod scheduled on this node
└── Container has hostPath volume mounted
    └── /var/log inside container → /var/log/pods on the node
        (reads the actual node files)
```

```yaml
spec:
  volumes:
  - name: node-log-dir
    hostPath:
      path: /var/log/pods
      type: Directory

  containers:
  - name: log-reader
    image: busybox
    volumeMounts:
    - name: node-log-dir
      mountPath: /var/log/pods   # same path — reads real node logs
```

---

## hostPath `type` Field

The `type` field controls what Kubernetes checks before mounting:

```yaml
volumes:
- name: my-hostpath
  hostPath:
    path: /some/path
    type: ""                # no checks — mount anything
```

| Type | What It Requires | What Happens If Path Doesn't Exist |
|------|-----------------|-----------------------------------|
| `""` (empty) | Nothing | Nothing — whatever is at the path |
| `Directory` | An existing directory | Pod fails to start |
| `DirectoryOrCreate` | — | Creates the directory (mode 0755) |
| `File` | An existing file | Pod fails to start |
| `FileOrCreate` | — | Creates an empty file (mode 0644) |
| `Socket` | An existing Unix socket | Pod fails to start |
| `CharDevice` | An existing character device | Pod fails to start |
| `BlockDevice` | An existing block device | Pod fails to start |

Always specify an explicit `type` — never use `""` in production because it silently ignores missing paths.

---

## Why hostPath is Dangerous

Using `hostPath` gives a Pod access to the node's filesystem — with implications:

**Security risk — privilege escalation:**
```yaml
# A malicious or compromised Pod could:
hostPath:
  path: /etc                  # read /etc/passwd, /etc/shadow
  type: Directory

hostPath:
  path: /var/run/docker.sock  # control the Docker daemon → root on the node
  type: Socket

hostPath:
  path: /                     # access the ENTIRE node filesystem
  type: Directory
```

A container with access to the Docker socket can start privileged containers — effectively gaining root access to the node and potentially the entire cluster.

**Operational risk — node-specific data:**
```
Pod uses hostPath: /data
Pod runs on Node A → writes data to /data on Node A
Pod is rescheduled to Node B → /data on Node B is EMPTY or different
Application sees no data → production outage
```

Any application restart or node failure can reschedule the Pod to a different node, making `hostPath` data inaccessible.

---

## When hostPath IS Appropriate

`hostPath` is the right choice for **system-level components** that genuinely need access to node resources:

### DaemonSet: Node-Level Log Collection

```yaml
# Fluentd DaemonSet collects logs from all Pods on the node
spec:
  containers:
  - name: fluentd
    image: fluentd:v1.16
    volumeMounts:
    - name: varlog
      mountPath: /var/log
      readOnly: true               # read-only — log collection only
    - name: varlibdockercontainers
      mountPath: /var/lib/docker/containers
      readOnly: true
  volumes:
  - name: varlog
    hostPath:
      path: /var/log
      type: Directory
  - name: varlibdockercontainers
    hostPath:
      path: /var/lib/docker/containers
      type: Directory
```

The log collector runs on every node (DaemonSet) and needs to read the actual log files that containerd writes to the node's disk. No other volume type can provide this access.

### DaemonSet: Node Monitoring Agent

```yaml
# Prometheus Node Exporter — monitors node-level metrics
spec:
  containers:
  - name: node-exporter
    image: prom/node-exporter:v1.6.0
    volumeMounts:
    - name: proc
      mountPath: /host/proc
      readOnly: true
    - name: sys
      mountPath: /host/sys
      readOnly: true
    - name: rootfs
      mountPath: /rootfs
      readOnly: true
    args:
    - '--path.procfs=/host/proc'
    - '--path.sysfs=/host/sys'
    - '--path.rootfs=/rootfs'
  volumes:
  - name: proc
    hostPath:
      path: /proc
      type: Directory
  - name: sys
    hostPath:
      path: /sys
      type: Directory
  - name: rootfs
    hostPath:
      path: /
      type: Directory
```

Node Exporter needs `/proc` and `/sys` to read CPU, memory, and disk metrics from the Linux kernel. It mounts them read-only.

### CNI Plugin Access (Advanced)

CNI plugins (like AWS VPC CNI's `aws-node` DaemonSet) need access to the host network namespace and CNI configuration directory:

```yaml
volumes:
- name: cni-bin-dir
  hostPath:
    path: /opt/cni/bin
    type: DirectoryOrCreate   # create if not exists during CNI setup
- name: cni-net-dir
  hostPath:
    path: /etc/cni/net.d
    type: DirectoryOrCreate
```

---

## Security Controls for hostPath

When you must use `hostPath`, apply these security controls:

### 1. Always Use `readOnly: true` When Possible

```yaml
volumeMounts:
- name: host-logs
  mountPath: /var/log
  readOnly: true    # cannot write to node filesystem — essential for log readers
```

### 2. Restrict with PodSecurity Admission

In Kubernetes 1.23+, PodSecurity admission policies restrict hostPath usage:

```yaml
# PodSecurity namespace label — restrict hostPath to specific paths
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    pod-security.kubernetes.io/enforce: "privileged"   # only for monitoring tools
    pod-security.kubernetes.io/warn: "restricted"
```

Standard pods cannot use `hostPath` under the `restricted` or `baseline` policy.

### 3. Use NodeSelector or Affinity with hostPath

If you use `hostPath` for data (not recommended but sometimes unavoidable), pin the Pod to the same node:

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: ip-10-0-1-100.ec2.internal  # always this node

  volumes:
  - name: data
    hostPath:
      path: /data/my-app
      type: DirectoryOrCreate
```

This prevents rescheduling to a different node — though it also means the Pod cannot run if that node fails.

---

## Verifying hostPath Mounts

```bash
# Check that the node path exists
kubectl debug node/ip-10-0-1-100.ec2.internal -it --image=busybox -- \
  ls /var/log/pods

# Check what's mounted inside the Pod
kubectl exec fluentd-xyz -n kube-system -- df -h

# Check if files are readable
kubectl exec fluentd-xyz -n kube-system -- ls /var/log/
# app.log  kube-apiserver.log  kubelet.log  ...

# Check if it's read-only (try to write — should fail)
kubectl exec fluentd-xyz -n kube-system -- touch /var/log/test
# touch: /var/log/test: Read-only file system   ← correct
```

---

## Summary

- `hostPath` mounts a file or directory from the worker node's filesystem into a Pod — data persists on that node but is tied to it
- Always specify a `type` field — empty type silently ignores missing paths
- `hostPath` is dangerous for application data: security risks (node filesystem access), operational risks (data lost on node change)
- Appropriate uses: DaemonSet log collectors (read node `/var/log`), monitoring agents (read `/proc`, `/sys`), CNI plugins (access `/opt/cni/bin`)
- Always mount `readOnly: true` unless the DaemonSet genuinely needs to write to the node filesystem
- PodSecurity admission restricts `hostPath` usage — `restricted` and `baseline` policies block it; only `privileged` allows it

This completes Chapter 12 — Storage Basics. In Chapter 13, `01-persistent-volumes.md`, you will learn about PersistentVolumes — the cluster-level storage resources that decouple your data from any Pod or node lifecycle.
