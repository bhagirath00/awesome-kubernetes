# ArgoCD RBAC, Multi-Tenancy & Project Governance

## 1. Architectural Concept & Deep Dive

In enterprise environments, multiple engineering teams share a single central ArgoCD control plane. Without strict multi-tenancy and Role-Based Access Control (RBAC):
- The frontend team could accidentally delete or overwrite the backend database deployment.
- Developers could point ArgoCD to untrusted personal GitHub repositories.
- Applications could bypass security by deploying `ClusterRoleBinding` resources granting themselves root privileges.

ArgoCD enforces multi-tenancy through two complementary security layers:
1. **`AppProject` CRD**: The logical boundary defining **what resources** an application can deploy, **which Git repos** it can pull from, and **which clusters and namespaces** it can target.
2. **ArgoCD RBAC ConfigMap (`argocd-rbac-cm`)**: The authorization policy defining **which humans and CI bots** can perform actions (read, sync, update, delete) in the Web UI and CLI.

```
+─────────────────────────────────────────────────────────────────────────+
|                 ARGOCD MULTI-TENANCY & RBAC ARCHITECTURE                |
|                                                                         |
|  [ User: Alice (Group: engineering:payments) ]                          |
|         │                                                               |
|         ├── Authenticates via Okta / GitHub OIDC                        |
|         ▼                                                               |
|  [ ArgoCD RBAC Engine (argocd-rbac-cm) ]                                |
|         │                                                               |
|         ├── Evaluates: g, engineering:payments, role:payment-admin      |
|         ├── Permits:   p, role:payment-admin, applications, sync, ...   |
|         ▼                                                               |
|  [ AppProject Boundary: "payment-project" ]                             |
|         │                                                               |
|         ├── Allowed Repos:     https://github.com/myorg/payments.git    |
|         ├── Allowed Clusters:  production-eks-cluster                   |
|         ├── Allowed Namespaces:payment-prod, payment-staging            |
|         ├── Blacklisted Kinds: ClusterRole, ClusterRoleBinding          |
|         ▼                                                               |
|  [ Target Namespace: payment-prod on EKS Cluster ]                      |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Defining Logical Boundaries with `AppProject`

Every `Application` belongs to an `AppProject` (defaulting to the `default` project if unspecified). Production teams should create isolated projects:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payment-team-project
  namespace: argocd
spec:
  description: "Governance boundary for the core payment services team"

  # 1. Source Repositories Whitelist (Blocks rogue repos!)
  sourceRepos:
    - "https://github.com/myorg/payment-gitops.git"
    - "https://charts.bitnami.com/bitnami"

  # 2. Destination Clusters and Namespaces Whitelist
  destinations:
    - namespace: "payment-staging"
      server: "https://kubernetes.default.svc"
    - namespace: "payment-production"
      name: "eks-prod-us-east-1"

  # 3. Cluster-Scoped Resources Blacklist (Prevent Privilege Escalation!)
  clusterResourceBlacklist:
    - group: "rbac.authorization.k8s.io"
      kind: "ClusterRole"
    - group: "rbac.authorization.k8s.io"
      kind: "ClusterRoleBinding"
    - group: ""
      kind: "Namespace"

  # 4. Namespace-Scoped Resources Whitelist
  namespaceResourceWhitelist:
    - group: "apps"
      kind: "Deployment"
    - group: ""
      kind: "Service"
    - group: ""
      kind: "ConfigMap"
    - group: "networking.k8s.io"
      kind: "Ingress"
```

---

## 3. Configuring User RBAC in `argocd-rbac-cm`

ArgoCD RBAC is configured via CSV policy lines inside the `argocd-rbac-cm` ConfigMap in the `argocd` namespace.

### The Policy CSV Syntax
```
p, <subject/role>, <resource>, <action>, <object>, <allow/deny>
g, <user/group>, <role>
```

- **Resources**: `applications`, `projects`, `clusters`, `repositories`, `certificates`, `logs`, `exec`.
- **Actions**: `get`, `create`, `update`, `delete`, `sync`, `override`.

### Production `argocd-rbac-cm` Manifest
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  # Set default policy for all authenticated users to read-only
  policy.default: role:readonly

  # Scopes to match against incoming OIDC token claims
  scopes: "[groups, email]"

  policy.csv: |
    # 1. Define custom role for payment engineers
    p, role:payment-developer, applications, get, payment-team-project/*, allow
    p, role:payment-developer, applications, sync, payment-team-project/*, allow
    p, role:payment-developer, logs, get, payment-team-project/*, allow
    p, role:payment-developer, exec, create, payment-team-project/*, allow

    # 2. Define custom role for DevOps platform admins
    p, role:platform-admin, *, *, *, allow

    # 3. Map IdP Groups (GitHub / Okta / Azure AD) to ArgoCD Roles
    g, "myorg:payments-team", role:payment-developer
    g, "myorg:devops-admins", role:platform-admin
    g, "myorg:security-auditors", role:readonly
```

---

## 4. Web Terminal & Exec RBAC (`exec`)

ArgoCD allows running interactive shell sessions inside running pods directly from the Web UI! Because this is a high-risk capability, restrict `exec` access exclusively to designated emergency roles:

```
# Allow exec ONLY for payment-developer role on pods in payment-team-project
p, role:payment-developer, exec, create, payment-team-project/*, allow
```
If an unprivileged user clicks the **Terminal** tab in the UI, ArgoCD denies access with `PermissionDenied`.

---

## 5. Verification & Troubleshooting Runbook

### Testing RBAC Policies via CLI
You can test your policy rules without logging in as the target user:

```bash
# Validate policy syntax and test permissions for a specific user
argocd admin settings rbac can "alice@myorg.com" sync applications "payment-team-project/payment-api" \
  --namespace argocd
```
Output:
```
Yes
```

Check if user can delete cluster roles:
```bash
argocd admin settings rbac can "alice@myorg.com" create clusterrolebindings "*" \
  --namespace argocd
```
Output:
```
No
```

---

## 6. Key Takeaways
- Use `AppProject` CRDs to enforce infrastructure boundaries (whitelisting allowed Git repos, target clusters, and permitted K8s kinds).
- Use `argocd-rbac-cm` CSV policies to govern what actions human users and CI bots can perform.
- Always set `policy.default: role:readonly` to follow the principle of least privilege.
- Map IdP groups directly from Okta, Azure AD, or GitHub to ArgoCD roles to automate identity lifecycle management.
