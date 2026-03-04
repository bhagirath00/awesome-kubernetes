# Kubernetes API Server

## Introduction

The Kubernetes API Server (`kube-apiserver`) is the single most important component in the entire cluster. Every single operation — whether you run `kubectl get pods`, a controller updates a resource, or a kubelet reports Pod status — goes through the API Server. There is no way to interact with the cluster that bypasses it.

Understanding the API Server deeply makes you a better Kubernetes operator. You will debug issues faster, write better automation, and understand why certain operations behave the way they do.

---

## The API Server as the Control Plane's Front Door

```
External Clients:           Internal Components:
  kubectl                     Scheduler
  CI/CD pipelines             Controller Manager
  Helm                        kubelet (on each node)
  Argo CD                     kube-proxy
  Prometheus                  CoreDNS
         │                           │
         └──────────────┬────────────┘
                        ▼
               ┌──────────────────┐
               │   API Server     │
               │  (kube-apiserver)│
               └────────┬─────────┘
                        │
                        ▼
               ┌──────────────────┐
               │      etcd        │
               └──────────────────┘
```

Every operation goes in through the API Server and out through etcd. Nothing goes around it. This design gives you a single place for security enforcement, audit logging, and rate limiting.

---

## The Four Stages of Every API Request

When any request hits the API Server, it passes through four stages in order. A request must pass all four stages, or it is rejected:

### Stage 1: Authentication — Who Are You?

The API Server verifies the identity of every request. Kubernetes supports multiple authentication methods simultaneously:

**X.509 Client Certificates**
The client presents a TLS certificate. The API Server verifies it was signed by the cluster's Certificate Authority (CA). This is how the Scheduler and Controller Manager authenticate.

```bash
# kubectl uses the certificate in your kubeconfig
cat ~/.kube/config | grep client-certificate-data
# (base64-encoded certificate)
```

**Bearer Tokens**
A long random string that represents an identity. Service Accounts use bearer tokens. Applications running in Pods use this method.

```bash
# A Pod's service account token is automatically mounted at:
cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

**AWS IAM Authentication (EKS)**
On EKS, you authenticate with your IAM identity. The `aws eks get-token` command (called automatically by `kubectl` via an exec credential plugin) exchanges your AWS credentials for a short-lived Kubernetes token.

```bash
# This is what happens when you run kubectl on EKS:
aws eks get-token --cluster-name my-cluster --region us-east-1

# Output:
# {
#   "kind": "ExecCredential",
#   "apiVersion": "client.authentication.k8s.io/v1beta1",
#   "status": {
#     "expirationTimestamp": "2024-01-01T12:15:00Z",
#     "token": "k8s-aws-v1.eyJ..."
#   }
# }
```

The API Server validates this token against the AWS IAM service. Your IAM identity is then mapped to a Kubernetes user or group in the `aws-auth` ConfigMap.

```bash
# The aws-auth ConfigMap maps IAM roles to Kubernetes RBAC
kubectl get configmap aws-auth -n kube-system -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789:role/eks-worker-node-role
      username: system:node:{{EC2PrivateDNSName}}
      groups:
      - system:bootstrappers
      - system:nodes
    - rolearn: arn:aws:iam::123456789:role/DevOpsTeamRole
      username: devops-user
      groups:
      - system:masters     # full admin access
```

### Stage 2: Authorization — Are You Allowed?

After authentication, the API Server checks if the authenticated identity has permission to perform the requested action on the requested resource. Kubernetes uses **RBAC (Role-Based Access Control)** for authorization.

```
Action: DELETE Pod named "web-pod" in namespace "production"

API Server checks:
Does user "alice" have a RoleBinding that grants "delete" on "pods" in "production"?
         │
    Yes → proceed
    No  → 403 Forbidden
```

RBAC is made of four objects:

**Role** — permissions for resources within a namespace:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

**ClusterRole** — same as Role, but cluster-wide (applies to non-namespaced resources like nodes):
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

**RoleBinding** — assigns a Role to a user, group, or service account:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-pod-reader
  namespace: production
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Testing RBAC** — check what a user or service account can do:
```bash
# Can alice delete Pods in production?
kubectl auth can-i delete pods -n production --as alice
# no

# Can alice list Pods in production?
kubectl auth can-i list pods -n production --as alice
# yes

# List all permissions for the current user
kubectl auth can-i --list -n production
```

### Stage 3: Admission Control — Should This Request Be Allowed?

Even after authentication and authorization, an admission controller can still reject or modify a request. Admission controllers enforce cluster-wide policies.

**Built-in admission controllers** (always running):

| Controller | What It Does |
|-----------|-------------|
| `NamespaceLifecycle` | Prevents creating resources in a terminating namespace |
| `LimitRanger` | Enforces resource limits on Pods in namespaces that have LimitRanges |
| `ServiceAccount` | Automatically attaches ServiceAccounts to Pods |
| `ResourceQuota` | Enforces namespace resource quotas |
| `PodSecurity` | Enforces Pod security standards (replaces PodSecurityPolicy) |
| `MutatingAdmissionWebhook` | Calls external webhooks that can modify objects |
| `ValidatingAdmissionWebhook` | Calls external webhooks that can reject objects |

**Webhook-based admission controllers** — tools like **OPA Gatekeeper** or **Kyverno** use the webhook mechanism to enforce custom policies:

```yaml
# Kyverno policy: all Pods must have resource limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-limits
spec:
  rules:
  - name: check-limits
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Resource limits are required for all containers"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"   # must have memory limit
                cpu: "?*"      # must have cpu limit
```

With this policy in place, any `kubectl apply` that creates a Pod without resource limits gets rejected by the API Server — even if the user has RBAC permission to create Pods.

### Stage 4: Validation and Persistence

After passing all three previous stages, the API Server:

1. Validates the object schema (required fields, correct types, valid values)
2. Applies defaulting (fills in fields you did not specify, like `imagePullPolicy: IfNotPresent`)
3. Writes the object to etcd
4. Returns the response to the client

---

## The Kubernetes REST API Structure

The API Server exposes a RESTful HTTP API. Every kubectl command is an HTTP request:

| kubectl Command | HTTP Method | API Path |
|----------------|-------------|----------|
| `kubectl get pods` | GET | `/api/v1/namespaces/default/pods` |
| `kubectl get pod web` | GET | `/api/v1/namespaces/default/pods/web` |
| `kubectl apply -f pod.yaml` | POST/PATCH | `/api/v1/namespaces/default/pods` |
| `kubectl delete pod web` | DELETE | `/api/v1/namespaces/default/pods/web` |
| `kubectl logs web` | GET | `/api/v1/namespaces/default/pods/web/log` |
| `kubectl exec web -- bash` | POST | `/api/v1/namespaces/default/pods/web/exec` |

You can call the API directly:

```bash
# Get the API Server URL from kubeconfig
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')

# Get a token for authentication
TOKEN=$(kubectl create token default --namespace default)

# Call the API directly
curl -s -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/api/v1/namespaces/default/pods" \
  --cacert /tmp/ca.crt | jq '.items[].metadata.name'
```

This is how all Kubernetes tools work internally — they build HTTP requests to the API Server. Understanding this makes it easier to write automation and debug issues.

---

## API Versioning and Stability

Kubernetes APIs are versioned. The version tells you how stable the API is:

| Version | Stability | Usage |
|---------|-----------|-------|
| `v1` | **Stable** — will not change in a breaking way | Use in production |
| `v1beta1` | **Beta** — mostly stable, may have minor changes | Generally safe |
| `v1alpha1` | **Alpha** — experimental, can be removed or changed | Do not use in production |

```yaml
# Stable API — safe for production
apiVersion: apps/v1
kind: Deployment

# Beta API — generally stable but check release notes
apiVersion: policy/v1beta1
kind: PodDisruptionBudget

# Use the stable version when it exists
apiVersion: policy/v1    # Graduated to v1 in K8s 1.21
kind: PodDisruptionBudget
```

```bash
# Always check which API version to use
kubectl explain deployment
# GROUP: apps
# VERSION: v1    ← use this

kubectl explain deployment --api-version=apps/v1
```

---

## API Server High Availability

The API Server is stateless (state is in etcd). This makes it trivial to run multiple instances. In a production cluster, you run at least 2 API Server instances behind a load balancer:

```
kubectl / external clients
         │
         ▼
  ┌──────────────┐
  │ Load Balancer │
  └──────┬───────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
API Server  API Server
  (node 1)  (node 2)
    │             │
    └──────┬──────┘
           │
           ▼
          etcd
```

On AWS EKS, this is handled automatically. The EKS control plane runs multiple API Server instances behind an AWS Network Load Balancer. The endpoint you see in your kubeconfig points to this NLB.

```bash
# Your EKS API Server endpoint is actually an NLB
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
# https://ABC123.gr7.us-east-1.eks.amazonaws.com
# This resolves to an NLB IP — multiple API Servers behind it
```

---

## API Server Audit Logging

The API Server can log every request for security and compliance. The audit log records:
- Who made the request (user/service account)
- What they requested (verb + resource)
- Whether it was allowed or denied
- The response code

On EKS, you enable control plane logging from the console or with Terraform:

```hcl
# Enable API Server audit logging on EKS
resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks.arn

  enabled_cluster_log_types = [
    "api",           # API Server logs (including audit)
    "audit",         # Audit logs specifically
    "authenticator", # IAM authentication logs
    "controllerManager",
    "scheduler"
  ]
}
```

Logs are sent to CloudWatch Logs under `/aws/eks/{cluster-name}/cluster`. You can search them with CloudWatch Insights:

```
# CloudWatch Insights query: find all DELETE operations on Pods
fields @timestamp, @message
| filter @logStream like "kube-apiserver-audit"
| filter verb == "delete" and objectRef.resource == "pods"
| sort @timestamp desc
| limit 50
```

---

## Interacting With the API Server — Useful Commands

```bash
# See all available API resources and their short names
kubectl api-resources

# See the full API documentation for any resource
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources

# Check API Server health endpoints
curl -k https://<api-server>/healthz     # overall health
curl -k https://<api-server>/livez      # liveness
curl -k https://<api-server>/readyz     # readiness

# Watch API: stream events from the API Server (like tail -f for Kubernetes)
kubectl get pods --watch

# Get raw API response for debugging
kubectl get pod web-pod -o json
kubectl get pod web-pod -o yaml

# See what kubectl is doing under the hood (HTTP requests)
kubectl get pods -v=6    # log level 6 shows HTTP method + URL
kubectl get pods -v=8    # log level 8 shows full request and response body
```

---

## Summary

- The API Server is the single entry point for all cluster operations — no component bypasses it
- Every API request passes through four stages: **Authentication** (who are you), **Authorization** (are you allowed), **Admission Control** (should this be allowed), **Validation + Persistence** (write to etcd)
- On EKS, authentication uses AWS IAM — your IAM role is mapped to a Kubernetes user/group in the `aws-auth` ConfigMap
- RBAC controls what each user, group, or service account can do — use `kubectl auth can-i` to test permissions
- Admission controllers (OPA Gatekeeper, Kyverno) enforce cluster-wide policies — they can reject or modify requests even after RBAC allows them
- The API Server exposes a standard REST API — every `kubectl` command is an HTTP request
- API versions (`v1`, `v1beta1`, `v1alpha1`) indicate stability — always use the stable version in production
- On EKS, multiple API Server instances run behind an NLB automatically
- Enable audit logging on EKS to record who did what to your cluster — logs go to CloudWatch

This completes Chapter 2 — Kubernetes Architecture. You now have a complete, deep understanding of how every component in a Kubernetes cluster works: the control plane (API Server, etcd, Scheduler, Controller Manager) and the worker nodes (kubelet, kube-proxy, containerd).

This also completes Module 1 — Introduction. You have covered what Kubernetes is, why it exists, how it compares to alternatives, where it fits in the DevOps pipeline, and the architecture that makes it all work.

In Module 2, you will move from theory to practice — installing `kubectl`, setting up your first EKS cluster with `eksctl` and Terraform, and deploying your first real application to Kubernetes on AWS.
