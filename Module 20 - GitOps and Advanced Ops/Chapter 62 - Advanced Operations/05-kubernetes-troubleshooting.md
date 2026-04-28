# Production Kubernetes Troubleshooting & Incident Triage

When production alerts trigger, engineers need a structured, deterministic triage methodology. Guessing causes or restarting pods at random destroys transient diagnostic evidence. 

This guide outlines a comprehensive **60-Second Triage Protocol**, an exhaustive failure decision tree, exit-code forensics, and deep network/node troubleshooting procedures.

---

## 1. The 60-Second Pod Triage Protocol

Follow this linear sequence before touching any production workload:

```
  Step 1: Check Pod Status
          kubectl get pods -n <namespace> -o wide
          |
          +---> Status: Pending? -------------> Check Node capacity, PVCs, Taints
          |
          +---> Status: ImagePullBackOff? ----> Check Registry credentials, tags
          |
          +---> Status: CrashLoopBackOff? ----> Inspect Exit Codes & Application Logs
          |
          +---> Status: Running (0/1 Ready)? -> Check Liveness / Readiness Probes

  Step 2: Inspect Kubernetes Events (Chronological)
          kubectl describe pod <pod-name> -n <namespace>

  Step 3: Check Current & Previous Logs
          kubectl logs <pod-name> -n <namespace> --tail=100
          kubectl logs <pod-name> -n <namespace> --previous   # CRITICAL for crashed pods
```

---

## 2. Pod Lifecycle Troubleshooting Decision Tree

```
                                   +-------------------+
                                   |  Pod is Unhealthy |
                                   +---------+---------+
                                             |
                 +---------------------------+---------------------------+
                 |                                                       |
                 v                                                       v
       [ Phase: PENDING ]                                      [ Phase: RUNNING / WAITING ]
                 |                                                       |
   +-------------+-------------+                           +-------------+-------------+
   |                           |                           |                           |
   v                           v                           v                           v
Scheduling Issues         Volume Issues             Container Crashes            Image Issues
- Insufficient CPU/RAM    - PVC in "Pending"        - OOMKilled (Exit 137)       - ErrImagePull
- NodeSelector mismatch   - StorageClass missing    - App Exception (Exit 1)     - ImagePullBackOff
- Taints/Tolerations      - Multi-Attach error      - Missing ConfigMap/Secret   - Auth failed (401/403)
- Node Affinity rules                                 (CreateContainerConfigError)
```

---

## 3. Deep-Dive: Common Pod Failures & Remediation

### 1. `Pending` State
The scheduler cannot assign the pod to any available worker node.

*Run diagnosis:*
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 10 Events:
```

*Common Causes & Fixes:*
- **`0/12 nodes are available: 12 Insufficient cpu`**: Cluster is out of compute capacity. Scale up node groups or adjust pod `spec.containers[*].resources.requests`.
- **`0/12 nodes are available: 12 node(s) had untolerated taint`**: The available nodes have taints (e.g., `dedicated=database:NoSchedule` or `node-role.kubernetes.io/control-plane`). Add corresponding tolerations to the pod.
- **`pod has unbound immediate PersistentVolumeClaims`**: PVC cannot bind to a PV. Run `kubectl describe pvc <pvc-name>` to check storage class capacity or volume zone constraints.

---

### 2. `ImagePullBackOff` / `ErrImagePull`
The kubelet on the assigned worker node cannot download the specified container image.

*Common Causes & Fixes:*
- **Tag Typo or Missing Image**: Confirm the repository and tag exist in the registry.
- **Private Registry Authentication**: Missing or invalid `imagePullSecrets`.
  ```bash
  kubectl create secret docker-registry regcred \
    --docker-server=https://index.docker.io/v1/ \
    --docker-username=user \
    --docker-password=token \
    --docker-email=user@company.com -n <namespace>
  ```
- **Registry Rate Limiting**: Docker Hub rate limits (429 Too Many Requests). Use an internal mirror (Harbor, AWS ECR Pull Through Cache).

---

### 3. Container Exit Codes & `CrashLoopBackOff`

When a container terminates repeatedly, Kubernetes places it in `CrashLoopBackOff` with exponential backoff delays. Inspect the exact exit code:

```bash
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
```

| Exit Code | Meaning | Typical Root Cause | Actionable Fix |
| :--- | :--- | :--- | :--- |
| **0** | Clean Exit | App finished execution (e.g., batch script, missing foreground daemon) | Ensure entrypoint process stays in foreground; check if meant to be a `Job`. |
| **1** | Application Error | Uncaught exception, syntax error, missing environment variable | Check `kubectl logs <pod-name> --previous`. |
| **137** | **SIGKILL / OOMKilled** | Pod exceeded container `resources.limits.memory` or node Kernel OOM | Increase memory limits or profile memory leaks in application. |
| **139** | Segmentation Fault | Application binary attempted to access unmapped memory | Native C/C++ library bug or incompatible OS base image. |
| **143** | SIGTERM | Graceful termination signal sent by Kubernetes | Pod was evicted or deployment was updated/scaled down. |

---

## 4. Troubleshooting Networking & Service Connectivity

When Pod A cannot communicate with Pod B via its Kubernetes Service name (`backend.prod.svc.cluster.local:8080`), follow the 5-point network triage:

```
  [ Client Pod ] ---> 1. CoreDNS Resolution
                            |
                            v
                      2. Service ClusterIP & Port
                            |
                            v
                      3. Service Endpoints / EndpointSlice
                            |
                            v
                      4. NetworkPolicy Allowed?
                            |
                            v
                      5. Target Pod Listening & Healthy
```

### Network Diagnostic Steps

```bash
# 1. Verify DNS Resolution inside cluster using an ephemeral debug container
kubectl run dns-test --rm -it --image=curlimages/curl -- sh
/ $ nslookup payment-service.production.svc.cluster.local
/ $ curl -v http://payment-service.production.svc.cluster.local:8080/healthz

# 2. Check if the Service actually has registered Endpoints
kubectl get endpoints payment-service -n production
# If ENDPOINTS is <none>, the Service selector does NOT match the Pod labels!

# 3. Compare Service Selector and Pod Labels
kubectl get service payment-service -n production -o jsonpath='{.spec.selector}'
kubectl get pods -n production --show-labels

# 4. Check for blocking NetworkPolicies
kubectl get networkpolicies -n production
```

---

## 5. Troubleshooting Node Failures: `NodeNotReady`

When a node transitions to `NotReady`, the kubelet has stopped reporting heartbeat health to the API server.

```bash
# Check node conditions
kubectl describe node <node-name>
```

Look at the `Conditions` section:

```
Conditions:
  Type             Status  Reason
  ----             ------  ------
  DiskPressure     False   KubeletHasNoDiskPressure
  MemoryPressure   False   KubeletHasSufficientMemory
  PIDPressure      False   KubeletHasSufficientPID
  Ready            False   KubeletNotReady
```

### Common Node Failures:
1. **Container Runtime / Kubelet Dead**:
   SSH into the worker node and inspect systemd logs:
   ```bash
   sudo systemctl status kubelet
   sudo journalctl -u kubelet --no-pager -n 100
   sudo systemctl status containerd
   ```
2. **Disk Pressure (Garbage Collection Failure)**:
   The container runtime disk is over 85% full.
   ```bash
   df -h /var/lib/containerd
   sudo crictl rmi --prune # Clean unused images
   ```
3. **CNI Network Plugin Broken**:
   If CNI pods (Cilium, Calico, AWS-VPC-CNI) fail on the node, the node cannot assign IP addresses to new pods and stays `NotReady`. Check `kubectl get pods -n kube-system -o wide` for crashes on that node.

---

## 6. Live Interactive Debugging with `kubectl debug`

Never install `curl`, `net-tools`, or debug utilities into minimal production distroless images. Use **`kubectl debug`** to inject an ephemeral troubleshooting container into an existing pod:

### Debugging with Shared Process Namespace
```bash
# Attach an ephemeral debug container sharing the target pod's network and process namespace
kubectl debug -it <target-pod-name> -n production \
  --image=nicolaka/netshoot \
  --target=<target-container-name>
```

Inside the `netshoot` container, you have instant access to:
- `tcpdump` for packet analysis.
- `ss -tulpn` to see all listening ports in the target container.
- `curl`, `dig`, `traceroute`, `iperf` for network latency testing.
- Process inspection (`ps aux`) of the target application.

---

## 7. Emergency Triage Quick-Reference

```bash
# Get all non-running pods across the entire cluster
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# Sort recent cluster events to see what broke first
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -n 30

# Restart stuck pods in a deployment gracefully
kubectl rollout restart deployment/<deployment-name> -n <namespace>

# Temporarily cordon and drain a malfunctioning node
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
