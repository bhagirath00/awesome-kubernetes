# Installing Falco in Production via Helm & Modern eBPF

## 1. Architectural Concept & Deep Dive

Falco runs as a **DaemonSet** across all Kubernetes nodes to ensure complete host-level system call coverage. Deploying Falco in enterprise clusters requires:
1. **Modern eBPF Driver**: Bypasses the need to build kernel modules or mount host kernel headers, ensuring stability across AWS Bottlerocket, Ubuntu, and Red Hat Enterprise Linux.
2. **Sidekick Event Routing**: Integrating `falcosidekick` as a deployment to aggregate, filter, and route alerts to webhooks, Slack, or SIEM pipelines without burdening the node daemonsets.
3. **Buffer Tuning**: Configuring kernel-to-userspace ring buffer sizes to prevent packet drops during heavy container write/fork bursts.

```
+─────────────────────────────────────────────────────────────────────────+
|                       FALCO HELM DEPLOYMENT TOPOLOGY                    |
|                                                                         |
|  [ Worker Node 1 ]          [ Worker Node 2 ]       [ Control Plane ]   |
|  +──────────────────────+   +───────────────────+   +─────────────────+ |
|  | Falco DaemonSet Pod  |   | Falco DaemonSet   |   | Falco DaemonSet | |
|  | (modern_ebpf probe)  |   | (modern_ebpf)     |   | (modern_ebpf)   | |
|  +──────────┬───────────+   +─────────┬─────────+   +─────────┬───────+ |
|             │                         │                       │         |
|             └────────────────────┬────┴───────────────────────┘         |
|                                  ▼ (gRPC / HTTP)                        |
|                     +───────────────────────────+                       |
|                     |  Falcosidekick Deployment |                       |
|                     |  (2 Replicas, HA)         |                       |
|                     +─────────────┬─────────────+                       |
|                                   │                                     |
|             ┌─────────────────────┼─────────────────────┐               |
|             ▼                     ▼                     ▼               |
|     Falcosidekick UI            Slack / Email         Datadog / SIEM    |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Production Helm Installation

### Step 1: Add Falco Security Helm Charts
```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

### Step 2: Create Production Values File (`falco-values.yaml`)
```yaml
# 1. Driver Configuration: Modern eBPF
driver:
  kind: modern_ebpf

# 2. DaemonSet Resource Management
resources:
  limits:
    cpu: 1000m
    memory: 1024Mi
  requests:
    cpu: 200m
    memory: 512Mi

# 3. Schedule on all nodes including Control Plane
tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
    operator: Exists
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
    operator: Exists

# 4. Ring Buffer Performance Tuning
falco:
  # Size presets: small (1MB), medium (2MB), large (8MB)
  modern_ebpf:
    cpus_buffer_size: 4194304  # 4MB per CPU core
  json_output: true
  log_stderr: true
  log_level: info

# 5. Enable Falcosidekick & UI
falcosidekick:
  enabled: true
  replicaCount: 2
  webui:
    enabled: true
  config:
    slack:
      webhookurl: "https://hooks.slack.com/services/T00/B00/XXXX"
      minimumpriority: "warning"
    datadog:
      apikey: "your-datadog-api-key"
      minimumpriority: "error"
```

### Step 3: Install Falco via Helm
```bash
helm upgrade --install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  -f falco-values.yaml
```

---

## 3. Verifying Driver Loading and DaemonSet Status

Check that all Falco daemonset pods are running and their modern eBPF driver initialized successfully:

```bash
kubectl get pods -n falco -o wide
```
Output:
```
NAME                                 READY   STATUS    RESTARTS   AGE
falco-7b8x9                          2/2     Running   0          3m
falco-9m4k2                          2/2     Running   0          3m
falco-falcosidekick-5d8f99b4-2z7lx   1/1     Running   0          3m
falco-falcosidekick-5d8f99b4-8r4tp   1/1     Running   0          3m
falco-falcosidekick-ui-6f7d4b-q1w2e  1/1     Running   0          3m
```

Inspect logs to confirm eBPF driver initialization:
```bash
kubectl logs -n falco -l app.io/name=falco -c falco --tail=30
```
```
{"time":"2026-10-14T09:30:15.102Z","level":"info","msg":"Falco initialized with version 0.37.1"}
{"time":"2026-10-14T09:30:15.105Z","level":"info","msg":"Loading modern eBPF probe"}
{"time":"2026-10-14T09:30:15.210Z","level":"info","msg":"eBPF ring buffer allocated successfully"}
{"time":"2026-10-14T09:30:15.420Z","level":"info","msg":"Loaded rules file: /etc/falco/falco_rules.yaml"}
{"time":"2026-10-14T09:30:15.450Z","level":"info","msg":"Starting engine loop"}
```

---

## 4. Testing Falco Detection with a Synthetic Attack

To verify that Falco is actively monitoring and triggering alerts, simulate a real-world adversarial action: spawning an interactive shell inside a running container.

### Step 1: Launch a Test Pod
```bash
kubectl run falco-test --image=alpine -- sleep 3600
```

### Step 2: Trigger the Malicious Event (`kubectl exec`)
```bash
kubectl exec -it falco-test -- sh
```

### Step 3: Inspect Falco Real-Time Logs
In a separate terminal, stream Falco logs:
```bash
kubectl logs -n falco -l app.io/name=falco -c falco --tail=10 | jq .
```

Output:
```json
{
  "hostname": "worker-node-1",
  "output": "09:32:45.120401201: Notice A shell was spawned in a container with an attached terminal (evt_type=execve user=root user_uid=0 user_loginuid=-1 process=sh proc_exepath=/bin/busybox parent=containerd-shim cmdline=sh terminal=34816 container_id=e7b1a402 container_name=falco-test k8s_pod_name=falco-test k8s_ns_name=default image=alpine:latest)",
  "priority": "Notice",
  "rule": "Terminal shell in container",
  "source": "syscall",
  "tags": ["container", "shell", "mitre_execution"]
}
```
**Falco instantly caught the unauthorized interactive shell invocation!**

---

## 5. Verification & Troubleshooting Runbook

### Health Verification Commands
```bash
# Check if Falco is dropping syscall events under heavy load
kubectl logs -n falco -l app.io/name=falco -c falco | grep -i "drops"

# Port-forward Falcosidekick UI for real-time dashboard inspection
kubectl port-forward -n falco svc/falco-falcosidekick-ui 2802:2802
# Access http://localhost:2802 (Default: admin/admin)
```

### Common Pitfalls & Solutions
1. **`falco: failed to load modern eBPF driver`**:
   - Cause: Node kernel version is older than Linux 5.8 or BTF (BPF Type Format) is missing from host kernel.
   - Solution: Upgrade host AMI/kernel, or fall back to `driver.kind=ebpf` with downloaded probe artifacts.
2. **Buffer drops (`n_drops > 0`) during high traffic**:
   - Cause: High container disk/network churn overwhelming the eBPF ring buffer.
   - Solution: Increase `cpus_buffer_size` to 8MB or tune `falco_rules.local.yaml` to exclude known noisy services.

---

## 6. Key Takeaways
- Deploy Falco as a DaemonSet using the modern eBPF driver (`driver.kind=modern_ebpf`) for zero-dependency kernel tracing.
- Include control plane tolerations to ensure full cluster coverage.
- Pair Falco with Falcosidekick to distribute alerts to SIEMs and team chat channels.
- Validate detection using synthetic `kubectl exec` tests to confirm end-to-end alert delivery.
