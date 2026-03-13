# External Secrets Operator

## Introduction

Native Kubernetes Secrets have a fundamental problem: you have to put the actual secret value somewhere to create them. If you put them in YAML and commit to Git, the secret is exposed. If you type them in the terminal, they appear in shell history.

The **External Secrets Operator (ESO)** solves this by reading secrets from external systems — AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, GCP Secret Manager — and automatically creating and updating Kubernetes Secrets from them. Your Pods use standard Kubernetes Secrets. The actual values live in AWS Secrets Manager, never in Git or in kubectl commands.

---

## How External Secrets Operator Works

```
AWS Secrets Manager
  /myapp/production/db-password = "SuperSecret123!"   ← real value here
         │
         │ ESO polls for changes every N seconds
         ▼
External Secrets Operator (running in kube-system)
         │
         │ creates/updates
         ▼
Kubernetes Secret: db-credentials
  data:
    password: U3VwZXJTZWNyZXQxMjMh  ← base64 of the real value
         │
         │ injected into
         ▼
Pod: API server container
  ENV: DB_PASSWORD=SuperSecret123!
```

The developer workflow:
1. Store the real secret value in AWS Secrets Manager (one time)
2. Create an `ExternalSecret` resource in Kubernetes (committed to Git — contains no sensitive values)
3. ESO reads from Secrets Manager and creates the Kubernetes Secret automatically
4. Pods use the Kubernetes Secret as normal

---

## Installing External Secrets Operator on EKS

```bash
# Install via Helm
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set installCRDs=true

# Verify installation
kubectl get pods -n external-secrets
# NAME                                                READY   STATUS    RESTARTS
# external-secrets-abc123                             1/1     Running   0
# external-secrets-cert-controller-def456            1/1     Running   0
# external-secrets-webhook-ghi789                    1/1     Running   0
```

---

## Setting Up IRSA for ESO — IAM Permissions

The External Secrets Operator needs permission to read from AWS Secrets Manager. Use IRSA (IAM Roles for Service Accounts) — no static AWS credentials needed:

```bash
# Step 1: Create the IAM policy for Secrets Manager access
cat > eso-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789:secret:myapp/*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ExternalSecretsPolicy \
  --policy-document file://eso-policy.json

# Step 2: Create the IAM role with trust relationship for the ESO ServiceAccount
eksctl create iamserviceaccount \
  --name external-secrets-sa \
  --namespace external-secrets \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::123456789:policy/ExternalSecretsPolicy \
  --approve \
  --region us-east-1
```

---

## Configuring a SecretStore

A `SecretStore` defines how ESO connects to the external secrets provider. It is namespace-scoped:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa    # the IRSA ServiceAccount
```

For cluster-wide usage (one SecretStore for all namespaces), use `ClusterSecretStore`:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager-cluster
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

---

## Creating an ExternalSecret

An `ExternalSecret` tells ESO which key to read from Secrets Manager and which Kubernetes Secret to create:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h          # how often ESO checks for updates in Secrets Manager
  
  secretStoreRef:
    name: aws-secrets-manager  # which SecretStore to use
    kind: SecretStore
  
  target:
    name: db-credentials       # name of the Kubernetes Secret to create
    creationPolicy: Owner      # ESO owns this Secret — deletes it when ExternalSecret is deleted
  
  data:
  - secretKey: password        # key in the resulting Kubernetes Secret
    remoteRef:
      key: myapp/production/db-credentials  # path in AWS Secrets Manager
      property: password                    # JSON key within the secret value
  
  - secretKey: username
    remoteRef:
      key: myapp/production/db-credentials
      property: username
```

In AWS Secrets Manager, `myapp/production/db-credentials` stores:
```json
{
  "username": "admin",
  "password": "SuperSecret123!"
}
```

ESO reads this and creates a Kubernetes Secret with `username` and `password` keys.

---

## Syncing an Entire Secret

If your Secrets Manager secret is a JSON object, sync all keys at once with `dataFrom`:

```yaml
spec:
  dataFrom:
  - extract:
      key: myapp/production/app-secrets    # JSON object in Secrets Manager
      # All JSON keys become Secret keys automatically
      # {"DB_PASSWORD":"...", "API_KEY":"...", "JWT_SECRET":"..."}
      # → Kubernetes Secret keys: DB_PASSWORD, API_KEY, JWT_SECRET
```

This is the most convenient approach — one ExternalSecret for all your application's secrets.

---

## Storing Secrets in AWS Secrets Manager

```bash
# Create a secret in AWS Secrets Manager
aws secretsmanager create-secret \
  --name "myapp/production/app-secrets" \
  --description "Production application secrets" \
  --secret-string '{
    "DB_PASSWORD": "SuperSecret123!",
    "API_KEY": "sk_live_abc123def456",
    "JWT_SECRET": "super-long-random-string-for-jwt"
  }' \
  --region us-east-1

# Enable automatic rotation (for RDS)
aws secretsmanager rotate-secret \
  --secret-id "myapp/production/db-credentials" \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789:function:SecretsManagerRDSRotation \
  --rotation-rules AutomaticallyAfterDays=30

# Update a secret value
aws secretsmanager update-secret \
  --secret-id "myapp/production/app-secrets" \
  --secret-string '{"DB_PASSWORD": "NewPassword456!", "API_KEY": "sk_live_abc123def456"}'
# ESO will detect the new version and update the Kubernetes Secret within refreshInterval
```

---

## Verifying ESO Is Working

```bash
# Check ExternalSecret status
kubectl get externalsecret -n production

# NAME             STORE                   REFRESH INTERVAL   STATUS   READY
# db-credentials   aws-secrets-manager     1h                 Valid    True

# Get detailed status (shows last sync time and any errors)
kubectl describe externalsecret db-credentials -n production

# Status:
#   Conditions:
#     Last Transition Time:  2024-01-15T10:00:00Z
#     Message:               1 secrets synced
#     Status:                True
#     Type:                  Ready

# Verify the Kubernetes Secret was created
kubectl get secret db-credentials -n production

# NAME             TYPE     DATA   AGE
# db-credentials   Opaque   2      5m

# Verify the Secret has the right keys (not the values — never print values)
kubectl get secret db-credentials -n production \
  -o jsonpath='{.data}' | python3 -c "import sys, json; print(list(json.load(sys.stdin).keys()))"
# ['password', 'username']

# If ESO failed to sync, check events:
kubectl describe externalsecret db-credentials -n production | grep -A 5 Events
# Warning  SecretSyncedError  AccessDeniedException: User: arn:aws:sts:...
# → IRSA permission issue — check IAM role policy
```

---

## Force a Secret Refresh

```bash
# Trigger an immediate sync (useful after rotating a secret in Secrets Manager)
kubectl annotate externalsecret db-credentials \
  -n production \
  force-sync="$(date +%s)" \
  --overwrite

# ESO immediately fetches the latest value from Secrets Manager and updates the Kubernetes Secret
```

---

## The Complete Production Pattern

```
Git Repository (safe to commit):
├── externalsecret.yaml          ← ExternalSecret definition — no real values
├── clusterSecretStore.yaml      ← how to connect to Secrets Manager
└── deployment.yaml              ← references Kubernetes Secret name only

AWS Secrets Manager (not in Git):
└── myapp/production/app-secrets
    ├── DB_PASSWORD = "SuperSecret123!"     ← real values here only
    ├── API_KEY = "sk_live_abc123..."
    └── JWT_SECRET = "long-random-string"

Kubernetes (auto-managed by ESO):
└── Secret: app-secrets
    ├── DB_PASSWORD = <synced from Secrets Manager>
    ├── API_KEY = <synced from Secrets Manager>
    └── JWT_SECRET = <synced from Secrets Manager>

Pod:
└── env DB_PASSWORD from secret/app-secrets (standard Kubernetes Secret)
```

---

## Summary

- External Secrets Operator reads real secret values from AWS Secrets Manager and automatically creates and updates Kubernetes Secrets
- Developers commit `ExternalSecret` resources to Git (no real values) — the real values stay in AWS Secrets Manager
- ESO uses IRSA for passwordless authentication to AWS Secrets Manager — no static AWS credentials needed
- `SecretStore` configures the connection to the external provider; `ExternalSecret` maps a Secrets Manager key to a Kubernetes Secret key
- `dataFrom.extract` syncs an entire JSON secret from Secrets Manager into a Kubernetes Secret — convenient for bulk-loading application secrets
- Force a sync with `kubectl annotate externalsecret` after rotating a secret in AWS Secrets Manager
- With ESO + AWS Secrets Manager rotation Lambdas, secrets rotate automatically with zero Kubernetes changes needed

This completes Chapter 10 — Secrets. You now have a complete understanding of how to handle sensitive data in Kubernetes, from basic Secret creation through the production-grade External Secrets Operator pattern.

In Chapter 11, `01-resource-requests.md`, you will learn how to specify resource requests for Pods — the foundation of Kubernetes scheduling and quality of service guarantees.
