# AWS SSM Parameter Store Integration: Cost-Effective Configuration & Secrets

## 1. Architectural Concept & Deep Dive

While AWS Secrets Manager is ideal for credentials requiring automated rotation (RDS passwords, API keys), **AWS Systems Manager (SSM) Parameter Store** is the industry standard for hierarchical configuration management and low-cost secret storage.

### Cost Architecture: Secrets Manager vs SSM Parameter Store

| Metric / Dimension | AWS Secrets Manager | AWS SSM Parameter Store |
| :--- | :--- | :--- |
| **Pricing** | **$0.40 per secret / month** + $0.05 per 10k API calls | **$0.00 (FREE)** for Standard Parameters (up to 10,000 parameters!) |
| **Secret Types** | JSON or Plaintext string | `String`, `StringList`, and **`SecureString`** (KMS Encrypted) |
| **Automated Rotation**| Built-in native Lambda rotation | Requires custom EventBridge + Lambda |
| **Hierarchy** | Flat naming (or path-like tags) | **Native Tree Hierarchy** (`/app/env/component/key`) |
| **Production Fit** | High-value database credentials, auto-rotated third-party tokens | Environment configs, feature flags, microservice settings, low-churn secrets |

In a cluster with 500 microservices storing 10 config values each:
- Using Secrets Manager: $2,000 / month in AWS bills.
- Using SSM Parameter Store: **$0.00 / month**.

```
+─────────────────────────────────────────────────────────────────────────+
|               SSM PARAMETER STORE HIERARCHY IN K8S                      |
|                                                                         |
|  AWS SSM Parameter Store Hierarchy:                                     |
|  /production/                                                           |
|  ├── payment-service/                                                   |
|  │   ├── DB_HOST (String: db.prod.internal)                            |
|  │   └── DB_PASSWORD (SecureString: KMS Encrypted)                      |
|  └── order-service/                                                     |
|      ├── LOG_LEVEL (String: INFO)                                       |
|      └── STRIPE_KEY (SecureString: KMS Encrypted)                       |
|                                                                         |
|                                    │                                    |
|                                    ▼                                    |
|               [ External Secrets Operator / CSI Driver ]                |
|                                    │                                    |
|         ┌──────────────────────────┴──────────────────────────┐         |
|         ▼                                                     ▼         |
|   Standard Parameter (String)                    SecureString (KMS)     |
|   Synced to Kubernetes ConfigMap                 Decrypted & Synced to  |
|                                                  Kubernetes Secret      |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Pattern A: Secrets Store CSI Driver with SSM

Configure the `SecretProviderClass` to fetch parameters from SSM using `objectType: "ssmparameter"`:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: ssm-config-provider
  namespace: production
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "/production/payment/DB_HOST"
        objectType: "ssmparameter"
        objectAlias: "db_host"
      - objectName: "/production/payment/DB_PASSWORD"
        objectType: "ssmparameter"
        objectAlias: "db_password"
```

---

## 3. Pattern B: External Secrets Operator (ESO) with SSM

ESO provides native support for pulling entire parameter hierarchies at once using `dataFrom.find.path`:

### Step 1: Define `SecretStore` for SSM
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-ssm-store
  namespace: production
spec:
  provider:
    aws:
      service: ParameterStore
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-irsa-serviceaccount
```

### Step 2: Fetch Hierarchical Parameters Automatically
Instead of listing individual keys, pull all parameters residing under `/production/payment/`:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-ssm-secrets
  namespace: production
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: aws-ssm-store
    kind: SecretStore
  target:
    name: payment-config-secret
    creationPolicy: Owner
  dataFrom:
    # Recursively find all parameters under the application path!
    - find:
        path: "/production/payment"
```
- If an engineer adds `/production/payment/REDIS_URL` in the AWS console, ESO automatically discovers it and adds it to the Kubernetes `Secret` during the next sync cycle!

---

## 4. Securing SecureString Parameters with Customer Managed Keys (CMK)

By default, SSM uses the default AWS-managed KMS key (`alias/aws/ssm`). 
For SOC 2 and PCI compliance, configure a dedicated **Customer Managed Key (CMK)**:

```bash
# 1. Create a parameter using a custom KMS Key
aws ssm put-parameter \
  --name "/production/payment/DB_PASSWORD" \
  --value "SuperSecretPassword123" \
  --type "SecureString" \
  --key-id "arn:aws:kms:us-east-1:123456789012:key/abc-123-def" \
  --overwrite
```

### IAM Policy Required for the Kubernetes Pod:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/production/payment/*"
    },
    {
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/abc-123-def"
    }
  ]
}
```

---

## 5. Verification & Troubleshooting Runbook

### Diagnostic Commands
```bash
# Check status of ExternalSecret synchronization
kubectl get externalsecret payment-ssm-secrets -n production

# Inspect ESO controller logs if synchronization fails
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets --tail=50
```

### Common Pitfalls & Solutions
1. **`AccessDeniedException: User is not authorized to perform: kms:Decrypt`**:
   - Cause: The IRSA IAM role has permission to read SSM, but lacks `kms:Decrypt` permission on the specific KMS key used to encrypt the `SecureString`.
   - Solution: Add the KMS key ARN to the IAM role policy statement.
2. **SSM API Throttling (`ThrottlingException`)**:
   - Cause: Standard SSM Parameter Store has a default throughput limit of 40 requests/sec per account.
   - Solution: Enable **Higher Throughput** in AWS SSM Settings (increases limit to 10,000 req/sec) and increase ESO `refreshInterval` to 1 hour.

---

## 6. Key Takeaways
- SSM Parameter Store is a cost-effective alternative to Secrets Manager ($0 for standard tier).
- `SecureString` parameters are encrypted at rest using AWS KMS.
- ESO's `find.path` directive allows auto-discovering entire parameter hierarchies (`/production/app/*`).
- Always grant both `ssm:GetParameter` and `kms:Decrypt` in the pod's IRSA role when accessing `SecureString` parameters.
