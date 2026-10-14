# HashiCorp Vault in Kubernetes: Architecture & Auth Method

## 1. Architectural Concept & Deep Dive

**HashiCorp Vault** is the industry standard for identity-based secret management, encryption as a service, and privileged access management. 

In Kubernetes, integrating Vault eliminates static credentials by turning the Kubernetes **ServiceAccount token** into a verifiable identity. Pods never store master passwords, root API keys, or long-lived database credentials. Instead:
1. The Pod presents its cryptographic JWT token to Vault.
2. Vault validates the token with the Kubernetes API server via the **`TokenReview` API**.
3. Vault maps the Pod's ServiceAccount to an internal **Vault Policy**.
4. Vault issues short-lived, dynamically generated, automatically revoked credentials.

```
+─────────────────────────────────────────────────────────────────────────+
|                 VAULT KUBERNETES AUTHENTICATION FLOW                    |
|                                                                         |
|  [ Microservice Pod ]                                                   |
|  ServiceAccount: "payment-app"                                          |
|  Mounts projected SA token: /var/run/secrets/tokens/vault-token         |
|         │                                                               |
|         ├── 1. POST /v1/auth/kubernetes/login (Presents JWT Token)      |
|         ▼                                                               |
|  [ HashiCorp Vault Cluster ]                                            |
|         │                                                               |
|         ├── 2. POST /apis/authentication.k8s.io/v1/tokenreviews         |
|         │   (Vault asks API Server: "Is this token valid for payment?") |
|         ▼                                                               |
|  [ Kubernetes API Server ]                                              |
|         │                                                               |
|         ├── 3. Returns TokenReview: Valid! UID=123, Namespace=prod      |
|         ▼                                                               |
|  [ HashiCorp Vault Cluster ]                                            |
|         │                                                               |
|         ├── 4. Maps Pod to Role: "payment-role"                         |
|         ├── 5. Attaches Policy: "read /secret/data/production/payment"   |
|         ▼                                                               |
|  Returns Short-Lived Vault Client Token (TTL: 1 Hour) ──► Pod           |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Vault Deployment Models on Kubernetes

Depending on enterprise architecture, Vault operates in one of two topologies:

| Topology | Architecture | Pros & Cons |
| :--- | :--- | :--- |
| **External Vault Cluster** (Recommended for Enterprise) | Vault runs on dedicated VMs or HashiCorp Cloud Platform (HCP) outside the cluster. | High availability decoupled from Kubernetes lifecycle; central vault serves multi-cloud and bare metal. |
| **In-Cluster Vault** | Vault deployed as a Kubernetes `StatefulSet` with integrated Raft storage (`storage "raft"`). | Self-contained, zero cloud dependencies, low latency; however, cluster disaster recovery impacts Vault. |

---

## 3. Configuring the Kubernetes Auth Method in Vault

To allow Kubernetes pods to authenticate with Vault, configure the `auth/kubernetes` engine.

### Step 1: Enable the Kubernetes Auth Engine
```bash
vault auth enable kubernetes
```

### Step 2: Configure Vault with Kubernetes API Coordinates
Vault needs to reach the API server to perform TokenReview verifications:

```bash
# Retrieve internal cluster CA and API server address
K8S_HOST="https://kubernetes.default.svc.cluster.local:443"

vault write auth/kubernetes/config \
    kubernetes_host="$K8S_HOST"
```
*(In Kubernetes 1.21+, Vault uses its own local service account token to communicate with the TokenReview API).*

---

## 4. Vault Policies & Role Mapping

Vault enforces authorization using declarative **HCL (HashiCorp Configuration Language) Policies**.

### Step 1: Define the Policy (`payment-policy.hcl`)
```hcl
# Grant read-only access to KV v2 secrets for the payment application
path "secret/data/production/payment/*" {
  capabilities = ["read"]
}

# Grant permission to request dynamic PostgreSQL credentials
path "database/creds/payment-role" {
  capabilities = ["read"]
}
```

Write policy to Vault:
```bash
vault policy write payment-policy payment-policy.hcl
```

### Step 2: Create a Bound Vault Role
A **Role** binds the policy to specific Kubernetes identities:

```bash
vault write auth/kubernetes/role/payment-app-role \
    bound_service_account_names=payment-sa \
    bound_service_account_namespaces=production \
    policies=payment-policy \
    ttl=1h
```
- **Security Invariant**: Even if an attacker steals the role name, Vault will strictly reject authentication unless the request originates from a pod running as `payment-sa` inside the `production` namespace!

---

## 5. Core Secret Engines Used in Kubernetes

| Secret Engine | Path | Purpose |
| :--- | :--- | :--- |
| **KV (Key-Value v2)** | `secret/` | Static secrets with versioning, rollback, and soft-delete capabilities. |
| **Database** | `database/` | **Dynamic on-demand credentials**. Generates unique DB users per pod with automated TTL expiration. |
| **PKI** | `pki/` | Acts as an internal Certificate Authority (CA) generating short-lived TLS/mTLS certificates. |
| **Transit** | `transit/` | Cryptography-as-a-Service (encrypts/decrypts data without storing it). |

---

## 6. Key Takeaways
- Vault eliminates hardcoded credentials by using Kubernetes ServiceAccount tokens as cryptographic identities.
- The `auth/kubernetes` method validates tokens using the Kubernetes `TokenReview` API.
- Vault Roles bind Kubernetes ServiceAccounts and Namespaces to fine-grained HCL access policies.
- Supports both static versioned secrets (KV v2) and dynamic, short-lived database credentials.
