# Vault CSI Provider: Node-Level Secret Architecture

## 1. Architectural Concept & Deep Dive

While the **Vault Agent Injector** (Chapter 57.2) is popular for its rich templating capabilities, it requires running two containers (`vault-agent-init` and `vault-agent`) **inside every single application pod**.

In clusters running thousands of pods, injecting two sidecar containers per pod creates massive operational overhead:
- **Resource Bloat**: Thousands of sidecars consume CPU limits and hundreds of gigabytes of node memory.
- **Pod Lifecycle Delays**: Init containers delay pod startup times during autoscaling events.
- **Connection Storms**: Thousands of independent Vault Agents bombard the Vault cluster with token renewal traffic.

The **Vault CSI Provider** addresses this by shifting secret retrieval to a **single, shared DaemonSet per node**. Secrets are fetched via the Kubernetes **Secrets Store CSI Driver** and mounted as native filesystem volumes without injecting any sidecars into application pods.

```
+─────────────────────────────────────────────────────────────────────────+
|               VAULT AGENT INJECTOR vs VAULT CSI PROVIDER                |
|                                                                         |
|  PATTERN A: Vault Agent Injector          PATTERN B: Vault CSI Provider |
|  (Sidecar Per Pod Architecture)           (Node DaemonSet Architecture) |
|                                                                         |
|  [ Pod 1 ]      [ Pod 2 ]                 [ Pod 1 ]       [ Pod 2 ]     |
|  ├── app        ├── app                   └── app         └── app       |
|  ├── v-init     ├── v-init                       │               │      |
|  └── v-agent    └── v-agent                      └── CSI Mount ──┘      |
|        │              │                                  │              |
|        └──────┬───────┘                                  ▼              |
|               ▼                                  [ Node CSI DaemonSet ] |
|  Thousands of sidecars bombard Vault                    │ (Single link) |
|                                                         ▼               |
|                                                [ Vault Cluster ]        |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Architectural Comparison Matrix

| Dimension | Vault Agent Injector | Vault CSI Provider |
| :--- | :--- | :--- |
| **Footprint** | 2 containers injected **per Pod** | 1 DaemonSet pod **per Node** |
| **Cluster Memory Overhead**| **High** (~100MB+ per pod across cluster) | **Negligible** (Shared node footprint) |
| **Startup Latency** | Slower (Init container must complete) | Fast (Handled during standard volume mount phase) |
| **Templating** | Full Consul Template engine | Basic key mapping |
| **Secret Sync to K8s Secret**| Requires custom scripting | **Native support** via `secretObjects` |
| **Best Used For** | Microservices requiring complex config file generation or signal reloads (`kill -HUP`). | High-density clusters, massive autoscaling workloads, zero-sidecar mandates. |

---

## 3. Production Installation via Helm

The Vault CSI Provider runs alongside the Kubernetes Secrets Store CSI Driver.

```bash
# 1. Add HashiCorp Helm Repository
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# 2. Deploy Vault with CSI Provider enabled
helm upgrade --install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "csi.enabled=true" \
  --set "injector.enabled=false" # Disable injector if running pure CSI
```

Verify that the DaemonSet is running on all nodes:
```bash
kubectl get daemonset -n vault -l app.kubernetes.io/name=vault-csi-provider
```

---

## 4. Hands-on Lab: Configuring the `SecretProviderClass`

### Step 1: Create the Vault `SecretProviderClass`
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-credentials
  namespace: production
spec:
  provider: vault
  parameters:
    roleName: "production-payment-role"
    vaultAddress: "https://vault.vault.svc.cluster.local:8200"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/production/database"
        secretKey: "password"
      - objectName: "db-username"
        secretPath: "secret/data/production/database"
        secretKey: "username"
```

### Step 2: Deploy Application Pod
Mount the secrets via the standard `csi` volume driver:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-processor
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-processor
  template:
    metadata:
      labels:
        app: payment-processor
    spec:
      serviceAccountName: payment-sa  # Used for Vault Kubernetes Auth!
      containers:
        - name: app
          image: myorg/payment:v1.0
          volumeMounts:
            - name: vault-secrets
              mountPath: "/etc/secrets"
              readOnly: true
      volumes:
        - name: vault-secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "vault-db-credentials"
```

---

## 5. Verification & Inspection

Apply the manifests:
```bash
kubectl apply -f secretproviderclass.yaml
kubectl apply -f deployment.yaml
```

Verify the files inside the running container:
```bash
kubectl exec -it deployment/payment-processor -n production -- ls -la /etc/secrets/
```
Output:
```
total 8
-rw-r--r-- 1 root root 12 Oct 14 09:40 db-password
-rw-r--r-- 1 root root  8 Oct 14 09:40 db-username
```

Check the plaintext values:
```bash
kubectl exec -it deployment/payment-processor -n production -- cat /etc/secrets/db-password
# Output: S3cur3P@ssw0rd!
```

---

## 6. Key Takeaways
- The Vault CSI Provider moves secret retrieval out of application pods into a shared node-level DaemonSet.
- Eliminates sidecar memory bloat and API connection storms, making it ideal for high-density production clusters.
- Configured declaratively using the `SecretProviderClass` CRD with `provider: vault`.
- Uses native Kubernetes volume mounts to provide secrets directly to containers as in-memory files.
