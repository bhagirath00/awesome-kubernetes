# Secrets Store CSI Driver: Decoupling Secrets from etcd

## 1. Architectural Concept & Deep Dive

The ultimate pattern for Kubernetes secrets management is not encrypting secrets in etcd, but **eliminating secrets from etcd entirely**.

The **Secrets Store CSI Driver** (a sub-project of Kubernetes SIG Storage) allows the kubelet to mount secrets stored in external enterprise secret management systems—such as **HashiCorp Vault, AWS Secrets Manager, Google Secret Manager, and Azure Key Vault**—directly into Pod volumes as in-memory files.

```
+─────────────────────────────────────────────────────────────────────────+
|                  SECRETS STORE CSI DRIVER ARCHITECTURE                  |
|                                                                         |
|   Pod Manifest defines Volume:                                          |
|   • csi.driver: secrets-store.csi.k8s.io                                |
|   • secretProviderClass: aws-database-secrets                           |
|                 │                                                       |
|                 ▼                                                       |
|      [ Worker Node Kubelet ]                                            |
|                 │                                                       |
|                 ├── Invokes CSI Mount Hook                              |
|                 ▼                                                       |
|      [ Secrets Store CSI Driver (DaemonSet) ]                           |
|                 │                                                       |
|                 ├── Calls Provider Plugin (AWS / Vault / Azure / GCP)   |
|                 │   Uses Pod's Workload Identity (IRSA / Token)         |
|                 ▼                                                       |
|      [ External Secret Store (AWS Secrets Manager / Vault) ]            |
|                 │                                                       |
|                 ▼ (Returns Secret Payload)                              |
|      [ CSI Driver writes files to In-Memory tmpfs volume ]              |
|                 │                                                       |
|                 ▼                                                       |
|   Pod Container mounts volume at /mnt/secrets/db_password               |
|                                                                         |
|   * Zero secrets written to etcd! Nothing stored in cluster state!      |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Core Operational Advantages

| Feature | Native K8s Secret | Secrets Store CSI Driver |
| :--- | :--- | :--- |
| **Storage Location** | etcd (Vulnerable to snapshot leaks) | **External Vault/AWS KMS** (Never stored in etcd) |
| **Audit Trails** | Coarse Kubernetes API audit | **Granular HSM Audit** (e.g. AWS CloudTrail logs exact IAM role access) |
| **Rotation** | Manual rollout / restart required | **Automated Dynamic Rotation** without pod restarts |
| **Enterprise Sync** | Duplicate secrets across environments | **Single Source of Truth** across VMs, serverless, and clusters |

---

## 3. Production Installation via Helm

Deploy both the generic CSI driver and the specific cloud provider plugin (e.g., AWS provider):

```bash
# 1. Add Secrets Store CSI Driver Helm repository
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm repo update

# 2. Install CSI Driver with Rotation and Secret Sync enabled
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set enableSecretRotation=true \
  --set rotationPollInterval=120s \
  --set syncSecret.enabled=true

# 3. Install Provider Plugin (Example: AWS Provider)
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

---

## 4. Hands-on Lab: Mounting AWS Secrets into a Pod

### Step 1: Define the `SecretProviderClass` CRD
The `SecretProviderClass` bridges Kubernetes namespaces with the external secrets engine:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-app-secrets
  namespace: production
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "production/database/credentials"
        objectType: "secretsmanager"
        jmesPath: 
          - path: username
            objectAlias: db_user
          - path: password
            objectAlias: db_password
  
  # Optional: Also sync to a native K8s Secret (for env var injection)
  secretObjects:
    - secretName: synced-db-secret
      type: Opaque
      data:
        - objectName: db_password
          key: DB_PASSWORD
        - objectName: db_user
          key: DB_USER
```

### Step 2: Deploy Pod with CSI Volume Mount
Attach the volume to your deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: billing-service
  namespace: production
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: billing
    spec:
      serviceAccountName: billing-service-account # Uses AWS IRSA!
      containers:
        - name: app
          image: myorg/billing:v1.0
          volumeMounts:
            - name: secrets-store-inline
              mountPath: "/mnt/secrets"
              readOnly: true
          # Read from synced native secret
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: synced-db-secret
                  key: DB_PASSWORD
      volumes:
        - name: secrets-store-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "aws-app-secrets"
```

---

## 5. Automated Dynamic Rotation (`enableSecretRotation`)

When an external secret is rotated (e.g., database password rotated every 90 days in AWS Secrets Manager or Vault):
1. The CSI Driver polls the external provider every `rotationPollInterval` (e.g. 120 seconds).
2. When a new version is detected, the driver updates the mounted in-memory files at `/mnt/secrets/db_password`.
3. If `secretObjects` is configured, it patches the synced Kubernetes `Secret` automatically.
4. Applications watching the filesystem reload the new credential with **zero pod restarts**.

---

## 6. Verification & Troubleshooting Runbook

### Health & Diagnostic Commands
```bash
# Verify CSI driver daemonset is healthy across all nodes
kubectl get daemonset -n kube-system -l app.kubernetes.io/name=secrets-store-csi-driver

# Inspect SecretProviderClass status and mounted events
kubectl describe secretproviderclass aws-app-secrets -n production

# Verify files are mounted inside the running container
kubectl exec -it deployment/billing-service -n production -- ls -la /mnt/secrets/
```

### Common Pitfalls & Solutions
1. **Pod hangs in `ContainerCreating`**:
   - Cause: CSI driver fails to authenticate to external provider, or secret name does not exist.
   - Solution: Run `kubectl describe pod <NAME>`. Look at the `Events` section: check for `failed to get secret` or AWS IAM permission denied errors.
2. **Secrets not mounted if no volume mount defined**:
   - Cause: The Secrets Store CSI Driver is a volume driver. Secrets are **only fetched when a pod mounting the volume starts up**. Syncing to `secretObjects` will NOT happen without a pod mounting the volume.

---

## 7. Key Takeaways
- The Secrets Store CSI Driver decouples secrets from etcd, keeping external secrets engines as the single source of truth.
- Mounts external credentials directly into containers as in-memory files via a CSI volume.
- Supports automated background polling and rotation without requiring pod redeployments.
- Can optionally synchronize external credentials into native Kubernetes Secrets for legacy applications requiring environment variables.
