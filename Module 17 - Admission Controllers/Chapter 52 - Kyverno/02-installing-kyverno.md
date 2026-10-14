# Installing and Configuring Production Kyverno

## 1. Architectural Concept & Deep Dive

Deploying Kyverno into enterprise production environments requires strict adherence to High-Availability (HA) principles. Because Kyverno registers as both a `MutatingWebhookConfiguration` and `ValidatingWebhookConfiguration` with the Kubernetes API server, a crashed or degraded admission controller can cause cascading control plane timeouts or prevent pods from scheduling.

```
+─────────────────────────────────────────────────────────────────────────+
|                 HIGH-AVAILABILITY KYVERNO DEPLOYMENT                    |
|                                                                         |
|                Kubernetes API Server (Control Plane)                    |
|                                  │                                      |
|                                  ├── Validating/Mutating Webhook        |
|                                  │   (mTLS over Port 9443)              |
|                                  ▼                                      |
|                      [ Service: kyverno-svc ]                           |
|                                  │                                      |
|         ┌────────────────────────┼────────────────────────┐             |
|         ▼                        ▼                        ▼             |
|  [ Pod: kyverno-1 ]       [ Pod: kyverno-2 ]       [ Pod: kyverno-3 ]   |
|  (Node: Worker-A)         (Node: Worker-B)         (Node: Worker-C)     |
|  Anti-Affinity Enforced   Anti-Affinity Enforced   Anti-Affinity Enforced
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Helm Installation: Production Configuration

The recommended and supported method to deploy Kyverno is via the official Helm chart.

### Step 1: Add Kyverno Helm Repository
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```

### Step 2: Create Production Values File (`kyverno-values.yaml`)
```yaml
# Highly Available Replica Counts
admissionController:
  replicas: 3
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/component: admission-controller

  # Production Resource Allocation
  resources:
    limits:
      cpu: 1000m
      memory: 1024Mi
    requests:
      cpu: 200m
      memory: 256Mi

  # Failure Policy: Ignore prevents API lockouts during network partition
  init:
    webhooks:
      failurePolicy: Ignore

backgroundController:
  replicas: 2
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi

reportsController:
  replicas: 2
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi

cleanupController:
  replicas: 1

# Prometheus Metrics & ServiceMonitors
features:
  logging:
    format: json
    verbosity: 2
```

### Step 3: Install via Helm
```bash
helm upgrade --install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  -f kyverno-values.yaml
```

---

## 3. Webhook Failure Policies: `Fail` vs `Ignore`

The `failurePolicy` setting in the webhook configuration determines how the Kubernetes API server reacts if the Kyverno pod cannot be reached:

| Policy | Behavior when Kyverno is Down | Production Rationale |
| :--- | :--- | :--- |
| **`Ignore`** (Recommended Default) | API server bypasses the webhook and **admits the request**. | Prevents catastrophic cluster lockouts where critical pods (DNS, CNI, ingress) cannot start during a node reboot. |
| **`Fail`** (High Security / PCI-DSS)| API server **rejects the request** with an internal error. | Enforces strict zero-trust compliance at the cost of operational availability if Kyverno goes down. |

> **Production Best Practice**: Always start with `failurePolicy: Ignore`. Once your HA deployment has demonstrated zero downtime and low p99 webhook latency (< 15ms), configure `failurePolicy: Fail` only for critical compliance namespaces.

---

## 4. The Kyverno CLI: Shift-Left Testing in CI/CD

One of Kyverno's most powerful features is the standalone **`kyverno` CLI**. It allows developers and DevOps engineers to test and validate policies against Kubernetes manifests inside GitHub Actions or GitLab CI **before manifests are ever applied to a live cluster**:

### Step 1: Install Kyverno CLI
```bash
# On Linux
curl -LO https://github.com/kyverno/kyverno/releases/download/v1.11.4/kyverno-cli_v1.11.4_linux_x86_64.tar.gz
tar -xvf kyverno-cli_v1.11.4_linux_x86_64.tar.gz
sudo mv kyverno /usr/local/bin/
```

### Step 2: Validate Manifests Locally
```bash
# Test a local deployment manifest against a policy file
kyverno apply policy.yaml --resource deployment.yaml
```
Output:
```
Applying 1 policy to 1 resource...

pass: 1, fail: 0, warn: 0, error: 0, skip: 0 
```

If a violation occurs, the CLI exits with return code `1`, failing the CI pipeline:
```
policy require-non-root-user -> resource default/Deployment/web failed:
1. autogen-check-run-as-non-root: Running as root is forbidden!
```

---

## 5. Verification & Troubleshooting Runbook

### Health Verification Commands
```bash
# Check all Kyverno controller pods are Running and Ready
kubectl get pods -n kyverno -o wide

# Verify dynamic webhooks registered with the API server
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations | grep kyverno

# Check Kyverno admission latency metric
kubectl get --raw /metrics | grep kyverno_admission_review_duration_seconds
```

### Common Pitfalls & Solutions
1. **API server returns `x509: certificate signed by unknown authority` on webhook**:
   - Cause: Webhook CA bundle mismatch or cert-manager sync issue.
   - Solution: Restart the Kyverno admission controller pod. Kyverno automatically regenerates its internal TLS certificates and patches the `caBundle` into the webhook configurations.
2. **Kyverno memory consumption scales uncontrollably**:
   - Cause: High frequency background scanning on clusters with tens of thousands of pods.
   - Solution: Adjust background scan interval: set `--background-scan-interval=1h` or disable background scanning on high-churn ephemeral namespaces.
3. **Pods in `kube-system` fail to start**:
   - Cause: Kyverno policy accidentally evaluated against system infrastructure pods.
   - Solution: Always configure `excludedNamespaces: [kube-system, kyverno]` on all `ClusterPolicy` definitions.

---

## 6. Key Takeaways
- Always deploy Kyverno in High-Availability mode (minimum 3 admission controller replicas across zones).
- Use `topologySpreadConstraints` and Pod Anti-Affinity to prevent co-locating replicas on the same node.
- Utilize the Kyverno CLI to validate policies in CI/CD pipelines before deploying manifests to production clusters.
- Protect cluster infrastructure by setting `excludedNamespaces: [kube-system]` on all cluster policies.
