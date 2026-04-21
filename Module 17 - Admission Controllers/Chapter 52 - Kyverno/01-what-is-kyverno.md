# What is Kyverno: Kubernetes-Native Policy Engine

## 1. Architectural Concept & Deep Dive

**Kyverno** (Greek for "govern") is a Cloud Native Computing Foundation (CNCF) graduated policy engine designed specifically for Kubernetes. 

While general-purpose policy engines like Open Policy Agent (OPA) require platform engineers to learn a specialized query language (Rego), **Kyverno policies are written purely in Kubernetes YAML**. If you know how to write a Pod or Deployment manifest, you already know how to write Kyverno policies.

Kyverno operates as dynamic admission controllers (`ValidatingAdmissionWebhook` and `MutatingAdmissionWebhook`), intercepting Kubernetes API requests to evaluate compliance, transform payloads, and audit existing resources.

```
+─────────────────────────────────────────────────────────────────────────+
|                           KYVERNO ARCHITECTURE                          |
|                                                                         |
|   kubectl apply -f manifest.yaml                                        |
|                 │                                                       |
|                 ▼                                                       |
|      [ Kubernetes API Server ]                                          |
|                 │                                                       |
|                 ├── MutatingWebhookConfiguration                        |
|                 │          │                                            |
|                 │          ▼                                            |
|                 │   +─────────────────────────────+                     |
|                 │   |  Kyverno Admission Webhook  |                     |
|                 │   |  1. Mutate   (Inject values)|                     |
|                 │   |  2. Verify   (Cosign/Notary)|                     |
|                 │   |  3. Validate (Allow/Block)  |                     |
|                 │   |  4. Generate (Clone sync)   |                     |
|                 │   +──────────────┬──────────────+                     |
|                 │                  │                                    |
|                 ▼                  ▼                                    |
|   [ Admission Decision: ALLOW (200) / DENY (403) / MUTATED ]            |
|                                                                         |
|   Background Operations:                                                |
|   • Kyverno Background Controller ──► Continuous cluster scans          |
|   • Kyverno Reports Controller    ──► Emits PolicyReport CRDs           |
|   • Kyverno Cleanup Controller    ──► Cron-based resource TTL deletion  |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. The Four Functional Pillars of Kyverno

Unlike traditional admission controllers that only validate (pass or fail), Kyverno provides four native primitives:

| Rule Type | Action | Production Use Case |
| :--- | :--- | :--- |
| **1. Validate** | Accepts or rejects resources based on pattern matching or assertions. Can run in `Enforce` or `Audit` mode. | Mandate non-root containers, drop all Linux capabilities, require labels. |
| **2. Mutate** | Dynamically alters incoming YAML payloads before saving to etcd using JSON patch or strategic merge. | Auto-inject default resource requests, add sidecar proxies, inject security contexts. |
| **3. Generate** | Creates new auxiliary resources when a trigger resource is created or updated. | Auto-generate default NetworkPolicies, RoleBindings, or Quotas whenever a Namespace is created. |
| **4. Verify Images** | Cryptographically validates container image signatures, attestations, and SBOMs using Cosign / Sigstore. | Block un-signed supply-chain artifacts from ever running on the cluster. |

---

## 3. Policy Scoping: `ClusterPolicy` vs `Policy`

Kyverno separates policies into two distinct CRD scopes:

1. **`ClusterPolicy`**: Cluster-wide scope. Applies across all namespaces, with optional namespace exclusion or inclusion selectors.
2. **`Policy`**: Namespace-scoped. Governs only the specific namespace where the CRD is deployed, enabling self-service team governance.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-root-user
spec:
  validationFailureAction: Enforce  # Enforce (block) or Audit (report)
  background: true                  # Continuously scan existing resources in etcd
  rules:
    - name: check-run-as-non-root
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Running as root is forbidden. Set securityContext.runAsNonRoot to true."
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
```

---

## 4. Kyverno Background Scanning & Policy Reports

A major limitation of standard webhooks is that they only evaluate requests at *admission time*. If a cluster administrator installs a new policy, existing workloads that violate the policy remain unnoticed.

Kyverno resolves this through **Background Scanning**:
- The `kyverno-background-controller` scans all resources already stored in etcd against active policies.
- Scan results are published to native Kubernetes Custom Resources: **`PolicyReport`** and **`ClusterPolicyReport`** (standardized by the Kubernetes Policy Working Group).

```bash
# View all policy violations across the cluster in real-time
kubectl get clusterpolicyreports -o wide
```
```
NAME                 PASS   FAIL   WARN   ERROR   SKIP   AGE
cpolrep-production    142      3      0       0      8   4d
```

You can query individual failures using standard `kubectl` without parsing complex log streams:
```bash
kubectl get policyreports -n staging -o yaml
```

---

## 5. Architectural Components Breakdown

Kyverno is deployed as high-availability microservices:
1. **`kyverno-admission-controller`**: Receives HTTPS webhook calls from the kube-apiserver, evaluates validation and mutation rules in memory (< 10ms latency).
2. **`kyverno-background-controller`**: Evaluates `generate` rules, reconciles mutating policies on existing resources, and monitors background compliance.
3. **`kyverno-reports-controller`**: Aggregates admission and background audit results and publishes `PolicyReport` CRDs.
4. **`kyverno-cleanup-controller`**: Executes scheduled or TTL-based resource cleanups (e.g., auto-deleting completed pods or ephemeral preview namespaces).

---

## 6. Key Takeaways
- Kyverno is a CNCF graduated, Kubernetes-native policy engine written entirely in YAML.
- It eliminates the steep learning curve of Rego, allowing engineers to write policies using standard Kubernetes resource patterns.
- Supports four primary capabilities: Validate (admit/deny), Mutate (auto-inject), Generate (sync resources), and Verify Images (Cosign validation).
- Features native background auditing that creates standardized `PolicyReport` CRDs for compliance visibility.
