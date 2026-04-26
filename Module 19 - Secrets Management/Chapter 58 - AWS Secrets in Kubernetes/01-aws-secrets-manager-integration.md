# AWS Secrets Manager Integration in Kubernetes

## 1. Architectural Concept & Deep Dive

For teams deploying Kubernetes on Amazon Web Services (AWS EKS or self-managed clusters on EC2), **AWS Secrets Manager** is the cloud-native single source of truth for database credentials, API keys, and third-party tokens.

There are two primary architectural patterns to integrate AWS Secrets Manager with Kubernetes:

```
+─────────────────────────────────────────────────────────────────────────+
|               AWS SECRETS MANAGER INTEGRATION PATTERNS                  |
|                                                                         |
|  PATTERN 1: Secrets Store CSI Driver        PATTERN 2: External Secrets |
|  (Direct In-Memory Volume Mount)            (Sync to Native K8s Secret) |
|                                                                         |
|  [ Pod Container ]                          [ Pod Container ]           |
|         │                                          │                    |
|         ├── Mounts /mnt/secrets/api_key            ├── Reads env from   |
|         │   (In-memory tmpfs)                      │   k8s Secret       |
|         ▼                                          ▼                    |
|  [ Secrets Store CSI Driver ]               [ Native K8s Secret ]       |
|         │                                          ▲                    |
|         │ (Fetches via IRSA)                       │ (Reconciles)       |
|         ▼                                   [ External Secrets Operator]|
|  +─────────────────────────────────────────────────┼────────────────+   |
|  |             AWS Secrets Manager (AWS Cloud)     │                |   |
|  |             • KMS 256-bit Encryption at rest    │                |   |
|  |             • Automated 30-day RDS Rotation     │                |   |
|  |             • CloudTrail Access Logging         │                |   |
|  +─────────────────────────────────────────────────┴────────────────+   |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Architectural Comparison: CSI Driver vs External Secrets Operator (ESO)

| Dimension | Secrets Store CSI Driver | External Secrets Operator (ESO) |
| :--- | :--- | :--- |
| **How Pod Consumes Secret** | **Filesystem Volume Mount** (`/mnt/secrets/`) | **Standard Kubernetes Secret** (`envFrom` / `secretKeyRef`) |
| **Storage in etcd** | **Never stored in etcd** (Pure in-memory tmpfs) | Stored in etcd (Requires K8s Encryption at Rest) |
| **Legacy App Compatibility**| Requires app to read files (or enable sync) | **100% Native** (Drop-in replacement for existing pods) |
| **Secret Lifecycle** | Tied to pod volume lifecycle | Decoupled (Background controller syncs continuously) |
| **AWS Authentication** | AWS IRSA / EKS Pod Identity | AWS IRSA / EKS Pod Identity |

---

## 3. Pattern A: Secrets Store CSI Driver with AWS Provider

### Step 1: Install CSI Driver & AWS Provider
```bash
# 1. Install CSI Driver
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver -n kube-system

# 2. Deploy AWS CSI Provider DaemonSet
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

### Step 2: Define `SecretProviderClass` with JMESPath Parsing
AWS Secrets Manager typically stores credentials as JSON objects. Use `jmesPath` to extract specific keys:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-rds-secret-provider
  namespace: production
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/aurora/postgres"
        objectType: "secretsmanager"
        jmesPath:
          - path: username
            objectAlias: DB_USER
          - path: password
            objectAlias: DB_PASSWORD
          - path: host
            objectAlias: DB_HOST
```

---

## 4. Pattern B: External Secrets Operator (ESO)

The **External Secrets Operator (ESO)** is the community favorite when microservices require traditional environment variables (`envFrom`) rather than reading filesystem files.

### Step 1: Install External Secrets Operator via Helm
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

### Step 2: Create a `SecretStore` Backed by AWS
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager-store
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: eso-irsa-serviceaccount
```

### Step 3: Define the `ExternalSecret` CRD
ESO reads the secret from AWS and creates a native Kubernetes `Secret` inside the namespace:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-database-secret
  namespace: production
spec:
  refreshInterval: "1h"  # Sync changes from AWS every hour
  secretStoreRef:
    name: aws-secrets-manager-store
    kind: SecretStore
  target:
    name: local-db-secret  # Name of the native Kubernetes Secret created in etcd
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: "prod/aurora/postgres"
        property: password
    - secretKey: DB_USER
      remoteRef:
        key: "prod/aurora/postgres"
        property: username
```

---

## 5. Security & Audit Logging (AWS CloudTrail)

Every time the CSI driver or ESO queries AWS Secrets Manager:
- AWS records a **`GetSecretValue`** event in **AWS CloudTrail**.
- The event records the exact caller ARN, IAM role, source IP, and timestamp.
- If a pod is compromised, CloudTrail provides an immutable audit log detailing exactly which secrets were accessed.

---

## 6. Key Takeaways
- Integrating AWS Secrets Manager unifies credential management across cloud infrastructure and Kubernetes workloads.
- The Secrets Store CSI Driver mounts secrets as in-memory files without storing anything in etcd.
- The External Secrets Operator (ESO) syncs AWS secrets into native Kubernetes Secrets, ideal for applications requiring environment variables.
- Both patterns leverage AWS IRSA (IAM Roles for Service Accounts) for zero-token, secure cloud authentication.
