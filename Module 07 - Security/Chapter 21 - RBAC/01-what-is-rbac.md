# What is RBAC (Role-Based Access Control)

## Introduction

RBAC is how Kubernetes answers one simple question: **"Who is allowed to do what?"**

You have already authenticated (you proved who you are). Now comes authorization: does that person have permission to create deployments? Can they delete secrets? Can they access production?

Without RBAC, anyone who can access the Kubernetes API can do anything. That is a security disaster. With RBAC, you define exactly what each user, group, or service account is allowed to do.

Think of it like a hotel:
- The front desk has a key to the front door (can check guests in)
- Housekeeping has a key to guest rooms (can clean them)
- Maintenance has a key to the basement (can fix pipes)
- The chef has a key to the kitchen (can prepare meals)

Each person has the minimum keys they need. If a housekeeping employee's key is stolen, the thief can only access guest rooms — not the kitchen, basement, or front desk.

Kubernetes RBAC works the same way. Each user/service account gets exactly the permissions they need to do their job. Nothing more.

---

## How RBAC Works (High Level)

When you run a kubectl command:

```
Step 1: You authenticate
        kubectl knows who you are (certificate, token, etc.)

Step 2: You make a request
        kubectl: "Create a deployment named 'my-app'"

Step 3: API Server receives it
        API Server: "You say you are 'patel@company.com'"

Step 4: API Server checks RBAC rules
        API Server: "Does 'patel@company.com' have permission to create deployments?"
        
Step 5a: If YES → Request allowed ✓
         Deployment is created

Step 5b: If NO → Request blocked ✗
         Error: "forbidden: User 'patel@company.com' cannot create deployments"
```

The rules that determine YES or NO are RBAC rules.

---

## The Three Questions RBAC Answers

### Question 1: Who?

**Who** is making the request? Could be:
- A person (developer, operations engineer, security team)
- A service account (pod running inside Kubernetes)
- A group (all developers, all DBAs)

Examples:
- User: `patel@company.com`
- Service Account: `my-app` (running in the `production` namespace)
- Group: `developers` (all people in the engineering team)

### Question 2: What?

**What** resource are they trying to access? Could be:
- Deployments
- Pods
- Secrets
- Services
- Any Kubernetes resource

Examples:
- Resource: `deployments` in `apps` API group
- Resource: `secrets` in the core API group
- Resource: `nodes` (cluster-wide resource)

### Question 3: How?

**How** are they trying to use it? The action/verb. Could be:
- `get` — read a single resource
- `list` — read multiple resources
- `create` — create a new resource
- `update` — modify an existing resource
- `delete` — remove a resource
- `watch` — monitor a resource for changes

Examples:
- Action: `get pods` (read a single pod)
- Action: `create deployments` (create new deployment)
- Action: `delete secrets` (delete a secret)

---

## RBAC Components

RBAC consists of four Kubernetes resources:

### 1. Role or ClusterRole

**A role is a set of permissions.** It says "this role can do these things."

```yaml
# Role — limited to one namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]
```

Translation: "In the `dev` namespace, this role can get/list/create/update/patch deployments."

```yaml
# ClusterRole — applies to entire cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: admin-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

Translation: "This role can do anything in any namespace."

### 2. RoleBinding or ClusterRoleBinding

**A binding connects a role to a user/service account.** It says "give this user this role."

```yaml
# RoleBinding — limited to one namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: dev
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer-role
subjects:
- kind: User
  name: patel@company.com
```

Translation: "Give the user `patel@company.com` the `developer-role` in the `dev` namespace."

```yaml
# ClusterRoleBinding — applies to entire cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin-role
subjects:
- kind: Group
  name: admins
```

Translation: "Give everyone in the `admins` group the `admin-role` cluster-wide."

### 3. User/Group/ServiceAccount

These are the **subjects** — the entities that get permissions.

- **User**: A person (e.g., `patel@company.com`)
- **Group**: A collection of people (e.g., `developers`)
- **ServiceAccount**: A pod or automation (e.g., `my-app-sa`)

### 4. The Full Flow

```
Role ("What can be done?")
    ↓
  + RoleBinding ("Who gets this role?")
    ↓
Result: User has permissions
```

Example:

```yaml
---
# Define what the role can do
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# Give this role to a user
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: User
  name: alice@company.com
```

**Result:** Alice can get/list/watch pods in production, but nothing else.

---

## Role vs ClusterRole

These are similar but different scope:

| Aspect | Role | ClusterRole |
|--------|------|-------------|
| **Scope** | Single namespace | Entire cluster |
| **Syntax** | `kind: Role` | `kind: ClusterRole` |
| **Use Case** | Team-level permissions | Cluster-level permissions |
| **Example** | Developer in `dev` namespace | Cluster admin |

**When to use Role:**
- Different teams manage different namespaces
- Each team should only see/modify their namespace
- Safer (limits blast radius if credentials stolen)

**When to use ClusterRole:**
- Cluster-wide resources (nodes, persistent volumes)
- Global permissions (cluster admin)
- Shared service accounts

Similarly, RoleBinding is single-namespace, ClusterRoleBinding is cluster-wide.

---

## Common RBAC Patterns

### Pattern 1: Developer Can Manage Their Namespace

```yaml
# Developers can do everything in dev namespace
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
# Apply to developer group
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
```

### Pattern 2: Read-Only Access

```yaml
# User can only read things
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]

---
# Apply to stakeholders
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: stakeholder-read-only
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: read-only
subjects:
- kind: Group
  name: stakeholders@company.com
```

### Pattern 3: Service Account with Minimal Permissions

```yaml
# Prometheus monitoring pod needs to read metrics
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]

---
# Apply to prometheus service account
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
```

---

## The Principle of Least Privilege

This is the most important RBAC principle: **Give each user/service account the minimum permissions they need to do their job. Nothing more.**

**WRONG:**
```yaml
# Way too much power
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployer
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]  # Can do anything!
```

If this account is compromised, the attacker has full cluster control.

**CORRECT:**
```yaml
# Exactly what's needed
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployer
rules:
# Can only create/update deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "update", "patch", "get", "list"]
# Can read configs (needed to configure the deployment)
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get"]
# Can read service status
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
```

If this account is compromised, the attacker can only deploy to production — they cannot delete databases, modify other services, or access secrets beyond what the deployment needs.

---

## Why RBAC Matters for Security

Imagine without RBAC:

```
All developers have cluster-admin access
    ↓
Developer's laptop gets stolen
    ↓
Attacker has full Kubernetes access
    ↓
Attacker deletes everything
    ↓
All production pods are gone
```

Now with proper RBAC:

```
Each developer has minimal permissions (can only deploy to dev)
    ↓
Developer's laptop gets stolen
    ↓
Attacker has dev namespace access
    ↓
Attacker can only affect dev (not production)
    ↓
Incident contained, easily recovered
```

The damage is limited to what that role can do.

---

## What's Next

In this chapter, we will go deep on RBAC:

1. **This file**: RBAC overview (done)
2. **Roles and ClusterRoles**: How to define permissions
3. **RoleBindings**: How to assign roles to users
4. **Service Accounts**: How pods get permissions
5. **RBAC Best Practices**: Common patterns and mistakes

---

## Summary

RBAC answers three questions:
- **Who?** User, group, or service account
- **What?** Which resources (pods, deployments, secrets)
- **How?** Which actions (get, list, create, delete)

RBAC consists of:
- **Roles/ClusterRoles**: Define permissions
- **RoleBindings/ClusterRoleBindings**: Assign roles to users

The core principle: **least privilege**. Give each user the minimum permissions they need.

In the next file, we will go deep on **Roles and ClusterRoles** — how to define exactly what permissions a role has.
