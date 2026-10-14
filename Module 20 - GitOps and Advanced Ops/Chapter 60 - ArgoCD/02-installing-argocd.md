# Installing and Configuring Production ArgoCD

## 1. Architectural Concept & Deep Dive

Deploying ArgoCD in an enterprise production environment requires a High-Availability (HA) topology capable of surviving node loss, handling massive Git repository rendering loads, and providing zero-downtime upgrades.

The official Argo project maintains two deployment models:
1. **Non-HA (`install.yaml`)**: Single replica per component, standalone in-memory Redis. Suitable only for local dev (Minikube / Kind).
2. **High-Availability (`ha/install.yaml` or Helm HA)**: Multi-replica API servers, Redis Sentinel failover cluster, horizontally scaled Repo Servers, and Controller sharding.

```
+─────────────────────────────────────────────────────────────────────────+
|                       ARGOCD HIGH-AVAILABILITY TOPOLOGY                 |
|                                                                         |
|                [ Ingress Controller (TLS & gRPC Passthrough) ]          |
|                                       │                                 |
|         ┌─────────────────────────────┼─────────────────────────────┐   |
|         ▼                             ▼                             ▼   |
|  [ argocd-server 1 ]          [ argocd-server 2 ]          [ argocd-server 3 ]
|         │                             │                             │   |
|         └─────────────────────────────┼─────────────────────────────┘   |
|                                       ▼                                 |
|                       [ Redis Sentinel HA Cluster ]                     |
|                       (1 Primary + 2 Replicas + 3 Sentinels)            |
|                                       │                                 |
|         ┌─────────────────────────────┴─────────────────────────────┐   |
|         ▼                                                           ▼   |
|  [ argocd-repo-server (HPA 2-10) ]                [ Application Controller ]
|  • Clones Git & renders templates                 • Sharded across replicas
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Production Helm Installation

The recommended production deployment method is the official `argo-cd` Helm chart.

### Step 1: Add Helm Repository
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### Step 2: Create Production Values File (`argocd-ha-values.yaml`)
```yaml
# 1. High Availability Redis with Sentinel
redis-ha:
  enabled: true
  haproxy:
    enabled: true
    replicas: 3

# 2. Production Controller Sharding
controller:
  replicas: 2
  env:
    - name: ARGOCD_CONTROLLER_REPLICAS
      value: "2"
  resources:
    limits:
      cpu: 2000m
      memory: 2048Mi
    requests:
      cpu: 500m
      memory: 1024Mi

# 3. Scalable Repo Server (Template Rendering Engine)
repoServer:
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
  resources:
    limits:
      cpu: 2000m
      memory: 2048Mi
    requests:
      cpu: 200m
      memory: 512Mi

# 4. API Server HA & Ingress Configuration
server:
  replicas: 3
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
      # Critical: Support gRPC for the argocd CLI!
      nginx.ingress.kubernetes.io/server-snippet: |
        grpc_read_timeout 300s;
        grpc_send_timeout 300s;
    hosts:
      - argocd.mycompany.com
    tls:
      - secretName: argocd-tls-certificate
        hosts:
          - argocd.mycompany.com
```

### Step 3: Execute Installation
```bash
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  -f argocd-ha-values.yaml
```

---

## 3. Initial Administrative Login & Password Retrieval

Upon initial deployment, ArgoCD generates a random administrative password stored in a Kubernetes Secret:

### Step 1: Retrieve the Initial Admin Password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### Step 2: Install the ArgoCD CLI Tool
```bash
# On Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

### Step 3: Log in via CLI
```bash
argocd login argocd.mycompany.com --username admin --password <INITIAL_PASSWORD>
```

### Step 4: Update the Admin Password and Delete Initial Secret
```bash
# Change password
argocd account update-password

# Delete initial secret (Security Best Practice!)
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## 4. Ingress Configuration: The gRPC Passthrough Requirement

The ArgoCD Web UI operates over standard HTTPS, but the **ArgoCD CLI communicates over gRPC**.

If you use Nginx Ingress Controller, ensure the following configuration is applied to prevent CLI connection drops:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true" # Or use separate gRPC ingress
```

Alternatively, deploy a dedicated gRPC ingress rule:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-grpc
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: nginx
  rules:
    - host: grpc.argocd.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
```

---

## 5. Verification & Troubleshooting Runbook

### Health Verification Commands
```bash
# Verify all pods are running across nodes
kubectl get pods -n argocd -o wide

# Check Redis Sentinel cluster status
kubectl exec -it -n argocd statefulset/argocd-redis-ha-server -c redis -- redis-cli -p 26379 sentinel masters

# Inspect ArgoCD Server logs for authentication or webhook errors
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=50
```

### Common Pitfalls & Solutions
1. **`argocd login` fails with `rpc error: code = Unavailable desc = connection error`**:
   - Cause: Ingress controller is buffering or terminating gRPC traffic improperly.
   - Solution: Use the `--grpc-web` flag with the CLI (`argocd login ... --grpc-web`) to tunnel gRPC over standard HTTP/1.1 or HTTP/2.
2. **Repo Server CPU spikes to 100% and crashes**:
   - Cause: Many applications triggering simultaneous Git clones and Helm template calculations.
   - Solution: Ensure horizontal autoscaling (HPA) is active and scale `redis-ha` cache memory limits.

---

## 6. Key Takeaways
- Enterprise production requires the HA topology: Redis Sentinel, multi-replica API servers, and HPA-scaled Repo Servers.
- The initial admin password must be retrieved from `argocd-initial-admin-secret` and deleted once rotated.
- The ArgoCD CLI requires gRPC support; configure Nginx Ingress with `backend-protocol: "GRPC"` or use `--grpc-web`.
