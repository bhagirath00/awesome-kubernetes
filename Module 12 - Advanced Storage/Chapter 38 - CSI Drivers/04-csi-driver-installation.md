# CSI Driver Installation and IAM Security

## The Permission Problem: Why CSI Drivers Need AWS Credentials

Every CSI controller plugin must talk to AWS APIs:

- **EBS driver calls:** CreateVolume, AttachVolume, DeleteVolume
- **EFS driver calls:** CreateAccessPoint, DeleteAccessPoint

These are powerful operations. Without proper access controls, a compromised pod could:
- Steal data (mount any EBS volume)
- Delete infrastructure (destroy volumes)
- Cause denial-of-service (delete all volumes)
- Impersonate other applications

### The Security Principle: Least Privilege

**Least Privilege:** Every service gets only the minimum permissions it needs, no more.

By default, containers in Kubernetes have **no permissions** to call AWS APIs. This is good for security—pods can't accidentally or maliciously modify AWS infrastructure.

The challenge: the CSI controller pods **do need** permissions to provision volumes. We must grant them carefully.

---

## Traditional Approach (Problematic)

An older approach was to attach AWS credentials as a secret mounted in the pod:

```yaml
# ANTI-PATTERN: Don't do this
env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef:
        name: aws-creds
        key: access-key
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: aws-creds
        key: secret-key
```

**Problems:**
1. **Plaintext storage**: Credentials in etcd (Kubernetes database) are encrypted but still vulnerable
2. **Shared secret**: If one pod compromises the secret, all CSI drivers lose security
3. **No fine-grained control**: All pods with the secret have full AWS access
4. **Credential rotation**: Changing credentials requires pod restarts
5. **Audit trail**: Hard to trace which pod made which API call

---

## Modern Approach: IAM Roles for Service Accounts (IRSA)

AWS and Kubernetes developed **IRSA** to solve this.

### How IRSA Works: The Security Architecture

IRSA leverages OpenID Connect (OIDC):

1. **EKS creates an OIDC provider** representing your cluster
2. **Kubernetes can sign tokens** identifying specific ServiceAccounts
3. **AWS trusts these signed tokens**
4. **Tokens are automatically mounted** into pods

**The Flow:**

```
1. Admin creates IAM Role with:
   └─ Trust relationship: "Trust my cluster's OIDC provider"
   └─ Attached policy: "AmazonEBSCSIDriverPolicy"

2. Admin creates Kubernetes ServiceAccount with annotation:
   └─ "eks.amazonaws.com/role-arn": arn:aws:iam::....:role/MyCSIRole

3. When pod starts, kubelet mounts a projected token:
   └─ /var/run/secrets/eks.amazonaws.com/serviceaccount/token
   └─ This token is signed by the EKS OIDC provider

4. CSI pod calls AWS API with this token:
   └─ "Here's my token proving I'm the EBS CSI controller"

5. AWS STS (Security Token Service) validates token:
   └─ "I recognize this cluster's OIDC signature"
   └─ "The token claims to be the ebs-csi-controller-sa ServiceAccount"
   └─ "My trust policy allows this ServiceAccount"
   └─ Returns temporary credentials (15 min expiry)

6. CSI driver calls AWS APIs with temporary credentials:
   └─ "Create volume, attach volume, etc."

7. Temporary credentials expire automatically:
   └─ No credential rotation overhead
   └─ No static secrets to manage
```

### Why IRSA is Superior

**Benefit 1: No Static Secrets**
- No access keys stored anywhere
- Credentials are temporary and short-lived
- One less thing for attackers to steal

**Benefit 2: Fine-Grained Control**
- EBS driver gets only EBS permissions
- EFS driver gets only EFS permissions
- Other pods don't get storage permissions

**Benefit 3: Audit Trail**
- AWS knows which ServiceAccount made each API call
- Complete audit trail in CloudTrail
- Perfect for compliance

**Benefit 4: Credential Rotation Unnecessary**
- Temporary credentials expire in 15 minutes
- Automatically refreshed
- No pod restarts needed

---

## Setting Up IRSA: The Step-by-Step Process

### Step 1: Verify OIDC Provider Exists

```bash
# On EKS, OIDC provider is automatically created
# Verify it exists:
aws iam list-open-id-connect-providers

# Output should include: arn:aws:iam::111122223333:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLEID

# If missing, create it:
eksctl utils associate-iam-oidc-provider \
  --cluster=my-cluster \
  --approve
```

### Step 2: Create IAM Role with Trust Relationship

AWS provides managed policies for CSI drivers:

**For EBS CSI:**
```bash
# Using eksctl (recommended):
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster my-cluster \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve \
    --role-only \
    --role-name AmazonEKS_EBS_CSI_DriverRole

# This creates the role with the trust relationship automatically
```

**What this does:**
1. Creates IAM Role: `AmazonEKS_EBS_CSI_DriverRole`
2. Attaches AWS-managed policy: `AmazonEBSCSIDriverPolicy`
3. Creates trust relationship allowing OIDC provider to assume the role
4. Creates Kubernetes ServiceAccount: `ebs-csi-controller-sa`
5. Annotates it with the role ARN

### Step 3: Install Driver with the Service Account

#### Option A: Using EKS Add-ons (Recommended)

EKS can manage the CSI driver for you:

```bash
aws eks create-addon \
    --cluster-name my-cluster \
    --addon-name aws-ebs-csi-driver \
    --service-account-role-arn arn:aws:iam::111122223333:role/AmazonEKS_EBS_CSI_DriverRole
```

Advantages:
- AWS manages versioning and updates
- Automatic compatibility checks
- One-command installation

Verify:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver

# Should show:
# ebs-csi-controller-xxxxxxx (Running)
# ebs-csi-node-xxxx (Running on each node)
```

#### Option B: Using Helm (For Non-EKS Clusters)

For self-managed Kubernetes on EC2:

```bash
# Add Helm repository
helm repo add aws-ebs-csi-driver \
  https://kubernetes-sigs.github.io/aws-ebs-csi-driver

helm repo update

# Install with IRSA annotation
helm install aws-ebs-csi-driver \
  --namespace kube-system \
  aws-ebs-csi-driver/aws-ebs-csi-driver \
  --set serviceAccount.create=true \
  --set serviceAccount.name=ebs-csi-controller-sa \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::111122223333:role/MyRole"
```

### Step 4: Create StorageClass

Once driver is installed:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
```

---

## Special Case: KMS Encryption with Custom Keys

If your StorageClass uses custom KMS encryption:

```yaml
parameters:
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:111122223333:key/12345678-1234-1234-1234-123456789012"
```

The default `AmazonEBSCSIDriverPolicy` **does not** include KMS permissions. You must add an inline policy:

```bash
# Create policy document
cat > kms-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:CreateGrant",
        "kms:Decrypt",
        "kms:DescribeKey",
        "kms:Encrypt",
        "kms:GenerateDataKey",
        "kms:ListGrants",
        "kms:ReEncrypt*"
      ],
      "Resource": "arn:aws:kms:us-east-1:111122223333:key/12345678-1234-1234-1234-123456789012"
    }
  ]
}
EOF

# Attach to role
aws iam put-role-policy \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --policy-name KMSAccess \
  --policy-document file://kms-policy.json
```

---

## Troubleshooting: Diagnosing IAM Permission Issues

### Symptom: PVC Stuck in Pending

```bash
kubectl describe pvc my-pvc

# Events show:
# Warning  ProvisioningFailed  failed to provision volume with ... UnauthorizedOperation: You are not authorized to perform this operation
```

**Diagnosis Checklist:**

1. **Check if driver is running:**
   ```bash
   kubectl get pods -n kube-system | grep ebs-csi
   # Both controller and node pods should be Running
   ```

2. **Check ServiceAccount annotation:**
   ```bash
   kubectl describe sa ebs-csi-controller-sa -n kube-system
   # Look for: eks.amazonaws.com/role-arn: arn:aws:iam::...
   ```

3. **Check IAM role has correct policy:**
   ```bash
   aws iam list-attached-role-policies \
     --role-name AmazonEKS_EBS_CSI_DriverRole
   # Should list: AmazonEBSCSIDriverPolicy
   ```

4. **Check pod token can be used:**
   ```bash
   # Exec into CSI controller pod and test AWS call
   kubectl exec -it ebs-csi-controller-0 -n kube-system -- bash
   
   # Inside pod, test AWS call:
   aws sts get-caller-identity
   
   # Should return assumed role ARN (not error)
   ```

5. **Check token is mounted:**
   ```bash
   kubectl describe pod ebs-csi-controller-0 -n kube-system | grep token
   # Look for: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
   ```

### Symptom: KMS Permission Denied

```bash
# Events show: failed to create volume ... KMS.NotAuthorizedForSourceException
```

**Fix:**
```bash
# Add KMS permissions (see KMS section above)
```

### Symptom: Too Many Permissions Granted

**Anti-pattern:**
```bash
# ❌ DON'T: Attach full AdministratorAccess
aws iam attach-role-policy \
  --role-name MyCSIRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

**Correct approach:**
```bash
# ✅ DO: Attach only CSI policy
aws iam attach-role-policy \
  --role-name MyCSIRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
```

---

## Architecture Decision: Where to Deploy CSI Controllers

**Option 1: Control Plane Nodes**
- CSI controllers run on control plane nodes
- Pros: Easier management, single location
- Cons: Uses control plane resources

**Option 2: Dedicated Storage Nodes**
- Deploy a few nodes specifically for storage operations
- CSI controllers run on these nodes
- Pros: Isolate storage overhead, production practice
- Cons: More infrastructure

**Option 3: Worker Nodes (Default EKS)**
- CSI controllers run on regular worker nodes
- Pros: Simpler setup, no extra infrastructure
- Cons: Storage overhead affects application performance

For production clusters, **Option 2** (dedicated storage nodes) is recommended to isolate storage operations from application workloads.

---

## Summary: Security and Operations

**Security Principles:**
- Use IRSA; never use static credentials
- Least privilege: attach only needed policies
- KMS encryption for sensitive data
- Monitor CloudTrail for audit trail

**Installation Checklist:**
1. ✅ OIDC provider exists
2. ✅ IAM role created with trust relationship
3. ✅ ServiceAccount annotated with role ARN
4. ✅ CSI driver installed (via add-on or Helm)
5. ✅ StorageClass created
6. ✅ Test: PVC provisions successfully
7. ✅ Verify token in pod

**Module 12 Chapter 38 Complete!** We've covered the full CSI driver ecosystem: architecture, EBS constraints, EFS multi-node sharing, and secure IAM setup. Next, Chapter 39 covers Volume Snapshots for disaster recovery.
