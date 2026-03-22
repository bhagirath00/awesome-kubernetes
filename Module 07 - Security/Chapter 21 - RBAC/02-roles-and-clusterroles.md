# Roles and ClusterRoles

## Introduction

A role is a set of permissions. It defines what actions can be performed on which resources.

This file teaches you how to write roles. You will learn the anatomy of a role, how to specify permissions precisely, and common patterns that security teams use.

---

## Anatomy of a Role

Every role has the same structure:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
```

Let me break this down:

### Field 1: `kind`

Either `Role` (namespace-scoped) or `ClusterRole` (cluster-wide).

### Field 2: `metadata`

The name and namespace:

```yaml
metadata:
  name: developer-role        # Name of the role
  namespace: dev              # (Only for Role, not ClusterRole)
```

### Field 3: `rules`

An array of permission rules. Each rule specifies:
- Which API groups
- Which resources
- Which verbs (actions)

#### API Groups

Resources are grouped by API. Examples:

| API Group | Resources | Examples |
|-----------|-----------|----------|
| `apps` | Deployments, StatefulSets, DaemonSets | `["apps"]` |
| `""` (empty) | Pods, Services, ConfigMaps, Secrets | `[""]` |
| `batch` | Jobs, CronJobs | `["batch"]` |
| `networking.k8s.io` | NetworkPolicies | `["networking.k8s.io"]` |
| `rbac.authorization.k8s.io` | Roles, RoleBindings | `["rbac.authorization.k8s.io"]` |
| `*` | All API groups | `["*"]` |

#### Resources

What can be accessed. Examples:

```yaml
resources:
- "pods"                    # Read/modify pods
- "deployments"             # Read/modify deployments
- "secrets"                 # Read/modify secrets
- "nodes"                   # Read/modify nodes
- "events"                  # Read events
- "*"                       # All resources
```

You can also specify sub-resources:

```yaml
resources:
- "pods/logs"              # Access pod logs
- "pods/exec"              # Execute commands in pods
- "deployments/scale"      # Scale deployments
```

#### Verbs

What actions can be performed:

```yaml
verbs:
- "get"                    # Read single resource (kubectl get pod my-pod)
- "list"                   # Read multiple resources (kubectl get pods)
- "watch"                  # Monitor resources for changes
- "create"                 # Create new resource
- "update"                 # Modify existing resource (kubectl apply)
- "patch"                  # Patch existing resource
- "delete"                 # Delete resource
- "deletecollection"       # Delete multiple resources
- "*"                      # All verbs
```

---

## Writing Effective Roles

### Rule 1: Be Specific

**WRONG — way too broad:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]  # Can do ANYTHING
```

**CORRECT — specific permissions:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
# Can manage deployments in dev namespace
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

# Can read services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]

# Can read config maps
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]

# Can read (but not modify) secrets
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

### Rule 2: Namespace vs Cluster Scope

**Use Role for namespace-scoped:**
- Different teams, different namespaces
- Developer should only access their namespace
- Limits blast radius

```yaml
# Developer in dev namespace only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev  # Important: limited to dev
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["*"]
```

**Use ClusterRole for cluster-scoped:**
- Cluster-wide resources (nodes, persistent volumes)
- Cluster admin functions
- Shared service accounts

```yaml
# Cluster-wide permission
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

### Rule 3: Combine Multiple Rules

One role can have many rules. Each rule is combined with OR logic (if any rule matches, action is allowed):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: application-admin
  namespace: production
rules:
# Rule 1: Can manage deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["*"]

# Rule 2: Can manage services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["*"]

# Rule 3: Can read events (for troubleshooting)
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]

# Rule 4: Can read pod logs (for debugging)
- apiGroups: [""]
  resources: ["pods/logs"]
  verbs: ["get", "list"]
```

### Rule 4: Resource Names (Optional)

Restrict to specific resource names:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: limited-deployer
  namespace: production
rules:
# Can only modify the "payment-service" deployment
- apiGroups: ["apps"]
  resources: ["deployments"]
  resourceNames: ["payment-service"]
  verbs: ["get", "update", "patch"]
```

This is powerful for isolation — a developer can only modify the deployment they own.

---

## Common Role Patterns

### Pattern 1: Read-Only Auditor

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: auditor
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list"]
```

An auditor can see everything but cannot modify anything.

### Pattern 2: Namespace Manager

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-manager
  namespace: dev
rules:
# Full permissions in this namespace only
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

A namespace manager can do anything in their namespace, but cannot affect other namespaces.

### Pattern 3: CI/CD Pipeline

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-cd
  namespace: production
rules:
# Can deploy (create/update deployments)
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "update", "patch"]

# Can update image (for rolling updates)
- apiGroups: ["apps"]
  resources: ["deployments/rollback"]
  verbs: ["create"]

# Can check status
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get", "list"]

# Can access secrets (needed for deployment credentials)
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
```

A CI/CD pipeline gets enough permission to deploy but cannot delete or modify other things.

### Pattern 4: Logging Service

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: log-collector
rules:
# Can read pod logs
- apiGroups: [""]
  resources: ["pods/logs"]
  verbs: ["get", "list"]

# Can list pods (to find ones to get logs from)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list", "get"]

# Can read events
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
```

A logging service gets permission to collect logs but not modify anything.

---

## Role vs ClusterRole: When to Use What

**Use Role (namespace-scoped) when:**
- Different teams own different namespaces
- You want to limit blast radius if credentials stolen
- You have dev/staging/production in different namespaces
- You want stronger isolation

**Use ClusterRole (cluster-wide) when:**
- Permission applies to cluster-wide resources (nodes, namespaces, persistent volumes)
- Service account needs access across multiple namespaces
- Cluster-level permissions (cluster admin, add nodes, etc.)

**Mixed approach (recommended):**
```yaml
# Namespace-specific permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

---
# Cluster-wide read-only (can see cluster resources but not modify)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-observer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
```

---

## Testing Your Roles

You can test if a role has a permission using `kubectl auth can-i`:

```bash
# Test if user can create deployments
kubectl auth can-i create deployments --namespace dev --as patel@company.com

# Test if service account can read secrets
kubectl auth can-i get secrets --namespace production --as system:serviceaccount:production:my-app
```

---

## Summary

A role is a set of rules. Each rule specifies:
- **apiGroups**: Which API group (apps, "", batch, etc.)
- **resources**: Which resource types (deployments, pods, secrets)
- **verbs**: Which actions (get, list, create, delete, etc.)

Key principles:
- **Be specific** — give only what's needed (least privilege)
- **Use namespaced roles** when possible (Role, not ClusterRole)
- **Combine rules** in one role for related permissions
- **Use resourceNames** to limit to specific resources

In the next file, `03-rolebindings.md`, we will learn how to assign these roles to users and service accounts.
