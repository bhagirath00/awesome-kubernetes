# Writing OPA Gatekeeper Policies with Rego

## 1. Architectural Concept & Deep Dive

**Open Policy Agent (OPA) Gatekeeper** enforces policy as code at the Kubernetes admission control layer. Gatekeeper decouples policy logic from Kubernetes API primitives using **Rego**, a declarative query language designed specifically for evaluating structured hierarchical data (JSON/YAML).

In Gatekeeper, policy enforcement is split into two Kubernetes Custom Resource Definitions (CRDs):
1. **`ConstraintTemplate`**: Defines the executable Rego logic, schema validation for parameters, and generates a dynamic custom constraint CRD.
2. **`Constraint`**: An instantiation of a ConstraintTemplate that specifies target scopes (namespaces, resource kinds) and passes concrete runtime parameters (e.g., allowed image registries or forbidden host paths).

```
+─────────────────────────────────────────────────────────────────────────+
|                     GATEKEEPER POLICY EVALUATION                        |
|                                                                         |
|  kubectl apply -f pod.yaml ──► Kubernetes API Server                    |
|                                      │                                  |
|                                      ▼ (ValidatingAdmissionWebhook)     |
|                       Gatekeeper Admission Webhook                      |
|                                      │                                  |
|         ┌────────────────────────────┴────────────────────────────┐     |
|         ▼                                                         ▼     |
|   ConstraintTemplate (Rego)                              Constraint CRD |
|   • Defines violation logic                              • Target: Pods |
|   • Parses input.review.object                           • Parameters:  |
|   • Compares against parameters                            repos: [...] |
|         │                                                         │     |
|         └────────────────────────────┬────────────────────────────┘     |
|                                      ▼                                  |
|                      Rego Engine Evaluates Expression                   |
|                                      │                                  |
|                 ┌────────────────────┴────────────────────┐             |
|                 ▼                                         ▼             |
|          violation[] is empty?                    violation[] has item? |
|          ==> ADMIT REQUEST                        ==> REJECT (403)      |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Anatomy of the Gatekeeper Input Object

When the Kubernetes API server intercepts a request, it serializes the `AdmissionReview` payload and hands it to Gatekeeper. In Rego, this object is accessible under the global **`input`** document:

- **`input.review.object`**: The full JSON representation of the resource being created or modified (e.g. `input.review.object.spec.containers`).
- **`input.review.userInfo`**: The identity of the caller (username, groups, UID).
- **`input.review.operation`**: The lifecycle operation (`CREATE`, `UPDATE`, `DELETE`).
- **`input.parameters`**: Key-value pairs supplied by the specific `Constraint` CRD instantiating the template.

---

## 3. Core Rego Language Constructs for Gatekeeper

Rego rules evaluate assertions. In Gatekeeper, a policy succeeds if no rule matches. If any **`violation[{"msg": msg}]`** set rule evaluates to true, Gatekeeper rejects the request with the specified message.

### 1. Iteration and Array Traversal (`some ... in ...` or `[_]`)
To inspect every container in a pod:
```rego
violation[{"msg": msg}] {
    container := input.review.object.spec.containers[_]
    not container.resources.limits.cpu
    msg := sprintf("Container '%v' has no CPU limit defined", [container.name])
}
```
*(The `[_]` wildcard iterates over all array elements; if any container lacks `.limits.cpu`, the rule fires).*

### 2. Set Operations & Prefix Matching
```rego
violation[{"msg": msg}] {
    container := input.review.object.spec.containers[_]
    image := container.image
    valid_repos := input.parameters.repos
    not any([startswith(image, repo) | repo := valid_repos[_]])
    msg := sprintf("Image '%v' is not from an approved registry: %v", [image, valid_repos])
}
```

---

## 4. Hands-on Lab: Building an Image Registry Whitelist Policy

Let's build a production-grade policy that mandates all container images originate exclusively from approved enterprise registries (e.g. `123456789.dkr.ecr.us-east-1.amazonaws.com/` or `ghcr.io/myorg/`).

### Step 1: Create the ConstraintTemplate (`template.yaml`)
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          satisfied := [good | repo = input.parameters.repos[_] ; good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("Container '%v' uses unauthorized image '%v'. Approved prefixes: %v", [container.name, container.image, input.parameters.repos])
        }

        # Check initContainers as well!
        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          satisfied := [good | repo = input.parameters.repos[_] ; good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("InitContainer '%v' uses unauthorized image '%v'. Approved prefixes: %v", [container.name, container.image, input.parameters.repos])
        }
```

Apply template:
```bash
kubectl apply -f template.yaml
```

### Step 2: Instantiate the Constraint (`constraint.yaml`)
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: whitelist-production-registries
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - production
      - staging
  parameters:
    repos:
      - "ghcr.io/myorg/"
      - "123456789.dkr.ecr.us-east-1.amazonaws.com/"
```

Apply constraint:
```bash
kubectl apply -f constraint.yaml
```

### Step 3: Test Enforcement
Attempt to deploy an unauthorized public Docker Hub image into `production`:
```bash
kubectl run rogue-pod --image=nginx:alpine -n production
```
Output:
```
Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request: 
[whitelist-production-registries] Container 'rogue-pod' uses unauthorized image 'nginx:alpine'. 
Approved prefixes: ["ghcr.io/myorg/", "123456789.dkr.ecr.us-east-1.amazonaws.com/"]
```

---

## 5. Phased Rollouts: Enforcement Actions

Never apply new policies with `enforcementAction: deny` directly in live production clusters. Gatekeeper supports three progressive enforcement modes:

| Enforcement Mode | Action on Violation | User Visibility | Audit Logged? |
| :--- | :--- | :--- | :--- |
| **`dryrun`** | Admitted | Zero user impact | Yes (logged in Constraint `.status.violations`) |
| **`warn`** | Admitted | Warning banner returned to terminal on `kubectl apply` | Yes |
| **`deny`** | Blocked (HTTP 403) | Request immediately rejected | Yes |

```yaml
spec:
  enforcementAction: warn  # Safe progressive testing!
```

---

## 6. Verification & Troubleshooting Runbook

### Diagnostic Commands
```bash
# View all active Gatekeeper ConstraintTemplates
kubectl get constrainttemplates

# Inspect violations detected across the cluster by a constraint
kubectl describe k8sallowedrepos whitelist-production-registries

# View Gatekeeper controller audit logs
kubectl logs -n gatekeeper-system -l control-plane=controller-manager --tail=100
```

### Common Pitfalls & Solutions
1. **Rego compile failure in ConstraintTemplate**:
   - Cause: Syntax error in Rego block or indentation issue.
   - Solution: Check `kubectl describe constrainttemplate <NAME>`. Look at `status.errors` for exact line numbers.
2. **Policy blocks pod updates from Deployments**:
   - Cause: Matching on `Pod` directly while mutating deployments can cause Deployments to create ReplicaSets that fail to create Pods.
   - Solution: Include higher-order workload kinds (`Deployment`, `StatefulSet`, `Job`) in `match.kinds` or test via dry-run first.
3. **Webhook timeout blocking all cluster operations**:
   - Cause: Gatekeeper controller pods crashed or overwhelmed by admission requests.
   - Solution: Gatekeeper installs with `failurePolicy: Ignore` by default to prevent control plane lockouts. If set to `Fail`, temporarily disable webhook via `kubectl delete validatingwebhookconfiguration gatekeeper-validating-webhook-configuration`.

---

## 7. Key Takeaways
- OPA Gatekeeper pairs declarative Rego logic (`ConstraintTemplate`) with parameterized CRDs (`Constraint`).
- The `input.review.object` tree contains the candidate Kubernetes manifest being evaluated.
- A policy triggers a rejection whenever the `violation[{"msg": msg}]` rule evaluates to true.
- Use `enforcementAction: dryrun` and `warn` to profile violations across existing workloads before enforcing strict `deny`.
