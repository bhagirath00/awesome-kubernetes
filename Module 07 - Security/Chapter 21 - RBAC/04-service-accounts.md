# Service Accounts

## Introduction

You know how RBAC works for people (users and groups). But what about pods? 

A pod is an application running inside Kubernetes. Sometimes that application needs to make API calls to Kubernetes. For example:
- A deployment needs to scale itself based on metrics
- A monitoring pod needs to read metrics from all pods
- A CI/CD pod needs to deploy applications

These pods need to authenticate to the Kubernetes API. That is what **service accounts** are for.

A service account is an identity for a pod. When a pod makes an API call, it uses the service account to authenticate. RBAC then determines what it is allowed to do.

---

## How Service Accounts Work

Every pod in Kubernetes gets a service account (default is `default`).

When a pod is created:

```
Step 1: Kubernetes creates a service account token (JWT)
        Token is like a password that proves "I am this service account"

Step 2: Token is mounted into the pod at
        /var/run/secrets/kubernetes.io/serviceaccount/token

Step 3: Pod can use token to make API calls
        curl -H "Authorization: Bearer $TOKEN" https://kubernetes.default/api/v1/pods

Step 4: RBAC checks what this service account is allowed to do
        If it has permission → API call succeeds
        If not → API call fails
```

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: my-app-sa  # Use this service account
  containers:
  - name: app
    image: myapp:latest
```

Inside the pod:

```bash
# Pod is running
# Kubernetes automatically mounted token here:
$ cat /var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwI...

# Pod can use token to talk to API server
$ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
$ curl -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default/api/v1/namespaces/production/pods
```

---

## Creating Service Accounts

By default, each namespace has a `default` service account. You can create custom ones for specific applications:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
```

Then assign RBAC roles to it:

```yaml
# Service account (identity)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production

---
# Role (permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

---
# RoleBinding (connect service account to role)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-app-role
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production

---
# Pod uses the service account
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: my-app-sa  # Use the service account
  containers:
  - name: app
    image: myapp:latest
```

Now the pod has only the permissions in `my-app-role` (can read pods in production, nothing else).

---

## Common Service Account Patterns

### Pattern 1: Application with Minimal Permissions

```yaml
# Service account for a web application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app
  namespace: production

---
# Role — can read config maps (for configuration)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: web-app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]

---
# Binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: web-app-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: web-app-role
subjects:
- kind: ServiceAccount
  name: web-app
  namespace: production
```

The web app can only read config maps. It cannot read secrets, create pods, or delete anything.

### Pattern 2: Monitoring Service Account

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring

---
# ClusterRole — can read metrics from all namespaces
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
# ClusterRoleBinding — apply to service account
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

Prometheus can read metrics from any namespace (because it uses ClusterRole).

### Pattern 3: Job with Elevated Permissions

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database-backup
  namespace: backup

---
# Role — can read all resources
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: database-backup
rules:
- apiGroups: [""]
  resources: ["pods", "persistentvolumes"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list"]

---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: database-backup
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: database-backup
subjects:
- kind: ServiceAccount
  name: database-backup
  namespace: backup
```

The backup job can enumerate all pods and databases to create consistent backups.

---

## The Default Service Account

Every namespace has a `default` service account:

```bash
$ kubectl get serviceaccounts --namespace production
NAME      SECRETS
default   1
```

By default, the `default` service account has **no permissions** (RBAC denies everything unless explicitly allowed).

If you don't specify `serviceAccountName` in a pod, it uses `default`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: myapp:latest
    # serviceAccountName is implicitly "default"
```

This is good for security — pods cannot make API calls unless explicitly given permission.

**WRONG — giving default service account too many permissions:**

```yaml
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
  namespace: production  # Now ALL pods in production are admins!
```

This is bad because every pod, even ones you didn't write, becomes cluster admin.

**CORRECT — create named service account for specific pods:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-admin
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: production

---
# Only this specific pod uses the admin service account
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: my-app-sa  # Explicitly use admin account
  containers:
  - name: app
    image: myapp:latest
```

---

## Automounting Service Account Tokens

By default, service account tokens are automatically mounted into pods. This is usually fine, but sometimes you don't want it.

**Disable automounting for a service account:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
automountServiceAccountToken: false  # Don't auto-mount token
```

Now pods using this service account won't get a token mounted. They can't make API calls.

**Disable automounting for a specific pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: production
spec:
  serviceAccountName: my-app
  automountServiceAccountToken: false  # Don't mount token for this pod
  containers:
  - name: app
    image: myapp:latest
```

This is useful if the pod doesn't need to talk to the API server, reducing attack surface.

---

## Inspecting Service Account Tokens

To see what a service account token contains:

```bash
# Get the token
TOKEN=$(kubectl get secret $(kubectl get secret -n production | grep my-app-sa-token | awk '{print $1}') -n production -o jsonpath='{.data.token}' | base64 -d)

# Decode the JWT (in Python)
python3 -c "import base64; import json; parts = '$TOKEN'.split('.'); print(json.dumps(json.loads(base64.b64decode(parts[1] + '====')), indent=2))"

# Output shows:
# {
#   "aud": ["https://kubernetes.default.svc.cluster.local"],
#   "exp": 1234567890,
#   "iat": 1234567890,
#   "iss": "https://kubernetes.default",
#   "kubernetes.io": {
#     "namespace": "production",
#     "serviceaccount": {
#       "name": "my-app",
#       "uid": "abc123"
#     }
#   },
#   "nbf": 1234567890,
#   "sub": "system:serviceaccount:production:my-app"
# }
```

The token proves:
- Service account name: `my-app`
- Namespace: `production`
- Expiration time

---

## Best Practices for Service Accounts

1. **Create named service accounts for each application**
   ```yaml
   serviceAccountName: my-app-sa  # Don't use "default"
   ```

2. **Give service accounts minimal permissions**
   - Only what the application actually needs
   - Not cluster-admin unless absolutely necessary

3. **Use separate service accounts for different applications**
   - App A and App B get different service accounts
   - If one is compromised, it doesn't affect the other

4. **Disable token automounting if not needed**
   ```yaml
   automountServiceAccountToken: false
   ```

5. **Monitor service account usage**
   - Who is using which service account?
   - Are any service accounts inactive?
   - Are permissions correct?

---

## Summary

Service accounts are identities for pods:

**Key concepts:**
- Every pod uses a service account (default is `default`)
- Service account gets a token automatically mounted into pod
- Pod uses token to authenticate to Kubernetes API
- RBAC determines what the service account can do

**Workflow:**
```
Create ServiceAccount
    ↓
Create Role (permissions)
    ↓
Create RoleBinding (connect service account to role)
    ↓
Pod specifies serviceAccountName
    ↓
Pod authenticates with token, gets permissions from RBAC
```

In the next file, `05-rbac-best-practices.md`, we will see common RBAC patterns and mistakes teams make.
