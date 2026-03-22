# RoleBindings and ClusterRoleBindings

## Introduction

A role defines what can be done. A role binding defines who can do it.

A role is useless sitting alone. You need to connect it to users, groups, or service accounts. That connection is called a **binding**.

Think of it like a key:
- The role is the key (what it opens)
- The binding is the lock on the door (where the key works)
- The subject is the person with the key

---

## Anatomy of a RoleBinding

Every binding has three parts:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding        # Name of the binding
  namespace: dev                 # Namespace (only for RoleBinding)
roleRef:                         # Points to a role
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer-role           # Which role to use
subjects:                        # Who gets this role
- kind: User
  name: patel@company.com
- kind: Group
  name: developers@company.com
```

### Part 1: `roleRef` (The Role)

Points to the role to bind:

```yaml
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role                     # Can be Role or ClusterRole
  name: developer-role           # Name of the role
```

If binding a **ClusterRole**, you can use either RoleBinding or ClusterRoleBinding:
- **RoleBinding + ClusterRole** = permissions in one namespace only
- **ClusterRoleBinding + ClusterRole** = permissions in all namespaces

### Part 2: `subjects` (The Users)

Who gets the role. Can be:
- **User**: A person (email-like: `patel@company.com`)
- **Group**: A collection of people (`developers@company.com`)
- **ServiceAccount**: A pod or automation (`my-app-sa` in namespace `production`)

```yaml
subjects:
# User
- kind: User
  name: alice@company.com

# Group
- kind: Group
  name: developers

# Service account in same namespace
- kind: ServiceAccount
  name: my-app
  namespace: production

# Service account in different namespace
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

### Part 3: `metadata`

The name and namespace:

```yaml
metadata:
  name: developer-binding        # Name of this binding
  namespace: dev                 # Namespace (only for RoleBinding, not ClusterRoleBinding)
```

---

## Role vs RoleBinding vs ClusterRole vs ClusterRoleBinding

This confuses everyone. Let me clarify:

| Combination | Scope | Use Case |
|-------------|-------|----------|
| **Role + RoleBinding** | Single namespace | Developer permissions in one namespace |
| **ClusterRole + ClusterRoleBinding** | All namespaces | Cluster admin permissions |
| **ClusterRole + RoleBinding** | Single namespace | Shared permission template used in one namespace |

### Example 1: Developer in One Namespace

```yaml
# Role — limited to namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["*"]

---
# Binding — limited to namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer
subjects:
- kind: User
  name: alice@company.com
```

**Result:** Alice can do anything with deployments in `dev`, but cannot access other namespaces.

### Example 2: Cluster Admin

```yaml
# ClusterRole — cluster-wide
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

---
# ClusterBinding — cluster-wide
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: Group
  name: admins@company.com
```

**Result:** Everyone in the `admins` group can do anything on the cluster.

### Example 3: Read-Only Access Across Cluster

```yaml
# ClusterRole — cluster-wide permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]

---
# But limited to one namespace with RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-read-only
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: read-only
subjects:
- kind: User
  name: observer@company.com
```

**Result:** The `observer` can read pods/services in `dev` namespace only (even though the role is cluster-wide).

---

## Common Binding Patterns

### Pattern 1: Multiple Users with Same Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer-role
subjects:
# Multiple people get the same role
- kind: User
  name: alice@company.com
- kind: User
  name: bob@company.com
- kind: User
  name: charlie@company.com
```

All three users get the same permissions.

### Pattern 2: One User with Multiple Roles

```yaml
# First binding: Alice as developer
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-developer
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer
subjects:
- kind: User
  name: alice@company.com

---
# Second binding: Alice as admin in production
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-admin
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: admin
subjects:
- kind: User
  name: alice@company.com
```

Alice gets developer permissions in `dev` AND admin permissions in `production`.

### Pattern 3: Using Groups

```yaml
# Define once, bind to group
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["*"]

---
# Entire group gets permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developers
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer
subjects:
# Everyone in this group automatically has permissions
- kind: Group
  name: developers@company.com
```

When a new developer joins the company, add them to the `developers@company.com` group. They automatically get permissions.

### Pattern 4: Service Account Binding

```yaml
# Service account for CI/CD
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-cd
  namespace: production

---
# Role that CI/CD needs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "update", "patch"]

---
# Bind role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-cd-deployer
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: deployer
subjects:
- kind: ServiceAccount
  name: ci-cd
  namespace: production
```

Now the CI/CD pipeline (running in a pod) has permission to deploy.

---

## Testing Bindings

You can test if a binding works:

```bash
# Test if Alice can create deployments in dev
kubectl auth can-i create deployments --namespace dev --as alice@company.com

# Output: yes (if binding exists and allows it)
# Output: no (if binding doesn't exist or denies it)

# Test if service account can get secrets
kubectl auth can-i get secrets --namespace production \
  --as system:serviceaccount:production:ci-cd

# List all bindings in a namespace
kubectl get rolebindings --namespace dev

# Describe a binding
kubectl describe rolebinding developers --namespace dev
```

---

## Debugging Bindings

When a user says "I don't have permission to do X", debug it:

**Step 1: Check if the role exists**
```bash
kubectl get role developer --namespace dev
```

**Step 2: Check if the role has the permission**
```bash
kubectl describe role developer --namespace dev
```

**Step 3: Check if there is a binding**
```bash
kubectl get rolebindings --namespace dev
```

**Step 4: Check if the user is in the binding**
```bash
kubectl describe rolebinding developer-binding --namespace dev
```

**Step 5: Test with auth can-i**
```bash
kubectl auth can-i create deployments --namespace dev --as alice@company.com
```

---

## Common Mistakes

### Mistake 1: Forgetting the Namespace

```yaml
# WRONG — RoleBinding without namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  # namespace: dev  <— MISSING!
roleRef:
  kind: Role
  name: developer-role
```

RoleBinding MUST have a namespace. It will default to `default` namespace, so Alice will have permissions in the wrong namespace.

**Correct:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev   # ← REQUIRED
```

### Mistake 2: Referencing Non-Existent Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: binding
  namespace: dev
roleRef:
  kind: Role
  name: non-existent-role  # ← This role doesn't exist!
subjects:
- kind: User
  name: alice@company.com
```

The binding will be created, but it won't grant permissions because the role doesn't exist. Always verify the role name is correct.

### Mistake 3: Wrong Subject Kind

```yaml
# WRONG — treating user email as a name
subjects:
- kind: User
  name: alice@company.com
  namespace: dev  # ← Users don't have namespaces!
```

Users and groups do not have namespaces. Only ServiceAccounts have namespaces.

**Correct:**
```yaml
# User (no namespace)
subjects:
- kind: User
  name: alice@company.com

# Service account (with namespace)
- kind: ServiceAccount
  name: my-app
  namespace: production
```

---

## Summary

A binding connects a role to users/groups/service accounts:

**Parts:**
1. **roleRef**: Which role to use
2. **subjects**: Who gets the role
3. **metadata**: Name and namespace

**Key combinations:**
- Role + RoleBinding = permissions in one namespace
- ClusterRole + ClusterRoleBinding = permissions in all namespaces
- ClusterRole + RoleBinding = shared template for one namespace

In the next file, `04-service-accounts.md`, we will learn how to create service accounts and give them permissions so pods can authenticate to the Kubernetes API.
