# Remediating Common CIS Kubernetes Benchmark Failures

## 1. Architectural Concept & Deep Dive

Running automated scans like `kube-bench` is only the diagnostic phase. The true engineering challenge lies in **remediation**—applying fixes without disrupting live workloads, creating cluster lockouts, or degrading control plane availability.

This runbook details step-by-step remediation procedures for the top 10 most critical CIS failures encountered across production environments.

```
+─────────────────────────────────────────────────────────────────────────+
|                    CIS BENCHMARK REMEDIATION FLOW                       |
|                                                                         |
|   1. SCAN (kube-bench output) ──► Identify failing rule ID              |
|                                           │                             |
|                                           ▼                             |
|   2. IMPACT ANALYSIS ───────────► Control Plane vs Worker Node?         |
|                                   Will pod restart be triggered?        |
|                                           │                             |
|                                           ▼                             |
|   3. EXECUTE REMEDIATION ───────► Update YAML / Chmod / Systemctl       |
|                                           │                             |
|                                           ▼                             |
|   4. VERIFY COMPLIANCE ─────────► Re-run kube-bench (Confirm PASS)      |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Top Control Plane Remediation Runbooks

### Issue 1: CIS 1.1.1 - 1.1.8 (Manifest File Permissions)
- **Failure**: Manifest permissions are world-readable (e.g. `0644`).
- **Remediation**:
```bash
# Set strict 600 permissions on all static pod manifests
sudo chmod 600 /etc/kubernetes/manifests/*.yaml

# Ensure root ownership
sudo chown root:root /etc/kubernetes/manifests/*.yaml
```

### Issue 2: CIS 1.2.1 (API Server Anonymous Authentication Enabled)
- **Failure**: `--anonymous-auth=true` or flag omitted.
- **Remediation**:
Edit `/etc/kubernetes/manifests/kube-apiserver.yaml` and set:
```yaml
    - --anonymous-auth=false
```

### Issue 3: CIS 1.2.25 (NodeRestriction Admission Plugin Missing)
- **Failure**: Kubelets can tamper with other nodes' labels.
- **Remediation**:
In `/etc/kubernetes/manifests/kube-apiserver.yaml`, locate `--enable-admission-plugins` and append `NodeRestriction`:
```yaml
    - --enable-admission-plugins=NodeRestriction,PodSecurity,LimitRanger
```

### Issue 4: CIS 1.2.30 (API Audit Log Path Not Set)
- **Failure**: Control plane actions are not logged for forensic audit.
- **Remediation**:
Add audit policy and log path arguments to `/etc/kubernetes/manifests/kube-apiserver.yaml` (as detailed in Chapter 53).

---

## 3. Top etcd Remediation Runbooks

### Issue 5: CIS 1.1.12 (etcd Data Directory Permissions Insecure)
- **Failure**: `/var/lib/etcd` permissions are `0755` or owned by non-root.
- **Remediation**:
```bash
sudo chmod 700 /var/lib/etcd
sudo chown -R root:root /var/lib/etcd
```

### Issue 6: CIS 2.2 (etcd Client Authentication Disabled)
- **Failure**: etcd accepts unauthenticated connections on port 2379.
- **Remediation**:
In `/etc/kubernetes/manifests/etcd.yaml`, verify:
```yaml
    - --client-cert-auth=true
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

---

## 4. Top Worker Node & Kubelet Remediation Runbooks

### Issue 7: CIS 4.2.1 & 4.2.2 (Kubelet Anonymous Auth & Mode AlwaysAllow)
- **Failure**: Kubelet port 10250 accepts unauthenticated commands.
- **Remediation**:
Edit `/var/lib/kubelet/config.yaml` on every worker node:
```yaml
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
```
Reload systemd:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

### Issue 8: CIS 4.2.4 (Kubelet Read-Only Port 10255 Open)
- **Failure**: Kubelet serves unauthenticated metrics over port 10255.
- **Remediation**:
In `/var/lib/kubelet/config.yaml`, set:
```yaml
readOnlyPort: 0
```

---

## 5. Top Workload & Pod Security Remediation Runbooks

### Issue 9: CIS 5.2.1 - 5.2.5 (Privileged Containers and Root Access Allowed)
- **Failure**: Pods run as root or with `privileged: true`.
- **Remediation via Built-in Pod Security Admission (PSA)**:
Label application namespaces to enforce the `restricted` standard:

```bash
# Enforce restricted baseline across application namespace
kubectl label --overwrite namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted
```

When developers attempt to submit a privileged pod:
```
Error from server (Forbidden): pods "rogue" is forbidden: violates PodSecurity "restricted:latest": 
privileged (container "rogue" must not set securityContext.privileged=true), 
allowPrivilegeEscalation != false (container "rogue" must set securityContext.allowPrivilegeEscalation=false)
```

---

## 6. Automated Remediation Bash Script for Worker Nodes

Deploy this script across your worker node fleet via Ansible or cloud-init:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "=== Applying CIS Benchmark Remediations to Node ==="

# 1. Fix Kubelet Configuration Permissions
if [ -f /var/lib/kubelet/config.yaml ]; then
  chmod 600 /var/lib/kubelet/config.yaml
  chown root:root /var/lib/kubelet/config.yaml
fi

# 2. Fix PKI Key Permissions
if [ -d /etc/kubernetes/pki ]; then
  chmod 700 /etc/kubernetes/pki
  find /etc/kubernetes/pki -name "*.key" -exec chmod 600 {} \;
fi

# 3. Patch Kubelet Config to disable anonymous auth and read-only port
KUBELET_CONFIG="/var/lib/kubelet/config.yaml"
if [ -f "$KUBELET_CONFIG" ]; then
  sed -i 's/enabled: true/enabled: false/g' "$KUBELET_CONFIG"
  sed -i 's/readOnlyPort: [0-9]*/readOnlyPort: 0/g' "$KUBELET_CONFIG"
  systemctl restart kubelet
fi

echo "=== Remediation Completed Successfully ==="
```

---

## 7. Key Takeaways
- Remediating static pod manifests causes kubelet to automatically restart the affected control plane component.
- Always verify Kubelet port hardening (`readOnlyPort: 0`, `mode: Webhook`) across all worker nodes.
- Enforce Pod Security Standards (`restricted`) at the namespace level to remediate workload compliance failures at scale.
- Re-run `kube-bench` after remediation to validate that previously failing checks now report `[PASS]`.
