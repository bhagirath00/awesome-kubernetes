# Hardening the Kubernetes API Server (kube-apiserver)

## 1. Architectural Concept & Deep Dive

The **Kubernetes API Server (`kube-apiserver`)** is the central gateway to the entire control plane. Every interaction—whether from human operators via `kubectl`, worker node daemons (`kubelet`), controllers (`kube-controller-manager`), or in-cluster microservices—passes through the API server pipeline.

If an attacker gains unauthorized administrative access to the API server, they gain total control over all compute, storage, secrets, and networks across the entire cluster.

```
+─────────────────────────────────────────────────────────────────────────+
|                       KUBE-APISERVER PIPELINE                           |
|                                                                         |
|   Inbound Request (Port 6443 over TLS)                                  |
|          │                                                              |
|          ▼                                                              |
|   1. AUTHENTICATION (Who are you?)                                      |
|      • Verify mTLS Client Certificates                                  |
|      • Validate OIDC Tokens (Okta / Azure AD / Keycloak)                |
|      • Validate Webhook Tokens                                          |
|      • Anonymous Auth: MUST BE DISABLED (--anonymous-auth=false)        |
|          │                                                              |
|          ▼                                                              |
|   2. AUTHORIZATION (What can you do?)                                   |
|      • Node Authorization (restricts kubelet to own pods)               |
|      • RBAC (Role-Based Access Control: Roles, ClusterRoles)            |
|          │                                                              |
|          ▼                                                              |
|   3. ADMISSION CONTROL (Can it be admitted?)                           |
|      • Mutating Webhooks                                                |
|      • Built-in: NodeRestriction, PodSecurity                           |
|      • Validating Webhooks (Kyverno / OPA Gatekeeper)                   |
|          │                                                              |
|          ▼                                                              |
|   4. PERSISTENCE                                                        |
|      • Encrypted at rest committed to etcd                              |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Critical Hardening Flags in `kube-apiserver.yaml`

In kubeadm and self-hosted control planes, API server flags reside in the static pod manifest at `/etc/kubernetes/manifests/kube-apiserver.yaml`.

### 1. Authentication Hardening
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    # Disallow unauthenticated requests
    - --anonymous-auth=false
    
    # Reject insecure basic auth or static token files
    # Ensure --basic-auth-file and --token-auth-file are NEVER present!
    
    # Enable enterprise identity integration via OIDC
    - --oidc-issuer-url=https://login.mycompany.com/oauth2/v1
    - --oidc-client-id=kubernetes-cluster-prod
    - --oidc-username-claim=email
    - --oidc-groups-claim=groups
```

### 2. Authorization Hardening
```yaml
    # Strictly enforce Node and RBAC modes; NEVER include AlwaysAllow!
    - --authorization-mode=Node,RBAC
```
- **`Node` Authorizer**: Ensures worker node kubelets can only read and modify resources (secrets, configmaps, pods) scheduled directly on their own physical node.

### 3. Admission Plugins Hardening
```yaml
    # Enable critical security admission controllers
    - --enable-admission-plugins=NodeRestriction,PodSecurity,LimitRanger,ResourceQuota
```
- **`NodeRestriction`**: Prevents a compromised worker node kubelet from modifying another node's labels or taints.

### 4. Cryptographic & Transport Hardening
```yaml
    # Enforce modern TLS 1.3 or TLS 1.2 with strong forward-secret ciphers
    - --tls-min-version=VersionTLS13
    - --tls-cipher-suites=TLS_AES_128_GCM_SHA256,TLS_AES_256_GCM_SHA384,TLS_CHACHA20_POLY1305_SHA256
    
    # Mutual TLS (mTLS) client verification
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
```

---

## 3. Configuring Production API Audit Logging

Without API audit logging, incident response is blind. Kubernetes provides a declarative audit policy engine capable of logging metadata, request bodies, and responses.

### Step 1: Create an Enterprise Audit Policy (`/etc/kubernetes/audit-policy.yaml`)
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  # 1. Log secret changes at Metadata level only (never log secret plaintext!)
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]

  # 2. Log high-risk authorization and workload modifications at RequestResponse
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete"]
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
      - group: "apps"
        resources: ["deployments", "daemonsets", "statefulsets"]

  # 3. Log pod exec and portforward commands at Request level
  - level: Request
    resources:
      - group: ""
        resources: ["pods/exec", "pods/portforward"]

  # 4. Default baseline for all other authenticated requests
  - level: Metadata
```

### Step 2: Mount and Enable Audit Logging in `kube-apiserver.yaml`
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    volumeMounts:
    - mountPath: /etc/kubernetes/audit-policy.yaml
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes
      name: audit-log
  volumes:
  - hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: audit-log
```

---

## 4. Managed Kubernetes Hardening (AWS EKS & GCP GKE)

In cloud-managed control planes (EKS, GKE, AKS), you do not manage `/etc/kubernetes/manifests/kube-apiserver.yaml` directly:
- **Private Endpoint Enforcement**: Always set cluster API access to **Private Only** (accessible only via VPN / Transit Gateway) or restrict public access via strict CIDR IP whitelists.
- **Control Plane Logging**: In AWS EKS, enable all 5 control plane log streams: `api`, `audit`, `authenticator`, `controllerManager`, and `scheduler` forwarding to CloudWatch.

---

## 5. Verification & Troubleshooting Runbook

### Health & Compliance Checks
```bash
# 1. Verify anonymous authentication is disabled
kubectl get nodes --as=system:anonymous
# Expected: Error from server (Forbidden): User "system:anonymous" cannot list resource "nodes"

# 2. Tail live API server audit events for pod exec calls
tail -f /var/log/kubernetes/audit.log | jq 'select(.verb=="create" and .objectRef.subresource=="exec")'

# 3. Check kube-apiserver TLS negotiation
openssl s_client -connect <CONTROL_PLANE_IP>:6443 -tls1_3
```

### Common Pitfalls & Solutions
1. **Control plane fails to restart after editing `kube-apiserver.yaml`**:
   - Cause: YAML syntax error or path mismatch in volume mounts.
   - Solution: Kubelet manages static pods. Check kubelet logs: `sudo journalctl -u kubelet -e | grep apiserver`.
2. **Audit log exhausts master node disk space**:
   - Cause: Missing log rotation parameters (`--audit-log-maxsize` and `--audit-log-maxbackup`).
   - Solution: Always configure size caps and retention days, or stream logs directly to an external SIEM collector.

---

## 6. Key Takeaways
- The API server is the primary attack vector for Kubernetes; enforce `Node,RBAC` authorization and disable anonymous auth.
- Always include `NodeRestriction` in `--enable-admission-plugins` to prevent node lateral takeover.
- Enforce modern cryptographic standards with `--tls-min-version=VersionTLS13`.
- Configure comprehensive audit logging using declarative policies to track all RBAC and workload modifications.
