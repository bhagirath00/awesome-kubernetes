# Bitnami Sealed Secrets for GitOps Workflows

## 1. Architectural Concept & Deep Dive

The fundamental principle of GitOps states that the Git repository must serve as the single source of truth for all Kubernetes infrastructure and application state.

However, this introduces the **GitOps Secret Paradox**:
> *How can you store application manifests in public or shared Git repositories when those manifests require database passwords, API tokens, and TLS private keys?*

Storing standard Kubernetes Secrets in Git is catastrophic because Base64 is trivially decoded. Encrypting the entire Git repository inhibits automated CI/CD tools like ArgoCD and Flux from reading manifests.

**Bitnami Sealed Secrets** solves this dilemma through **Asymmetric Public-Key Cryptography**:
1. An in-cluster controller generates an RSA key pair.
2. Developers use the **Public Key** and the `kubeseal` CLI tool to encrypt Secret manifests into **`SealedSecret`** Custom Resources.
3. The resulting `SealedSecret` is **safe to commit publicly to GitHub**. Nobody—not even the developer who created it—can decrypt it without the cluster's private key.
4. When ArgoCD syncs the `SealedSecret` into the cluster, the **Sealed Secrets Controller** uses its **Private Key** to decrypt the ciphertext and generate a standard, native Kubernetes `Secret` in-memory.

```
+─────────────────────────────────────────────────────────────────────────+
|                       SEALED SECRETS WORKFLOW                           |
|                                                                         |
|  [ DEVELOPER WORKSTATION ]                                              |
|  Plaintext Secret (secret.yaml)                                         |
|         │                                                               |
|         ├── kubeseal --format=yaml < secret.yaml                        |
|         │   (Encrypts using Cluster Public Key via RSA-OAEP + AES-GCM)  |
|         ▼                                                               |
|  SealedSecret CRD (sealedsecret.yaml)                                   |
|         │                                                               |
|         ├── git commit & git push (SAFE TO COMMIT TO PUBLIC REPO!)      |
|         ▼                                                               |
|  [ GIT REPOSITORY (GitHub / GitLab) ]                                   |
|         │                                                               |
|         ├── ArgoCD / FluxCD pulls manifest                              |
|         ▼                                                               |
|  [ KUBERNETES CLUSTER ]                                                 |
|  SealedSecret deployed to etcd                                          |
|         │                                                               |
|         ▼ (Controller watches SealedSecret)                             |
|  [ Sealed Secrets Controller ]                                          |
|         │ (Decrypts ciphertext using Private Key in kube-system)        |
|         ▼                                                               |
|  Standard Native Kubernetes Secret (Unsealed & Injected to Pod)         |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Installation: Controller & CLI

### Step 1: Install Controller in the Cluster via Helm
```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

helm install sealed-secrets-controller sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set-string keyrenewperiod="720h"  # Auto-rotate key every 30 days
```

### Step 2: Install the `kubeseal` CLI Tool (Linux)
```bash
KUBESEAL_VERSION="0.24.5"
curl -LO "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xvzf "kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz" kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

---

## 3. Hands-on Lab: Sealing a Production Secret

### Step 1: Create a Standard Kubernetes Secret Locally (Dry-Run)
Never commit this file to Git!

```bash
kubectl create secret generic database-credentials \
  --from-literal=username=dbadmin \
  --from-literal=password=SuperSecureProductionPassword99 \
  --namespace=production \
  --dry-run=client -o yaml > /tmp/secret.yaml
```

### Step 2: Encrypt Using `kubeseal`
The CLI fetches the public certificate from the running controller and seals the secret:

```bash
kubeseal --format=yaml < /tmp/secret.yaml > sealedsecret.yaml
```

Inspect the generated `sealedsecret.yaml`:
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  encryptedData:
    password: AgBy4F2...[long encrypted string]...Q==
    username: AgAo1E9...[long encrypted string]...8=
  template:
    metadata:
      name: database-credentials
      namespace: production
```
- **This file is now safe to commit directly to Git!** Delete the temporary `/tmp/secret.yaml`.

### Step 3: Apply SealedSecret and Verify Decryption
```bash
kubectl apply -f sealedsecret.yaml
```

Verify that the controller unsealed it into a real Secret:
```bash
kubectl get secret database-credentials -n production
```
Output:
```
NAME                   TYPE     DATA   AGE
database-credentials   Opaque   2      5s
```

Decode the unsealed secret to confirm plaintext integrity:
```bash
kubectl get secret database-credentials -n production -o jsonpath="{.data.password}" | base64 -d
# Output: SuperSecureProductionPassword99
```

---

## 4. Encryption Scopes: Strict vs Namespace vs Cluster

Sealed Secrets enforces cryptographic scoping to prevent attackers from copying encrypted secrets into different namespaces or re-naming them:

| Scope | Flag | Decryption Constraint | Best Use Case |
| :--- | :--- | :--- | :--- |
| **`strict`** (Default) | `--scope strict` | Must match **exact name** AND **exact namespace**. | Production microservice credentials. Prevents copying secrets across teams. |
| **`namespace-wide`** | `--scope namespace-wide` | Can be renamed, but must remain in the **same namespace**. | Rotating or versioned secret names within a single project. |
| **`cluster-wide`** | `--scope cluster-wide` | Can be decrypted into **any namespace** under any name. | Universal pull secrets (`imagePullSecrets`), global wildcards. |

```bash
# Example: Create a cluster-wide pull secret
kubeseal --scope cluster-wide --format=yaml < pull-secret.yaml > sealed-pull-secret.yaml
```

---

## 5. Disaster Recovery: Backing Up the Controller Private Key

If your cluster is destroyed and re-built, all existing `SealedSecrets` in Git become unrecoverable **unless you backed up the controller's private key**:

```bash
# Backup the private encryption key secret from kube-system
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > master-sealed-secrets-key.yaml
```
Store `master-sealed-secrets-key.yaml` in an encrypted enterprise vault (1Password, AWS Secrets Manager). On new cluster provisioning, apply this key *before* deploying workloads to restore decryption capabilities.

---

## 6. Key Takeaways
- Sealed Secrets solves the GitOps secret paradox using asymmetric public-key cryptography.
- Only the cluster controller possesses the private key; `SealedSecret` CRDs are safe to store in public Git repositories.
- `strict` scope guarantees encrypted strings cannot be hijacked into unauthorized namespaces.
- Always back up the controller's private key secret to prevent permanent data loss during disaster recovery.
