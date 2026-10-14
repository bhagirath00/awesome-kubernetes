# Secrets Best Practices

## Introduction

Secrets require much more careful handling than regular configuration. A single mistake — committing a secret to Git, giving too-broad RBAC access, or not rotating credentials — can lead to a security breach. This file covers the proven practices that production DevOps teams apply to keep Secrets secure throughout the application lifecycle.

---

## Practice 1: Never Store Secrets in Git

This is the most important rule. Once a secret value is committed to a Git repository — even in a private repo, even in a deleted file — it must be considered compromised:

```bash
# This is what you should NEVER do:
git add secret.yaml   # contains real password
git commit -m "Add database credentials"
git push

# Even if you immediately delete and force-push:
git log --all  # the commit still exists in local clone history
# Anyone who cloned the repo before the delete has the password
```

### What To Do Instead

**Option A: Never write secrets to files — create from stdin or flags:**
```bash
kubectl create secret generic db-credentials \
  --from-literal=password="$(read -s -p 'Password: ' p && echo $p)"
```

**Option B: Use SOPS (Secrets OPerationS) to encrypt YAML before committing:**
```bash
# Encrypt secret.yaml with AWS KMS before committing
sops --encrypt --kms arn:aws:kms:us-east-1:123456789:key/abc-123 \
  secret.yaml > secret.enc.yaml

# Commit only the encrypted version:
git add secret.enc.yaml   # safe to commit — encrypted with KMS

# At deploy time (CI/CD has KMS decrypt permission):
sops --decrypt secret.enc.yaml | kubectl apply -f -
```

**Option C: External Secrets Operator (recommended for AWS):**
Store real values in AWS Secrets Manager. Only sync references to Kubernetes. Covered in `05-external-secrets-operator.md`.

---

## Practice 2: Apply Least-Privilege RBAC for Secrets

By default, namespace admins can read all Secrets. Restrict this:

```yaml
# Only allow specific ServiceAccounts to read specific Secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames:
  - "db-credentials"          # only these named Secrets
  - "api-keys"
  verbs: ["get"]              # get only — NOT list, create, update, delete

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-server-secrets
  namespace: production
subjects:
- kind: ServiceAccount
  name: api-server-sa
  namespace: production
roleRef:
  kind: Role
  name: api-secret-reader
  apiGroup: rbac.authorization.k8s.io
```

Important: never grant `list` on `secrets` unless absolutely necessary. Listing Secrets returns all Secret names (metadata) — useful for an attacker to know which secrets exist.

---

## Practice 3: Enable Kubernetes Audit Logging for Secret Access

Know who is reading your Secrets. AWS EKS supports audit logging to CloudWatch:

```bash
# Enable EKS audit logging (include authenticator and audit logs)
aws eks update-cluster-config \
  --name my-cluster \
  --region us-east-1 \
  --logging '{"clusterLogging":[{"types":["audit"],"enabled":true}]}'
```

Query audit logs for Secret access:
```bash
# CloudWatch Insights query — who read db-credentials in the last 7 days?
fields @timestamp, user.username, requestURI, verb
| filter requestURI like "/api/v1/namespaces/production/secrets/db-credentials"
| filter verb in ["get", "watch"]
| sort @timestamp desc
```

Set up a CloudWatch alarm for unexpected Secret access (access from unexpected users or from outside business hours).

---

## Practice 4: Rotate Secrets Regularly

Secret rotation means changing the secret value periodically (or immediately after a suspected breach) without downtime:

### Manual Rotation Procedure

```bash
# Step 1: Update the external credential (e.g., rotate the RDS password)
aws rds modify-db-instance \
  --db-instance-identifier myapp-prod \
  --master-user-password "NewP@ssword789!" \
  --apply-immediately

# Step 2: Update the Kubernetes Secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='NewP@ssword789!' \
  -n production \
  --dry-run=client -o yaml | kubectl apply -f -

# Step 3: Rolling restart Pods to pick up new password
kubectl rollout restart deployment/api-server -n production

# Step 4: Verify new Pods are healthy
kubectl rollout status deployment/api-server -n production
```

### Automated Rotation with External Secrets Operator

When using AWS Secrets Manager + External Secrets Operator, rotation is automatic:
1. AWS Secrets Manager rotates the RDS password (built-in rotation Lambda)
2. External Secrets Operator detects the new version in Secrets Manager
3. The Kubernetes Secret is updated automatically within the configured `refreshInterval`
4. (Optional) A deployment annotation triggers an automatic rollout when the Secret changes

---

## Practice 5: Use Immutable Secrets in Production

Like immutable ConfigMaps, immutable Secrets cannot be modified — preventing accidental or malicious changes:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v2
  namespace: production
type: Opaque
immutable: true         # cannot be modified — delete and recreate with new name
stringData:
  username: admin
  password: "MyS3cur3P@ssw0rd!"
```

With immutable Secrets, adopt versioned names: `db-credentials-v1`, `db-credentials-v2`. Update the Deployment to reference the new version — triggering a rollout.

---

## Practice 6: Never Echo or Log Secret Values

This seems obvious but is a common mistake:

```bash
# WRONG — password visible in terminal and logs
kubectl exec api-pod -n production -- printenv DB_PASSWORD

# WRONG — password in application startup log
# In your app code — do NOT do this:
# logger.info(f"Connecting to {DB_HOST} with password {DB_PASSWORD}")

# RIGHT — log that configuration was loaded, not the values
# logger.info(f"Connecting to {DB_HOST} (credentials loaded from environment)")
```

In CI/CD pipelines, mask any environment variables that contain secret values in your pipeline configuration (GitHub Actions, Jenkins credentials).

---

## Practice 7: Secrets Security Checklist

Use this checklist when deploying any production workload:

```
[ ] Secrets are NOT stored in Git (use External Secrets, SOPS, or Sealed Secrets)
[ ] Envelope encryption enabled on EKS cluster (KMS key for etcd)
[ ] RBAC restricts Secret get/list to only required ServiceAccounts and users
[ ] resourceNames in RBAC rules limit access to specific Secret names
[ ] Audit logging enabled — alerts configured for unexpected Secret access
[ ] Application does not log secret values
[ ] Rotation schedule defined (or automated via AWS Secrets Manager)
[ ] Secrets mounted as files use readOnly: true and defaultMode: 0400
[ ] Secret values are not in environment variables visible to child processes
    (if using volume mounts — preferred for high-sensitivity secrets)
[ ] No secrets in ConfigMaps (verify by auditing all ConfigMaps)
```

---

## Auditing Existing Secrets for Security Issues

```bash
# Find all Secrets in all namespaces
kubectl get secrets --all-namespaces

# Look for Secrets with unusually large data (potential plaintext file storage)
kubectl get secrets --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): \(.data | to_entries[] | "\(.key)=\(.value | length) chars")"'

# Check if any ConfigMaps have data that looks like passwords (audit for misplacement)
kubectl get configmaps --all-namespaces -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)" as $cm | .data | to_entries[] | select(.key | test("password|secret|key|token"; "i")) | "\($cm): \(.key)"'
```

---

## Summary

- Never commit secrets to Git — use SOPS for encryption, External Secrets Operator for AWS Secrets Manager integration, or Sealed Secrets for GitOps
- Apply RBAC with `resourceNames` to limit Secret access to specific Secrets only; deny `list` to prevent enumeration
- Enable EKS audit logging and set up CloudWatch alerts for unexpected Secret access
- Rotate Secrets regularly — automate rotation with AWS Secrets Manager rotation Lambdas + External Secrets Operator for zero-downtime rotation
- Use immutable Secrets in production and version the names (`db-credentials-v2`) — prevents accidental changes
- Never log or echo secret values in application code, shell scripts, or CI/CD pipelines

In the next file, `05-external-secrets-operator.md`, you will learn how to integrate Kubernetes with AWS Secrets Manager using the External Secrets Operator — the production-grade pattern that keeps actual secret values out of your cluster entirely.
