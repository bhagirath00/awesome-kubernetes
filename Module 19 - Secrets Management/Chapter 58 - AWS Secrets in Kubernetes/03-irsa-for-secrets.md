# IAM Roles for Service Accounts (IRSA) for Secure Secrets Access

## 1. Architectural Concept & Deep Dive

The greatest security vulnerability in cloud-native applications is hardcoding static cloud provider credentials:
- Storing `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in Kubernetes Secrets or container environment variables.
- Sharing a broad IAM role attached to the EC2 worker node instances (granting all pods on that node full access to S3, KMS, and Secrets Manager).

**IAM Roles for Service Accounts (IRSA)** eliminates static cloud credentials entirely by establishing a cryptographic bridge between the **Kubernetes Control Plane (OIDC Issuer)** and **AWS Identity and Access Management (AWS IAM)**.

```
+─────────────────────────────────────────────────────────────────────────+
|                           AWS IRSA ARCHITECTURE                         |
|                                                                         |
|  [ Microservice Pod ]                                                   |
|  ServiceAccount: "billing-sa" (Annotated: eks.amazonaws.com/role-arn)   |
|         │                                                               |
|         ├── Injected: AWS_WEB_IDENTITY_TOKEN_FILE (JWT token)           |
|         ▼                                                               |
|  [ AWS SDK (Boto3 / Go AWS SDK / Java SDK) ]                           |
|         │                                                               |
|         ├── 1. Calls: sts:AssumeRoleWithWebIdentity                     |
|         │      Passes JWT token + Target Role ARN                       |
|         ▼                                                               |
|  [ AWS Security Token Service (STS) ]                                   |
|         │                                                               |
|         ├── 2. Verifies JWT signature against EKS OIDC Provider         |
|         ├── 3. Validates Trust Policy:                                  |
|         │      • Audience: "sts.amazonaws.com"                          |
|         │      • Subject: "system:serviceaccount:production:billing-sa" |
|         ▼                                                               |
|  [ AWS STS returns Temporary IAM Credentials (TTL: 15 min - 1 hour) ]   |
|         │                                                               |
|         ▼                                                               |
|  Pod accesses AWS Secrets Manager / KMS / S3 securely!                  |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Core Security Guarantees of IRSA

1. **Principle of Least Privilege**: Each individual Pod receives only the exact IAM permissions it requires (e.g., Billing Pod can read only `/production/billing/*` secrets; Order Pod cannot).
2. **Credential Isolation**: Pods running on the same worker node cannot access each other's cloud credentials.
3. **Zero Static Credentials**: No AWS access keys exist anywhere on disk, in git, or in etcd.
4. **Automated Cryptographic Rotation**: AWS STS issues ephemeral credentials rotated continuously in memory.

---

## 3. Step-by-Step Implementation Lab

### Step 1: Enable IAM OIDC Provider for Your EKS Cluster
An EKS cluster requires an associated IAM OIDC identity provider:

```bash
# Associate OIDC provider with your cluster using eksctl
eksctl utils associate-iam-oidc-provider \
  --cluster prod-cluster-us-east-1 \
  --approve
```

### Step 2: Create the IAM Role Trust Policy (`trust-policy.json`)
The trust policy restricts assumption exclusively to a specific Kubernetes ServiceAccount:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:production:billing-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

### Step 3: Create the IAM Role & Attach Permissions
```bash
# 1. Create the IAM Role
aws iam create-role \
  --role-name k8s-billing-secrets-role \
  --assume-role-policy-document file://trust-policy.json

# 2. Attach policy granting access to specific AWS Secrets
aws iam attach-role-policy \
  --role-name k8s-billing-secrets-role \
  --policy-arn arn:aws:iam::123456789012:policy/BillingSecretsReadPolicy
```

### Step 4: Annotate the Kubernetes ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: billing-sa
  namespace: production
  annotations:
    # Binds the Kubernetes ServiceAccount to the AWS IAM Role!
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/k8s-billing-secrets-role
```

---

## 4. How the Pod Consumes IRSA Under the Hood

When a pod is scheduled with `serviceAccountName: billing-sa`, the EKS Pod Identity Webhook automatically injects three components into the container:

1. **Environment Variable**: `AWS_ROLE_ARN=arn:aws:iam::123456789012:role/k8s-billing-secrets-role`
2. **Environment Variable**: `AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token`
3. **Projected Volume**: An OIDC JWT token written to that file path, signed by the Kubernetes cluster CA.

Standard AWS SDKs (Boto3 in Python, AWS SDK for Go, AWS Java SDK v2) automatically detect these environment variables and call `sts:AssumeRoleWithWebIdentity` with **zero custom authentication code**!

---

## 5. Verification & Troubleshooting Runbook

### Diagnostic Commands
```bash
# Verify environment variables were injected into the running pod
kubectl exec -it deployment/billing-app -n production -- env | grep AWS_
```
Expected output:
```
AWS_ROLE_ARN=arn:aws:iam::123456789012:role/k8s-billing-secrets-role
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

Verify AWS caller identity from inside the pod:
```bash
kubectl exec -it deployment/billing-app -n production -- aws sts get-caller-identity
```
```json
{
    "UserId": "AROA1234EXAMPLE:botocore-session-123",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/k8s-billing-secrets-role/botocore-session-123"
}
```

### Common Pitfalls & Solutions
1. **`An error occurred (AccessDenied) when calling the AssumeRoleWithWebIdentity operation`**:
   - Cause: Typo in the IAM trust policy condition.
   - Solution: Verify `sub` string matches `system:serviceaccount:<NAMESPACE>:<SERVICEACCOUNT_NAME>` exactly.
2. **Environment variables `AWS_ROLE_ARN` are missing from container**:
   - Cause: Pod was created before the ServiceAccount was annotated, or EKS Pod Identity Webhook is failing.
   - Solution: Restart deployment via `kubectl rollout restart deployment/<NAME>`.

---

## 6. Key Takeaways
- IRSA eliminates static cloud credentials by mapping Kubernetes ServiceAccounts directly to AWS IAM Roles.
- Uses Kubernetes-projected OIDC JWT tokens and AWS STS `AssumeRoleWithWebIdentity`.
- Enforces strict zero-trust boundaries: pods on the same node only receive permissions scoped to their specific service account.
- Seamlessly supported by all modern AWS SDKs, CSI Drivers, and the External Secrets Operator.
