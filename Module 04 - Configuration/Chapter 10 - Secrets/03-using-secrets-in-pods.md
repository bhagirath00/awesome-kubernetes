# Using Secrets in Pods

## Introduction

Injecting Secrets into Pods follows the same two patterns as ConfigMaps: environment variables and volume mounts. The mechanics are nearly identical but the security implications differ — Secrets injected as environment variables have some risks that file mounts avoid. This file covers both patterns with production-grade guidance.

---

## Method 1: Inject a Single Secret Key as an Environment Variable

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: api
        image: api-server:v2.0
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials   # Secret name
              key: password          # key within the Secret
        
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: api-credentials
              key: api-key
              optional: false        # Pod fails to start if Secret or key missing
```

```bash
# Verify Secret is injected (do not do this in a shared session)
kubectl exec -it api-pod-abc123 -n production -- printenv DB_USERNAME
# admin

# Check from application logs — look for connection string without password exposed
kubectl logs api-pod-abc123 -n production | grep "Connected to database"
# Connected to database: postgres-service:5432/myapp (user: admin)
```

---

## Method 2: Inject All Keys from a Secret — `envFrom`

```yaml
spec:
  containers:
  - name: api
    envFrom:
    - secretRef:
        name: db-credentials       # ALL keys in this Secret become env vars
        optional: false
    - secretRef:
        name: api-credentials
    - configMapRef:
        name: app-config           # Mix with ConfigMap — common pattern
```

With `db-credentials` containing `username` and `password`, the container gets:
```
username=admin
password=MyS3cur3P@ssw0rd!
```

---

## Method 3: Mount Secret as a Volume (Recommended for Sensitive Files)

Mounting a Secret as a volume creates files inside the container. Each key becomes a filename, each value becomes the file content (decoded from base64 automatically):

```yaml
spec:
  containers:
  - name: api
    image: api-server:v2.0
    volumeMounts:
    - name: db-creds
      mountPath: /etc/secrets/db      # directory inside the container
      readOnly: true                  # always set readOnly for Secrets
  
  volumes:
  - name: db-creds
    secret:
      secretName: db-credentials      # Secret name
      defaultMode: 0400               # only owner can read — most restrictive
```

After mounting, the container has:
```
/etc/secrets/db/username   ← plain text "admin" (auto-decoded from base64)
/etc/secrets/db/password   ← plain text "MyS3cur3P@ssw0rd!"
```

The application reads these files:
```python
# Python application reading credentials from mounted Secret files
with open('/etc/secrets/db/password', 'r') as f:
    db_password = f.read().strip()

with open('/etc/secrets/db/username', 'r') as f:
    db_username = f.read().strip()
```

---

## Mounting Specific Keys with `items`

Mount only specific keys and control the filename and permissions:

```yaml
volumes:
- name: tls-certs
  secret:
    secretName: api-tls-secret
    defaultMode: 0400
    items:
    - key: tls.crt
      path: certificate.pem     # mounted as /etc/tls/certificate.pem
    - key: tls.key
      path: private-key.pem     # mounted as /etc/tls/private-key.pem
      mode: 0400                # most restrictive — private key
```

---

## Environment Variables vs Volume Mounts — Security Comparison

This is an important security decision:

| | Environment Variables | Volume Mounts |
|---|----------------------|---------------|
| **Visible to child processes** | ✅ Yes — all child processes inherit env vars | ❌ No — only processes that read the file |
| **Visible in crash dumps** | ✅ Often — env vars appear in stack traces | ❌ Less likely |
| **Visible in Docker inspect** | ✅ Yes (if using Docker-based runtimes) | ❌ No |
| **Update without restart** | ❌ No — Pod must restart | ✅ Yes — files update within ~60 seconds |
| **Logged accidentally** | ✅ Higher risk — `printenv` in a log statement | ❌ Lower risk |
| **Application compatibility** | ✅ Most apps support env vars | Requires file reading code |

**Recommendation:**
- For maximum security (TLS private keys, PKI credentials): use **volume mounts**
- For standard database passwords and API keys: environment variables are acceptable, but volume mounts are better
- For External Secrets Operator-synced secrets: environment variables work fine since the actual secret never touches the cluster in plain text

---

## Using Secrets for Database Connection Strings

A common pattern: combine ConfigMap (for hostname) and Secret (for password) in a connection string:

```yaml
spec:
  containers:
  - name: api
    env:
    # Host from ConfigMap (not sensitive)
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_PORT
    - name: DB_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_NAME
    # Password from Secret (sensitive)
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
```

The application builds the connection string at runtime:
```python
import os
db_url = f"postgresql://{os.environ['DB_USER']}:{os.environ['DB_PASSWORD']}@{os.environ['DB_HOST']}:{os.environ['DB_PORT']}/{os.environ['DB_NAME']}"
```

---

## TLS Certificates for Ingress

A TLS Secret is referenced directly by Ingress — no volume mount needed:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
spec:
  tls:
  - hosts:
    - api.mycompany.com
    secretName: api-tls-secret    # references the kubernetes.io/tls Secret
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

The Ingress controller (NGINX Ingress or AWS ALB) reads the Secret directly from the Kubernetes API using its ServiceAccount — no Pod mounting needed.

---

## ServiceAccount Tokens — Automatic Secret Injection

Every Pod gets a ServiceAccount, and every ServiceAccount gets a token automatically mounted at:

```
/var/run/secrets/kubernetes.io/serviceaccount/token    ← JWT token
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt   ← cluster CA certificate
/var/run/secrets/kubernetes.io/serviceaccount/namespace ← current namespace
```

This token is used when your application needs to talk to the Kubernetes API:

```python
# Reading the ServiceAccount token for Kubernetes API calls
with open('/var/run/secrets/kubernetes.io/serviceaccount/token', 'r') as f:
    token = f.read()

# Use in API requests
headers = {"Authorization": f"Bearer {token}"}
response = requests.get("https://kubernetes.default.svc/api/v1/namespaces", headers=headers)
```

For EKS IRSA (IAM Roles for Service Accounts), a projected service account token is injected at a configurable path and used to assume AWS IAM roles.

---

## Debugging Secret Injection Issues

```bash
# Pod failed to start — missing Secret
kubectl describe pod api-pod-abc123 -n production | grep -A 5 "Events:"
# Warning  Failed  CreateContainerConfigError: secret "db-credentials" not found

# Fix: check if Secret exists
kubectl get secret db-credentials -n production
# Error from server (NotFound): secrets "db-credentials" not found

# Create the missing Secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password="..." \
  -n production

# Pod failed — wrong key name
kubectl describe pod api-pod-abc123 -n production
# Warning: couldn't find key "password" in Secret production/db-credentials

# Fix: check what keys the Secret has
kubectl get secret db-credentials -n production -o yaml | grep "^  [a-z]"
# passwd: ...   ← it's "passwd" not "password"
```

---

## Summary

- `secretKeyRef` injects a single Secret key as one env var; `secretRef` in `envFrom` injects all keys as env vars
- Volume mounts are more secure than env vars — Secrets mounted as files are not inherited by child processes and are less likely to appear in crash dumps
- Set `readOnly: true` and `defaultMode: 0400` on Secret volume mounts — most restrictive permissions for sensitive files
- TLS Secrets are referenced directly by Ingress resources — no Pod mounting required
- Every Pod automatically gets a ServiceAccount token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`
- Debug missing Secrets with `kubectl describe pod` — Events section shows `CreateContainerConfigError` with the missing Secret or key name

In the next file, `04-secrets-best-practices.md`, you will learn the operational practices that keep Secrets secure in production — rotation, RBAC, audit logging, and what to avoid.
