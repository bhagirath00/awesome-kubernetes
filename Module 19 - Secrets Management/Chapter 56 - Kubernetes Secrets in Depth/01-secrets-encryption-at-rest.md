# Kubernetes Secrets Encryption at Rest

## 1. Architectural Concept & Deep Dive

A fundamental reality of native Kubernetes Secrets:
> **Kubernetes Secrets are NOT encrypted by default. They are merely Base64-encoded strings.**

Base64 is an encoding format, not an encryption mechanism. Anyone with read access to the etcd database, access to backup snapshots, or permission to execute `kubectl get secret -o yaml` can decode credentials instantly (`echo "cGFzc3dvcmQ=" | base64 -d`).

To protect credentials against physical disk theft, unprivileged etcd access, and cloud snapshot exfiltration, the Kubernetes control plane provides **Encryption at Rest** via the `EncryptionConfiguration` API.

```
+─────────────────────────────────────────────────────────────────────────+
|                  SECRETS ENCRYPTION AT REST ARCHITECTURE                |
|                                                                         |
|  kubectl apply -f secret.yaml (Plaintext in transit over TLS)           |
|                 │                                                       |
|                 ▼                                                       |
|      [ Kubernetes API Server ]                                          |
|                 │                                                       |
|                 ▼ (Calls Encryption Provider)                           |
|      +───────────────────────────────────────────────────────+          |
|      |               EncryptionConfiguration                 |          |
|      |                                                       |          |
|      |   Option A: Local Key (aescbc / aesgcm)               |          |
|      |   Uses 32-byte key stored on master node disk         |          |
|      |                                                       |          |
|      |   Option B: KMS Envelope Encryption (Recommended)     |          |
|      |   1. Generate unique Data Encryption Key (DEK)        |          |
|      |   2. Encrypt Secret payload with DEK                  |          |
|      |   3. Call Cloud KMS (AWS KMS/Vault) to encrypt DEK    |          |
|      |   4. Store [Encrypted DEK + Ciphertext] in etcd       |          |
|      +──────────────────────────┬────────────────────────────+          |
|                                 │                                       |
|                                 ▼                                       |
|      [ etcd Database: /registry/secrets/<ns>/<name> ]                   |
|      Value: k8s:enc:kms:v2:my-kms-plugin: ... (Ciphertext)              |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Encryption Providers Comparison

Kubernetes supports multiple cryptographic providers in its evaluation chain:

| Provider | Cryptographic Algorithm | Key Storage | Production Suitability |
| :--- | :--- | :--- | :--- |
| **`identity`** | None (Plaintext / Base64) | N/A | Default in vanilla clusters. **Zero security**. |
| **`aescbc`** | AES-CBC with PKCS#7 padding | Stored in cleartext inside master node config | Acceptable for air-gapped clusters; vulnerable if master node is compromised. |
| **`aesgcm`** | AES-GCM (Authenticated Encryption) | Stored in cleartext inside master node config | Stronger than CBC, but requires key rotation every 200,000 writes to prevent nonce reuse. |
| **`kms` (v2)** | Envelope Encryption (AES-GCM + External KEK) | External HSM / KMS (AWS KMS, GCP KMS, Vault) | **Industry Gold Standard**. Keys never touch cluster nodes in plaintext; satisfies compliance. |

---

## 3. Envelope Encryption via KMS v2 Plugin

In the **KMS v2 Envelope Encryption** model:
1. **Key Encryption Key (KEK)**: Stored and managed securely inside an external Hardware Security Module (HSM) such as AWS KMS.
2. **Data Encryption Key (DEK)**: A dynamic, ephemeral AES-GCM key generated locally by the API server.
3. The API server encrypts the Secret with the DEK, asks AWS KMS to encrypt the DEK with the KEK, and stores the encrypted DEK alongside the encrypted secret payload in etcd.

### KMS v2 Configuration Manifest (`encryption-config-kms.yaml`)
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - kms:
          apiVersion: v2
          name: aws-kms-provider
          endpoint: unix:///var/run/kmsplugin/socket.sock
          timeout: 3s
      # Fallback provider to read unencrypted secrets during rollout
      - identity: {}
```

---

## 4. Step-by-Step Implementation Lab

### Step 1: Create Static Encryption Key Configuration (Local Provider)
If an external KMS is unavailable, deploy a hardened `aescbc` provider:

```bash
# Generate 32 bytes of secure random data and base64-encode it
SECRET_KEY=$(head -c 32 /dev/urandom | base64)

# Create configuration directory
sudo mkdir -p /etc/kubernetes/enc

# Generate configuration file
cat <<EOF | sudo tee /etc/kubernetes/enc/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${SECRET_KEY}
      - identity: {}
EOF

sudo chmod 600 /etc/kubernetes/enc/encryption-config.yaml
```

### Step 2: Enable Provider in `kube-apiserver.yaml`
Add the `--encryption-provider-config` argument and host volume mounts to `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
    volumeMounts:
    - mountPath: /etc/kubernetes/enc/encryption-config.yaml
      name: enc-config
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/enc/encryption-config.yaml
      type: File
    name: enc-config
```

### Step 3: Re-encrypt Pre-existing Secrets
Newly created secrets will be encrypted automatically. To encrypt all historical secrets already in etcd:

```bash
# Re-save all secrets across all namespaces to force encryption pass
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

---

## 5. Verification & Key Rotation Strategy

### 1. Verify Encryption in etcd Directly
```bash
# Query the raw etcd database value
sudo ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret
```
- The output should start with the header **`k8s:enc:aescbc:v1:key1:`**.
- Plaintext strings must not appear anywhere in the output!

### 2. Zero-Downtime Key Rotation
To rotate encryption keys without outage:
1. Generate a new key (`key2`) and prepend it as the **first entry** in the `keys` list.
2. Keep the old key (`key1`) as the **second entry** so existing secrets can still be decrypted.
3. Restart the API server.
4. Run `kubectl get secrets --all-namespaces -o json | kubectl replace -f -` to re-encrypt all secrets with `key2`.
5. Remove `key1` from the configuration file and restart the API server.

---

## 6. Key Takeaways
- Kubernetes Secrets are not encrypted by default; they are merely Base64-encoded.
- `EncryptionConfiguration` enables transparent, cryptographic encryption at rest before data reaches etcd.
- KMS v2 envelope encryption is the industry best practice, eliminating static keys on master disks.
- After enabling encryption, you must trigger an in-place replace (`kubectl replace`) to encrypt pre-existing secrets.
