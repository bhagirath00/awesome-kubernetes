# Production OPA Gatekeeper Policy Library

## 1. Architectural Concept & Deep Dive

This chapter provides a battle-tested library of five production-grade OPA Gatekeeper policies used in enterprise Kubernetes clusters to enforce security boundaries, prevent cost overruns, and maintain operational governance:

```
+─────────────────────────────────────────────────────────────+
|                ENTERPRISE GATEKEEPER GUARDRAILS             |
|                                                             |
|   1. Mandatory Labels        (Cost attribution & ownership) |
|   2. Disallow Privileged     (Prevent host breakout & root) |
|   3. Resource Guardrails     (Enforce CPU/Memory limits)    |
|   4. Disallow Latest Tag     (Enforce immutable deployments)|
|   5. Ingress/Service Guards  (Block accidental public IPs)  |
+─────────────────────────────────────────────────────────────+
```

---

## 2. Policy 1: Enforce Mandatory Metadata Labels

Ensures every namespace, deployment, and service has mandatory governance tags (`owner`, `environment`, `cost-center`) for FinOps cost allocation.

### ConstraintTemplate (`k8srequiredlabels-template.yaml`)
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Resource '%v' is missing mandatory labels: %v", [input.review.object.metadata.name, missing])
        }
```

### Constraint (`required-labels-constraint.yaml`)
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-and-env
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet"]
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["environment", "team", "cost-center"]
```

---

## 3. Policy 2: Block Privileged Containers (`privileged: true`)

Prevents containers from acquiring host-level capabilities and escaping into the underlying node operating system.

### ConstraintTemplate (`k8sdisallowprivileged-template.yaml`)
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowPrivileged
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowprivileged

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged container '%v' is forbidden!", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged initContainer '%v' is forbidden!", [container.name])
        }
```

### Constraint (`block-privileged-constraint.yaml`)
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowPrivileged
metadata:
  name: block-privileged-containers
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - cilium-system
```

---

## 4. Policy 3: Enforce CPU and Memory Limits

Protects nodes from noisy neighbors and OOM crashes by ensuring all containers declare both CPU and Memory requests and limits.

### ConstraintTemplate (`k8srequirelimits-template.yaml`)
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirelimits
spec:
  crd:
    spec:
      names:
        kind: K8sRequireLimits
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container '%v' must define resources.limits.cpu", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' must define resources.limits.memory", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.cpu
          msg := sprintf("Container '%v' must define resources.requests.cpu", [container.name])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.requests.memory
          msg := sprintf("Container '%v' must define resources.requests.memory", [container.name])
        }
```

---

## 5. Policy 4: Disallow `:latest` Image Tag

Enforces deterministic gitops rollouts by rejecting untagged images or `:latest` floating tags.

### ConstraintTemplate (`k8sdisallowlatesttag-template.yaml`)
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdisallowlatesttag
spec:
  crd:
    spec:
      names:
        kind: K8sDisallowLatestTag
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdisallowlatesttag

        has_latest_tag(image) {
          endswith(image, ":latest")
        }

        has_latest_tag(image) {
          not contains(image, ":")
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          has_latest_tag(container.image)
          msg := sprintf("Container '%v' uses disallowed image tag in '%v'. Pinned semantic tags or SHA256 digests required.", [container.name, container.image])
        }
```

---

## 6. Policy 5: Block Public LoadBalancer Services in Internal Namespaces

Prevents engineers from accidentally exposing internal databases or microservices to the public internet via cloud LoadBalancers (AWS NLB/ALB, GCP LB).

### ConstraintTemplate (`k8sblockloadbalancer-template.yaml`)
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockloadbalancer
spec:
  crd:
    spec:
      names:
        kind: K8sBlockLoadBalancer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblockloadbalancer

        violation[{"msg": msg}] {
          input.review.object.spec.type == "LoadBalancer"
          msg := sprintf("Service '%v' cannot be of type LoadBalancer in protected namespace '%v'. Only ClusterIP or NodePort permitted.", [input.review.object.metadata.name, input.review.object.metadata.namespace])
        }
```

### Constraint (`block-public-lb-internal.yaml`)
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockLoadBalancer
metadata:
  name: block-lb-in-internal-namespaces
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Service"]
    namespaces:
      - payment-backend
      - database-cluster
      - default
```

---

## 7. Testing & Auditing Deployed Constraints

```bash
# Apply all templates and constraints
kubectl apply -f templates/
kubectl apply -f constraints/

# Verify audit status across all constraints
kubectl get constraints
```
Output:
```
NAME                             ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
require-team-and-env             deny                 0
block-privileged-containers      deny                 0
block-lb-in-internal-namespaces  deny                 0
```

---

## 8. Key Takeaways
- Mandatory labels guarantee accurate FinOps cost grouping and organizational attribution.
- Dropping privileged mode in Gatekeeper protects against CVEs targeting Linux container isolation.
- Enforcing resource requests and limits ensures the Kubernetes scheduler and OOM killer function predictably.
- Blocking `:latest` tags prevents unpredictable rolling restart drift in production clusters.
