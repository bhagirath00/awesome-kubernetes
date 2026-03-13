# Creating Secrets

## Introduction

Creating Secrets correctly means never exposing sensitive values in plain text in version control or command history. This file covers all the ways to create Secrets in Kubernetes, explains which methods are safe for production, and shows how to verify and manage them.

---

## Method 1: kubectl from Literal Values

```bash
# Create an Opaque Secret with literal key-value pairs
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='MyS3cur3P@ssw0rd!' \
  -n production

# Verify (values are hidden)
kubectl get secret db-credentials -n production

# NAME             TYPE     DATA   AGE
# db-credentials   Opaque   2      5s

# Describe also hides values
kubectl describe secret db-credentials -n production

# Data
# ====
# password:  20 bytes   ← length shown, not value
# username:  5 bytes
```

> ⚠️ **Warning:** Literal values typed in a terminal are stored in shell history (`~/.bash_history`). For production, read the password from a file or environment variable:

```bash
# Read password from a file (does not appear in history)
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-file=password=./db-password.txt \
  -n production

# Or use a subshell to avoid history
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password="$(cat db-password.txt)" \
  -n production
```

---

## Method 2: kubectl from Files

When your secret is a file (TLS certificate, private key, `.env` file):

```bash
# Create a TLS Secret from cert files
kubectl create secret tls api-tls \
  --cert=certs/api.mycompany.com.crt \
  --key=certs/api.mycompany.com.key \
  -n production

# Create an Opaque Secret from multiple files
kubectl create secret generic app-secrets \
  --from-file=db-password.txt \
  --from-file=api-key.txt \
  --from-file=jwt-secret.txt \
  -n production

# The key name is the filename (unless specified):
# Keys: db-password.txt, api-key.txt, jwt-secret.txt

# Custom key names:
kubectl create secret generic app-secrets \
  --from-file=DB_PASSWORD=db-password.txt \
  --from-file=API_KEY=api-key.txt \
  -n production
# Keys: DB_PASSWORD, API_KEY
```

---

## Method 3: YAML Manifest with `data` (Base64 Encoded)

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: YWRtaW4=                  # base64("admin")
  password: TXlTM2N1cjNQQHNzdzByZCE= # base64("MyS3cur3P@ssw0rd!")
```

Generate base64 values:
```bash
echo -n "admin" | base64
# YWRtaW4=

echo -n "MyS3cur3P@ssw0rd!" | base64
# TXlTM2N1cjNQQHNzdzByZCE=
```

```bash
kubectl apply -f secret.yaml
```

> ⚠️ **Warning:** Never commit `secret.yaml` with real secret values to Git. Git history is permanent — once a secret is in Git history, it is compromised even if you delete the file later. Use one of the tools in Method 5 instead.

---

## Method 4: YAML with `stringData` (Plain Text — Easier But Unsafe Alone)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
stringData:                      # plain text — do not commit to Git
  username: admin
  password: "MyS3cur3P@ssw0rd!"
```

`stringData` is write-only — when you `kubectl get secret -o yaml`, the values are always shown in `data` (base64 encoded), never in `stringData`.

`stringData` is useful in CI/CD pipelines where you substitute values from a secrets vault at deploy time:

```bash
# In Jenkins/GitHub Actions — secret values come from the CI secret store
# Never hardcoded in the YAML that gets committed to Git

# Template (committed to Git — no real values)
cat secret-template.yaml
# apiVersion: v1
# kind: Secret
# ...
# stringData:
#   password: "${DB_PASSWORD}"    ← placeholder

# At deploy time, substitute real values
envsubst < secret-template.yaml | kubectl apply -f -
# The actual value comes from the CI/CD secret store (GitHub Actions secrets, Jenkins credentials)
# The applied Secret contains the real value
# But the committed file only has placeholders — safe to commit
```

---

## Method 5: Using AWS Secrets Manager (Production-Grade)

For production on AWS, the real secret value lives in AWS Secrets Manager. The External Secrets Operator syncs it into a Kubernetes Secret automatically. You never write the actual secret value in any file or command.

See `05-external-secrets-operator.md` for the full setup.

---

## Updating a Secret

```bash
# Method 1: Re-apply YAML with updated value
kubectl apply -f updated-secret.yaml

# Method 2: Patch a specific key
NEW_PASSWORD=$(echo -n "NewP@ssword456" | base64)
kubectl patch secret db-credentials -n production \
  --type='merge' \
  -p="{\"data\":{\"password\":\"${NEW_PASSWORD}\"}}"

# Method 3: Edit directly (opens in editor)
kubectl edit secret db-credentials -n production
# Values are shown base64 encoded in the editor
# Edit and save to apply

# Verify the change (decode the value)
kubectl get secret db-credentials -n production \
  -o jsonpath='{.data.password}' | base64 --decode
# NewP@ssword456
```

---

## Viewing Secret Values Safely

```bash
# Safe: get the raw value of a specific key (decode from base64)
kubectl get secret db-credentials -n production \
  -o jsonpath='{.data.password}' | base64 --decode

# Safe: get the entire Secret as JSON and decode specific fields
kubectl get secret db-credentials -n production -o json | \
  jq -r '.data.password | @base64d'

# List all Secrets (does not expose values)
kubectl get secrets -n production

# Show all keys in a Secret (sizes shown, not values)
kubectl describe secret db-credentials -n production

# DO NOT do this in a shared session — shows all values
kubectl get secret db-credentials -n production -o yaml
```

---

## Restricting Secret Access with RBAC

By default in many clusters, any user with access to a namespace can read all Secrets in it. Restrict this explicitly:

```yaml
# Allow application Pods' ServiceAccounts to read only specific Secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-credentials", "api-keys"]  # only these Secrets
  verbs: ["get"]   # get only — not list (list exposes Secret names)
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-secret-reader
  namespace: production
subjects:
- kind: ServiceAccount
  name: api-server-sa
  namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Test: verify the service account can read only the allowed Secrets
kubectl auth can-i get secret/db-credentials -n production \
  --as=system:serviceaccount:production:api-server-sa
# yes

kubectl auth can-i get secret/other-secret -n production \
  --as=system:serviceaccount:production:api-server-sa
# no

kubectl auth can-i list secrets -n production \
  --as=system:serviceaccount:production:api-server-sa
# no   ← cannot enumerate Secrets
```

---

## Summary

- Create Secrets with `kubectl create secret generic` for simple cases — avoid terminal history by reading values from files
- YAML manifests with `data` (base64) or `stringData` (plain text) are convenient but must never be committed to Git with real values
- In CI/CD pipelines, use `envsubst` or templating to substitute real values at deploy time — commit only placeholder templates
- For production, use AWS Secrets Manager + External Secrets Operator — actual secret values never appear in Kubernetes YAML files or CI/CD logs
- Restrict Secret access with RBAC — use `resourceNames` to limit access to specific Secrets only, and deny `list` to prevent enumeration

In the next file, `03-using-secrets-in-pods.md`, you will learn how to inject Secrets into Pods as environment variables and as files — with the same patterns as ConfigMaps but with additional security considerations.
