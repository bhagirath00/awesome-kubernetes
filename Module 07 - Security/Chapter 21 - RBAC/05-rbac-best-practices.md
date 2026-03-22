# RBAC Best Practices

## Introduction

You now understand how RBAC works. This file shows you the patterns that work and the mistakes that don't.

These are real patterns used by security teams at large companies. Follow them and your RBAC will be solid.

---

## Best Practice 1: Principle of Least Privilege

Every user/service account should have the **minimum permissions needed to do their job**. Nothing more.

**WRONG:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]  # Can do EVERYTHING
```

If this account is compromised (laptop stolen, password phished, credentials leaked), the attacker has full cluster control.

**CORRECT:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
# Can manage deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]

# Can read services
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]

# Cannot delete (safer)
# Cannot access production
# Cannot read secrets
```

If this account is compromised, the attacker can only:
- Deploy to dev namespace
- Read services
- Cannot delete anything
- Cannot access production
- Cannot read secrets

The damage is contained.

---

## Best Practice 2: Separate Roles for Different Responsibilities

Different people have different jobs. Give each person exactly what they need.

**WRONG — one-size-fits-all:**
```yaml
# Everyone gets this role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: engineer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

Now developers, DBAs, and SREs all have full cluster access. A junior developer makes a mistake, deletes production.

**CORRECT — separate roles:**

```yaml
# Role 1: Developer (can manage deployments in dev)
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
# Role 2: DBA (can manage databases)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dba
rules:
- apiGroups: [""]
  resources: ["persistentvolumes", "persistentvolumeclaims"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["get", "list"]

---
# Role 3: SRE (can manage anything in production)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sre
  namespace: production
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

---
# Bind developers to developer role
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
- kind: Group
  name: developers@company.com

---
# Bind DBAs to dba role
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dbas
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dba
subjects:
- kind: Group
  name: dbas@company.com

---
# Bind SREs to sre role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sres
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: sre
subjects:
- kind: Group
  name: sres@company.com
```

Now:
- A developer can only break dev
- A DBA can only see database resources
- An SRE can manage production
- Clear separation of concerns

---

## Best Practice 3: Use Groups, Not Individual Users

Managing individual users is tedious and error-prone.

**WRONG — binding individuals:**
```yaml
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
- kind: User
  name: alice@company.com
- kind: User
  name: bob@company.com
- kind: User
  name: charlie@company.com
- kind: User
  name: david@company.com
```

When a new developer joins, you manually update this file. When someone leaves, you manually remove them. Mistakes happen.

**CORRECT — use groups:**
```yaml
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
- kind: Group
  name: developers@company.com  # One line!
```

Now:
- Everyone in `developers@company.com` automatically gets permissions
- New developers added to the group automatically get access
- Developers removed from group automatically lose access
- No manual file updates needed

---

## Best Practice 4: Namespace Isolation

Different teams get different namespaces. Use Role (not ClusterRole) for team-specific permissions.

```yaml
# Team A namespace
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-a

---
# Team A can do anything in their namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-a-admin
  namespace: team-a
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-admin
  namespace: team-a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: team-a-admin
subjects:
- kind: Group
  name: team-a@company.com

---
# Team B namespace
---
apiVersion: v1
kind: Namespace
metadata:
  name: team-b

---
# Team B can do anything in their namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-b-admin
  namespace: team-b
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-b-admin
  namespace: team-b
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: team-b-admin
subjects:
- kind: Group
  name: team-b@company.com
```

Now:
- Team A can do anything in `team-a` namespace
- Team B can do anything in `team-b` namespace
- Team A cannot see or modify Team B's resources
- If Team A's credentials are stolen, attacker only affects Team A

---

## Best Practice 5: Environment Separation

Separate dev/staging/production with RBAC.

**WRONG — same permissions everywhere:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]  # Can do anything in any namespace
```

A developer accidentally deletes production.

**CORRECT — different permissions per environment:**

```yaml
# Developers can do everything in dev
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
# Developers can only read in production (no modifications)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]

---
# Only SREs can modify production
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: sre
  namespace: production
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

Now:
- Developers can experiment in dev
- Developers can observe production but not modify it
- Only SREs can change production
- Safety gate prevents accidents

---

## Best Practice 6: Service Accounts with Minimal Permissions

Every service account should be locked down.

**WRONG:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: app-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: production
```

If the pod is compromised, the attacker has full cluster access.

**CORRECT:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app
  namespace: production
rules:
# Can read config maps (for configuration)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]

# Can read secrets (for passwords)
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]

# Can read other pods (for service discovery)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

If the pod is compromised, the attacker can only read configs and secrets, not delete anything or access other resources.

---

## Best Practice 7: Audit Your RBAC

Regularly check who has what permissions.

```bash
# List all bindings in a namespace
kubectl get rolebindings -n production

# List all cluster-wide bindings
kubectl get clusterrolebindings

# See what a user can do
kubectl auth can-i --list --as alice@company.com -n production

# Check if a user can perform a specific action
kubectl auth can-i delete pods -n production --as alice@company.com

# Export all RBAC rules for audit
kubectl get roles,clusterroles,rolebindings,clusterrolebindings -A -o yaml > rbac-audit.yaml
```

---

## Common RBAC Mistakes

### Mistake 1: Giving Everyone cluster-admin

```yaml
# WRONG — now everyone is admin
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: everyone-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: Group
  name: company.com  # Everyone in the company!
```

This defeats the purpose of RBAC.

### Mistake 2: Resource Wildcards

```yaml
# WRONG — can do anything with secrets and pods
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev
rules:
- apiGroups: ["*"]
  resources: ["secrets", "pods"]
  verbs: ["*"]  # Can do anything
```

Should be:
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "create", "update"]  # Specific verbs
```

### Mistake 3: Forgetting Namespace

```yaml
# WRONG — RoleBinding without namespace defaults to "default"
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer  # No namespace!
roleRef:
  kind: Role
  name: developer-role
subjects:
- kind: User
  name: alice@company.com
```

The binding is in `default` namespace, but the developer thinks they have permissions in `dev`. Confusion and security holes.

### Mistake 4: Over-Permissive Service Accounts

```yaml
# WRONG — default service account gets cluster-admin
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: default-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: default
  namespace: production  # ALL pods get cluster-admin!
```

Now every pod in production is an admin. Bad.

---

## RBAC Audit Checklist

Use this to audit your RBAC:

- [ ] No users/groups have `cluster-admin` unless necessary
- [ ] Every role follows least privilege principle
- [ ] Developers cannot modify production
- [ ] Service accounts have minimal permissions
- [ ] Each team has separate namespace/role
- [ ] Default service account has no extra permissions
- [ ] All bindings have explicit namespaces
- [ ] No resource wildcards (`*`) unless intentional
- [ ] No verb wildcards (`*`) unless intentional
- [ ] Groups are used instead of individual users
- [ ] Audit logs show who accessed what

---

## Summary

RBAC best practices:

1. **Least privilege** — give minimum needed permissions
2. **Separate roles** — different jobs get different roles
3. **Use groups** — easier to manage than individual users
4. **Namespace isolation** — teams in separate namespaces
5. **Environment separation** — dev/staging/production different permissions
6. **Service account lockdown** — minimal permissions for pods
7. **Regular audits** — verify RBAC is correct

Follow these patterns and your Kubernetes RBAC will be solid and secure.

---

## Module 7 Chapter 21 Summary

You now understand complete RBAC:

- **File 1**: RBAC overview (what is it, how does it work)
- **File 2**: Roles and ClusterRoles (how to define permissions)
- **File 3**: RoleBindings (how to assign to users)
- **File 4**: Service Accounts (how pods get permissions)
- **File 5 (this)**: Best practices

In the next chapter, **Chapter 22 - Pod Security**, we will learn how to lock down the pods themselves using SecurityContext, Pod Security Standards, and other pod-level security controls.
