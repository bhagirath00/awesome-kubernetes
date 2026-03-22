# Kubernetes Security Best Practices

## Introduction

Now that you understand the attack surface, it is time to build defenses. Security best practices are concrete, actionable steps that reduce your attack surface and make it dramatically harder for attackers to compromise your cluster.

The practices in this file are not theoretical — they are what security teams at companies like Google, Netflix, and Amazon use to protect their production Kubernetes clusters. If you implement these practices, you will be ahead of the vast majority of Kubernetes deployments.

---

## Principle 1: The Defense in Depth

Never rely on a single security control. If one control fails, others should still protect you.

For example:
- Control 1: Don't expose API server to internet (network firewall)
- Control 2: Require strong authentication (username/password + MFA)
- Control 3: Audit all API access (logs show who accessed what)

If someone bypasses the firewall (Control 1 fails), strong authentication still protects you (Control 2). If authentication is compromised, logs still reveal the breach (Control 3).

---

## Principle 2: Least Privilege

Every user, pod, and service account should have the minimum permissions they need to do their job. Not more.

**Example:** A frontend pod does not need permission to delete deployments. If its service account has that permission and the pod is compromised, the attacker can delete deployments. If the pod only has permission to read ConfigMaps and Secrets it needs, the blast radius is limited.

---

## Best Practice 1: Authentication & Authorization

### Use Strong Authentication

**Never rely on default credentials.**

```bash
# WRONG — default Kubernetes auth (insecure)
kube-apiserver --basic-auth-file=/etc/kubernetes/auth.csv

# CORRECT — use certificate-based auth or external provider
kube-apiserver \
  --client-ca-file=/etc/kubernetes/ca.crt \
  --oidc-issuer-url=https://accounts.google.com \
  --oidc-client-id=yourapp.apps.googleusercontent.com
```

For production:
- Use certificate-based authentication (mTLS) for service-to-service communication
- Use OIDC (OpenID Connect) or similar for human users
- Enable multi-factor authentication for sensitive access

### Implement RBAC

RBAC (Role-Based Access Control) restricts what authenticated users can do. Chapter 21 covers this in depth, but the principle is simple: grant the minimum permissions needed.

```yaml
# Example: A developer should only manage deployments in the dev namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: dev
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "pods/logs"]
  verbs: ["get", "list", "watch"]
  # Notice: no "delete" for deployments, no access to secrets, no cluster-wide access
```

---

## Best Practice 2: Pod Security

### Run Containers as Non-Root

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
```

**What this does:**
- `runAsNonRoot: true` — Kubernetes rejects the pod if it tries to run as root
- `runAsUser: 1000` — Runs as UID 1000 (not root)
- `allowPrivilegeEscalation: false` — Cannot use sudo or setuid binaries
- `readOnlyRootFilesystem: true` — Root filesystem is read-only (prevents modifications)
- `capabilities: drop ALL` — Remove all Linux capabilities
- `capabilities: add NET_BIND_SERVICE` — Add back only what's needed (in this case, ability to bind to port 1024)

**Result:** Even if the application is compromised, the attacker:
- Has no root privileges
- Cannot modify application files
- Cannot install tools or backdoors
- Cannot escalate privileges

### Remove Unnecessary Capabilities

Linux capabilities are fine-grained permissions (separate from traditional "root user" concept). Most applications don't need them.

```yaml
# WRONG — most containers keep default capabilities
apiVersion: v1
kind: Pod
metadata:
  name: vulnerable
spec:
  containers:
  - name: app
    image: myapp:latest
    # No securityContext — keeps default capabilities

---
# CORRECT — drop all, add back only what's needed
apiVersion: v1
kind: Pod
metadata:
  name: secure
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE  # Only if you need to bind to ports < 1024
```

**Common capabilities you might need:**
- `NET_BIND_SERVICE` — bind to ports < 1024
- `SYS_ADMIN` — advanced system operations (rarely needed in containers)
- `SYS_PTRACE` — debug other processes (only for debugging)

**By default:** Drop everything. Add back only what testing proves you need.

---

## Best Practice 3: Network Policies

Network Policies restrict which pods can communicate with which other pods. This is your primary defense against lateral movement.

```yaml
# Step 1: Deny all traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Step 2: Allow frontend pods to receive traffic from internet
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # Anyone can send traffic (frontend is internet-facing)
    ports:
    - protocol: TCP
      port: 80

---
# Step 3: Allow frontend to call backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# Step 4: Allow backend to call database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

**Result:** Communication flows in one direction (frontend → backend → database). If frontend is compromised, the attacker cannot go backward to the database (because there's no network path). They can only go forward to backend (which you allow).

---

## Best Practice 4: Secrets Management

### Use Encrypted Secrets at Rest

Kubernetes Secrets are base64-encoded, not encrypted. Enable encryption at rest:

```yaml
# Configure kube-apiserver to encrypt secrets at rest
kube-apiserver \
  --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  provider:
  - name: aescbc
    aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-secret>
  - name: identity
    identity: {}
```

Now secrets are encrypted when stored in etcd.

### Don't Log Secrets

```yaml
# WRONG — secrets in logs
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_PASSWORD
      value: "SuperSecret123!"  # Visible in pod definition
```

**Correct approach:**

```yaml
# Store in Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  password: "SuperSecret123!"

---
# Reference in pod
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

### Use External Secret Managers (For Production)

For production, avoid storing secrets in Kubernetes at all. Use:
- AWS Secrets Manager
- HashiCorp Vault
- Azure Key Vault
- Google Secret Manager

Your pods authenticate to these services and fetch secrets at runtime (not stored in etcd).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: AWS_ROLE_ARN
      value: arn:aws:iam::123456789012:role/app-role
    - name: AWS_WEB_IDENTITY_TOKEN_FILE
      value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          audience: sts.amazonaws.com
          expirationSeconds: 86400
          path: token
```

Pod uses its identity to fetch secrets from AWS Secrets Manager. No secrets stored in Kubernetes.

---

## Best Practice 5: Image Security

### Scan Images for Vulnerabilities

Use tools like Trivy, Snyk, or Clair:

```bash
# Scan image before pushing
trivy image myregistry/myapp:v1.0

# Output:
# ...
# nginx (CRITICAL): CVE-2021-1234 - Remote Code Execution
# ...
```

Never push images with critical vulnerabilities.

### Use Private Registries

```yaml
# WRONG — pushing to public DockerHub
docker tag myapp:v1 myusername/myapp:v1
docker push myusername/myapp:v1

# Now anyone can pull your image
```

```yaml
# CORRECT — use private registry (ECR, GCR, DockerHub private)
docker tag myapp:v1 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1

# Control who can pull using IAM
```

### Sign Images

Use image signing to verify authenticity:

```bash
# Sign image with private key
cosign sign --key cosign.key myregistry/myapp:v1

# Verify signature before running
cosign verify --key cosign.pub myregistry/myapp:v1
```

### Enforce Image Pull Policy

```yaml
# Always pull fresh image (prevents using stale cached image)
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myregistry/myapp:v1
    imagePullPolicy: Always  # Always pull, never use cached
```

---

## Best Practice 6: Audit Logging

Track who did what and when:

```yaml
# Configure kube-apiserver audit logging
kube-apiserver \
  --audit-log-path=/var/log/audit.log \
  --audit-log-maxage=7 \
  --audit-log-maxbackup=3 \
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml
```

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  verbs: ["delete", "create", "update", "patch"]
  resources: ["*"]
  omitStages:
  - RequestReceived
- level: Metadata
  verbs: ["get", "list", "watch"]
  resources: ["*"]
```

**Result:** Every action is logged. If something goes wrong, you can see exactly who did what and when.

---

## Best Practice 7: Resource Limits

Prevent resource starvation:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
```

- **Requests** — minimum guaranteed resources
- **Limits** — maximum usable resources

If a pod tries to use more than the limit, Kubernetes kills it (before it starves the entire cluster).

---

## Implementation Checklist

Use this checklist to audit your cluster:

- [ ] API server not exposed to internet
- [ ] Strong authentication configured (OIDC or mTLS)
- [ ] RBAC roles follow least privilege principle
- [ ] All pods run as non-root with securityContext
- [ ] Network Policies implemented (default deny, explicit allow)
- [ ] Secrets encrypted at rest
- [ ] External secret manager used for production secrets
- [ ] Images scanned for vulnerabilities
- [ ] Images stored in private registry
- [ ] All pods have resource requests and limits
- [ ] Audit logging enabled
- [ ] Regular security audits (monthly minimum)

---

## Summary

Kubernetes security is built on layers: strong authentication, RBAC authorization, pod security controls, network isolation, encrypted secrets, and audit logging. Together, these practices make it extremely difficult for attackers to compromise your cluster.

The best practices here follow the principle of **defense in depth** — no single control is perfect, but multiple controls layered together create a robust security posture.

You now have the conceptual foundation for Kubernetes security. In Chapter 21, we dive deep into the **RBAC (Role-Based Access Control)** system — the mechanism that enforces "who can do what" in your cluster. This is where you translate these security principles into concrete Kubernetes resources.
