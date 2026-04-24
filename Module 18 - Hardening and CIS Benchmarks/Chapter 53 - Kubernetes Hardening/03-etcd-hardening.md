# Hardening etcd: Storage Security & Encryption at Rest

## 1. Architectural Concept & Deep Dive

**etcd** is the distributed, consistent key-value store that holds the entire state of a Kubernetes cluster—every secret, token, configmap, pod definition, and RBAC binding.

If an attacker gains direct read access to etcd, **all cluster security boundaries vanish**:
- They can dump every Kubernetes Secret in plaintext.
- They can create a `ClusterRoleBinding` granting themselves `cluster-admin`.
- They bypass the Kubernetes API server, audit logs, and admission webhooks entirely!

Hardening etcd requires two distinct defense mechanisms:
1. **Transport & Network Hardening (mTLS)**: Enforcing mutual TLS authentication for both client connections (API server -> etcd) and peer-to-peer raft synchronization (etcd node <-> etcd node).
2. **Encryption at Rest**: Ensuring that even if an attacker gains physical or snapshot access to `/var/lib/etcd`, all secret payloads stored in the bbolt database files are encrypted using cryptographic keys.

```
+─────────────────────────────────────────────────────────────────────────+
|                           ETCD HARDENING FLOW                           |
|                                                                         |
|   Kubernetes API Server                                                 |
|          │                                                              |
|          ├── 1. Client mTLS Handshake (Port 2379)                       |
|          │   • Requires client cert signed by etcd-ca                   |
|          ▼                                                              |
|   [ etcd Process (Linux UID: etcd) ]                                    |
|          │                                                              |
|          ├── 2. Peer mTLS Handshake (Port 2380)                         |
|          │   • Peer cluster synchronization                             |
|          ▼                                                              |
|   [ In-Memory Storage Engine ]                                          |
|          │                                                              |
|          ├── 3. Encryption at Rest (EncryptionConfiguration)            |
|          │   • AES-GCM / KMS Provider encrypts Secret payloads          |
|          ▼                                                              |
|   [ On-Disk Database: /var/lib/etcd/member/snap/db ]                    |
|   • File Permissions: 0700 / 0600 (Strictly owned by etcd:etcd)         |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Mutual TLS (mTLS) Hardening Flags

In `/etc/kubernetes/manifests/etcd.yaml`, verify the following mandatory flags are configured:

```yaml
spec:
  containers:
  - command:
    - etcd
    # Client-to-Server Authentication (Port 2379)
    - --client-cert-auth=true
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --listen-client-urls=https://127.0.0.1:2379,https://<NODE_IP>:2379

    # Peer-to-Peer Authentication (Port 2380)
    - --peer-client-cert-auth=true
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --listen-peer-urls=https://<NODE_IP>:2380
```
- Setting `--client-cert-auth=true` guarantees that unauthenticated requests or requests with invalid certificates receive an immediate TLS handshake rejection.

---

## 3. Filesystem & Directory Permissions Hardening

In accordance with CIS Kubernetes Benchmark Section 1.1:
- The etcd data directory contains the raw database file (`member/snap/db`).
- Permissions must be restricted to root or the dedicated etcd system user:

```bash
# Set directory permissions to 0700
sudo chmod 700 /var/lib/etcd

# Set file ownership
sudo chown -R root:root /var/lib/etcd  # Or etcd:etcd on external clusters

# Verify permissions
ls -ld /var/lib/etcd
# Output: drwx------ 3 root root 4096 Oct 14 09:00 /var/lib/etcd
```

Also audit the PKI certificates directory:
```bash
sudo chmod 700 /etc/kubernetes/pki/etcd
sudo chmod 600 /etc/kubernetes/pki/etcd/*.key
```

---

## 4. Encryption at Rest: Protecting Secrets

By default, Kubernetes writes Secrets into etcd as base64-encoded strings (which is NOT encryption). Anyone reading the database can decode them instantly.

### Step 1: Create an EncryptionConfiguration Manifest
Create `/etc/kubernetes/enc/encryption-config.yaml`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      # Provider 1: AES-GCM (Authenticated 256-bit AES encryption)
      - aescbc:
          keys:
            - name: key1
              # 32-byte random base64 key generated via: head -c 32 /dev/urandom | base64
              secret: 2s3Xy8vB6R5F4Q1w7P9mK0zN2aC4eG6h8J0lO2qS4uU=
      # Provider 2: Identity fallback (Required to read legacy unencrypted data during migration)
      - identity: {}
```

> **Enterprise Tip (KMS Integration)**: For high-security environments, replace the static `aescbc` provider with a **`kms` provider plugin** connected to AWS KMS, Google Cloud KMS, or HashiCorp Vault. This ensures encryption keys are rotated externally and never reside in plaintext on master node disks.

### Step 2: Configure API Server to Use Encryption Configuration
In `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
    volumeMounts:
    - mountPath: /etc/kubernetes/enc
      name: enc-config
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/enc
      type: Directory
    name: enc-config
```

### Step 3: Re-encrypt Existing Secrets
Applying the configuration encrypts *new* secrets. To encrypt all pre-existing secrets in etcd:
```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

---

## 5. Verification & Validation Lab

### Verify Encryption at Rest Directly from etcd
Create a test secret:
```bash
kubectl create secret generic prod-db-pass --from-literal=password=SuperSecretPassword123
```

Now query the raw database key using `etcdctl`:
```bash
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/prod-db-pass
```

Output:
```
/registry/secrets/default/prod-db-pass
k8s:enc:aescbc:v1:key1:...
```
- **Proof of Security**: The stored value begins with `k8s:enc:aescbc:v1:key1:` followed by encrypted ciphertext! The plaintext string `SuperSecretPassword123` is completely absent from etcd.

---

## 6. Key Takeaways
- etcd contains the keys to the cluster; unauthenticated access allows complete takeover.
- Always enforce `--client-cert-auth=true` and `--peer-client-cert-auth=true`.
- Enforce strict `0700` filesystem permissions on `/var/lib/etcd` and `0600` on private keys.
- Enable `EncryptionConfiguration` to encrypt all Secrets stored in etcd at rest, preferably backed by Cloud KMS.
