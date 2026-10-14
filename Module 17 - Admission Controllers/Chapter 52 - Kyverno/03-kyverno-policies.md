# Deep Dive: Writing Production Kyverno Policies

## 1. Architectural Concept & Deep Dive

Kyverno policies leverage the natural declarative syntax of Kubernetes. Instead of querying trees in an external language, Kyverno compares YAML blocks directly:

```
+─────────────────────────────────────────────────────────────+
|                 KYVERNO POLICY TYPES & FLOW                 |
|                                                             |
|   1. VALIDATION POLICY                                      |
|      Pattern Match: Compare target fields against overlay   |
|      Condition Match: Evaluate mathematical expressions     |
|                                                             |
|   2. MUTATION POLICY                                        |
|      Strategic Merge Patch: Inject default YAML keys        |
|      JSON 6902 Patch: Replace/Add/Remove precise array paths|
|                                                             |
|   3. GENERATION POLICY                                      |
|      Trigger: On Namespace CREATE                           |
|      Action: Generate NetworkPolicy / Secret / RoleBinding  |
|                                                             |
|   4. IMAGE VERIFICATION                                     |
|      Cryptographic Cosign Verification against public key   |
|      Attestation Check: Verify vulnerability scan passed    |
+─────────────────────────────────────────────────────────────+
```

---

## 2. Validation Policies (Pattern Matching & Foreach)

Validation rules ensure that incoming or existing resources match a specified pattern.

### Example 1: Enforce Non-Root and Read-Only Root Filesystem
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: hardened-pod-security
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: enforce-pod-hardening
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Containers must run as non-root and have read-only root filesystems."
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
            containers:
              - name: "*"
                securityContext:
                  readOnlyRootFilesystem: true
                  allowPrivilegeEscalation: false
                  capabilities:
                    drop:
                      - ALL
```

### Example 2: Foreach Array Evaluation (Check All Containers)
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-host-ports
spec:
  validationFailureAction: Enforce
  rules:
    - name: block-host-ports
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Host ports are forbidden."
        foreach:
          - list: "request.object.spec.containers"
            deny:
              conditions:
                any:
                  - key: "{{ element.ports[].hostPort | length(@) }}"
                    operator: GreaterThan
                    value: 0
```

---

## 3. Mutation Policies (Strategic Merge & JSON Patch)

Mutation policies dynamically transform resources before they are committed to etcd.

### Example 1: Auto-Inject Default Resource Requests via Strategic Merge
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: default-resource-requests
spec:
  rules:
    - name: inject-default-requests
      match:
        any:
          - resources:
              kinds:
                - Pod
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              # Targets all containers missing CPU/Memory requests
              - (name): "*"
                resources:
                  requests:
                    +(cpu): "100m"
                    +(memory): "128Mi"
```
*(Notice the `+(field)` operator: this means **add only if not already present**! Existing custom requests will not be overwritten).*

### Example 2: Safe Sidecar Injection (JSON 6902 Patch)
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: inject-log-forwarder
spec:
  rules:
    - name: add-fluentbit-sidecar
      match:
        any:
          - resources:
              kinds:
                - Deployment
              annotations:
                logging.myorg.io/enabled: "true"
      mutate:
        patchesJson6902: |-
          - op: add
            path: /spec/template/spec/containers/-
            value:
              name: fluentbit-sidecar
              image: fluent/fluent-bit:2.2
              resources:
                limits:
                  memory: 128Mi
                  cpu: 100m
```

---

## 4. Generation Policies (Auto-Provisioning Resources)

Generation rules create new resources in response to lifecycle events.

### Example: Auto-Generate Default NetworkPolicy on New Namespace
Whenever a developer creates a namespace, automatically provision a default-deny NetworkPolicy and sync it:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-default-network-policy
spec:
  rules:
    - name: default-deny-ingress
      match:
        any:
          - resources:
              kinds:
                - Namespace
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny-all
        namespace: "{{request.object.metadata.name}}"
        synchronize: true  # Sync updates from policy to target!
        data:
          spec:
            podSelector: {}
            policyTypes:
              - Ingress
              - Egress
```
- If an unauthorized user attempts to delete or alter this `NetworkPolicy` inside their namespace, **Kyverno automatically reconciles and restores it** because `synchronize: true` is configured.

---

## 5. Image Verification Policies (Cosign & Sigstore)

Securing the software supply chain requires cryptographically validating that container images were built by authorized CI pipelines and have not been tampered with in transit.

### Example: Verify Image Signatures via Public Key
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  rules:
    - name: verify-trusted-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAEh27y0...
                      -----END PUBLIC KEY-----
```
- If a deployment references an image that lacks a valid cryptographic signature matching the public key, the admission controller **blocks the pod immediately**.

---

## 6. Verification & Troubleshooting Runbook

### Testing Policies Live
```bash
# 1. Apply the cluster policy
kubectl apply -f policy.yaml

# 2. Check cluster policy status
kubectl get clusterpolicies

# 3. Attempt a non-compliant deployment
kubectl run rogue --image=nginx:alpine --dry-run=server
```
Expected output:
```
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request: 
policy hardened-pod-security/enforce-pod-hardening: Containers must run as non-root and have read-only root filesystems.
```

### Common Pitfalls & Solutions
1. **Mutation policy creates infinite reconciliation loops**:
   - Cause: A mutation rule triggers an update that re-triggers the mutation webhook.
   - Solution: Use `+(field)` conditional addition, or match specifically on `operation: CREATE`.
2. **`verifyImages` times out on registry query**:
   - Cause: OCI registry is network-firewalled or requires authentication credentials.
   - Solution: Configure image pull secrets on the Kyverno deployment via `imagePullSecrets` or attach an AWS IRSA role with ECR permissions.

---

## 7. Key Takeaways
- Kyverno validate rules use intuitive pattern overlays; `deny` conditions enable mathematical logic.
- The `+(key)` syntax in mutation rules ensures defaults are only injected when values are absent.
- Generation policies with `synchronize: true` guarantee compliance resources cannot be deleted by local namespace admins.
- Image verification validates Cosign signatures at the admission gate, stopping untrusted supply chain code before execution.
