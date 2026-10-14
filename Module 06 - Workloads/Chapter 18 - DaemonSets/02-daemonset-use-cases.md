# DaemonSet Use Cases

## Introduction

DaemonSets run exactly one Pod on every eligible node. This makes them the correct pattern for any component that serves the node itself — collecting its logs, monitoring its resources, enforcing security policies, or managing its network. This file covers the four most important DaemonSet use cases in production EKS clusters.

---

## Use Case 1: Log Collection — Fluent Bit to CloudWatch

Every node writes container logs to `/var/log/containers/` on the host filesystem. A DaemonSet running Fluent Bit on every node reads these logs and ships them to CloudWatch Logs:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: kube-system
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit-sa
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
        operator: Exists
      
      containers:
      - name: fluent-bit
        image: amazon/aws-for-fluent-bit:latest
        resources:
          limits:
            cpu: "200m"
            memory: "200Mi"
          requests:
            cpu: "50m"
            memory: "64Mi"
        env:
        - name: AWS_REGION
          value: "us-east-1"
        - name: LOG_GROUP_NAME
          value: "/eks/my-cluster/containers"
        - name: STREAM_PREFIX
          value: "node-"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
      
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: fluent-bit-config
        configMap:
          name: fluent-bit-config
```

Fluent Bit ConfigMap with CloudWatch output:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: kube-system
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Daemon        Off
        Log_Level     warn

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On

    [FILTER]
        Name               kubernetes
        Match              kube.*
        Kube_URL           https://kubernetes.default.svc:443
        Kube_CA_File       /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File    /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log          On
        Keep_Log           Off

    [OUTPUT]
        Name              cloudwatch_logs
        Match             *
        region            us-east-1
        log_group_name    /eks/my-cluster/containers
        log_stream_prefix ${HOSTNAME}-
        auto_create_group true
```

---

## Use Case 2: Node Monitoring — Prometheus Node Exporter

The Node Exporter DaemonSet exposes node-level hardware and OS metrics (CPU, memory, disk, network) for Prometheus to scrape:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
        prometheus.io/path: "/metrics"
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      tolerations:
      - operator: Exists
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.7.0
        args:
        - "--path.procfs=/host/proc"
        - "--path.sysfs=/host/sys"
        - "--path.rootfs=/rootfs"
        - "--web.listen-address=0.0.0.0:9100"
        ports:
        - containerPort: 9100
          hostPort: 9100
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
        securityContext:
          runAsUser: 65534
          runAsNonRoot: true
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

Key metrics exposed by Node Exporter:
```
node_cpu_seconds_total           → CPU usage per core, per mode (user, system, idle)
node_memory_MemAvailable_bytes   → available memory
node_disk_io_time_seconds_total  → disk I/O utilization
node_network_receive_bytes_total → network traffic received
node_filesystem_avail_bytes      → available disk space per mount
node_load1                       → 1-minute load average
```

---

## Use Case 3: Security Agent — Falco (Runtime Threat Detection)

Falco is a cloud-native security tool that monitors Linux system calls and Kubernetes API activity to detect runtime threats. It runs as a DaemonSet with access to the kernel:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: falco-system
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      tolerations:
      - operator: Exists
      hostNetwork: true
      hostPID: true
      priorityClassName: system-node-critical   # high priority — must run on all nodes
      containers:
      - name: falco
        image: falcosecurity/falco-no-driver:latest
        securityContext:
          privileged: true       # needs kernel-level access for syscall monitoring
        args:
        - /usr/bin/falco
        - --cri /run/containerd/containerd.sock
        - -K /var/run/secrets/kubernetes.io/serviceaccount/token
        - -k https://kubernetes.default.svc.cluster.local
        - --k8s-node=$(FALCO_K8S_NODE_NAME)
        env:
        - name: FALCO_K8S_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: run
          mountPath: /host/run
        - name: containerd-socket
          mountPath: /run/containerd/containerd.sock
      volumes:
      - name: dev
        hostPath:
          path: /dev
      - name: proc
        hostPath:
          path: /proc
      - name: run
        hostPath:
          path: /run
      - name: containerd-socket
        hostPath:
          path: /run/containerd/containerd.sock
          type: Socket
```

Falco alerts on rules like:
- Shell spawned in a container
- Sensitive file opened (e.g., `/etc/shadow`)
- Unexpected network connection from container
- Privilege escalation attempt

---

## Use Case 4: NVIDIA GPU Device Plugin

For ML workloads, the NVIDIA device plugin DaemonSet exposes GPU resources to Pods on GPU nodes:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
      
      nodeSelector:
        accelerator: nvidia-tesla-v100   # only on GPU nodes
      
      priorityClassName: system-node-critical
      containers:
      - name: nvidia-device-plugin-ctr
        image: nvcr.io/nvidia/k8s-device-plugin:v0.14.1
        env:
        - name: FAIL_ON_INIT_ERROR
          value: "false"
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
```

After the NVIDIA device plugin DaemonSet is running on GPU nodes, Pods can request GPU resources:

```yaml
resources:
  limits:
    nvidia.com/gpu: 2    # request 2 GPU cores
```

---

## Summary

- Fluent Bit DaemonSet collects container logs from every node's `/var/log/containers/` and ships to CloudWatch — the standard log collection pattern on EKS
- Node Exporter DaemonSet exposes node hardware metrics (CPU, memory, disk, network) for Prometheus — use `hostPID: true`, `hostNetwork: true`, and `hostPath` mounts for `/proc` and `/sys`
- Falco DaemonSet monitors Linux syscalls for runtime threats — requires `privileged: true` for kernel access
- NVIDIA device plugin DaemonSet exposes GPU resources to the Kubernetes scheduler — only runs on labeled GPU nodes via `nodeSelector`
- All production DaemonSets: tolerate all node conditions, use `priorityClassName: system-node-critical` for system DaemonSets, and set proper resource limits to avoid consuming too much node capacity

In the next file, `03-daemonset-rolling-updates.md`, you will learn how to safely update DaemonSets — the update strategies, verification steps, and how to handle nodes that go offline during an update.
