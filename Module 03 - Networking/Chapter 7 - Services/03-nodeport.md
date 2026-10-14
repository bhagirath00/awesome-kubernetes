# NodePort Service

## Introduction

A NodePort Service exposes your application on a **static port on every worker node** in the cluster. External traffic can reach the Service by hitting any node's IP on that port. NodePort is the simplest way to expose traffic externally, but it has limitations that make it unsuitable for most production deployments on AWS.

Understanding NodePort thoroughly helps you know when it is appropriate and when to use LoadBalancer or Ingress instead.

---

## How NodePort Works

When you create a NodePort Service, Kubernetes:
1. Assigns a ClusterIP (like a regular ClusterIP Service)
2. Allocates a port in the range **30000–32767** on every node
3. kube-proxy creates iptables rules to forward traffic from `<node-ip>:<nodePort>` to the backend Pods

```
External Client
      │
      ▼
Any Worker Node IP on port 30080
(e.g., 10.0.1.100:30080 or 10.0.1.101:30080)
      │
      ▼
kube-proxy iptables rule: forward to Service ClusterIP:80
      │
      ▼
Load balanced across backend Pods (10.0.1.45:8080, 10.0.1.46:8080)
```

The external client can hit **any node** in the cluster on the NodePort — even if that node has no Pods running for that Service. kube-proxy on that node will forward the traffic to a node that does have a Pod.

---

## Creating a NodePort Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service-nodeport
  namespace: production
spec:
  type: NodePort
  selector:
    app: web-server
  ports:
  - name: http
    port: 80            # ClusterIP port (internal)
    targetPort: 8080    # Pod port
    nodePort: 30080     # port exposed on every node (30000-32767)
                        # omit to let Kubernetes assign one automatically
    protocol: TCP
```

```bash
kubectl apply -f nodeport-service.yaml

kubectl get service web-service-nodeport -n production

# NAME                    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# web-service-nodeport    NodePort   172.20.45.124   <none>        80:30080/TCP   5m

# The PORT(S) column shows: 80:30080
# 80 = internal ClusterIP port
# 30080 = NodePort (external port on each node)
```

---

## Accessing a NodePort Service

```bash
# Get the external IPs of your worker nodes
kubectl get nodes -o wide

# NAME                         STATUS   ROLES    INTERNAL-IP    EXTERNAL-IP
# ip-10-0-1-100.ec2.internal   Ready    <none>   10.0.1.100     3.14.159.26
# ip-10-0-1-101.ec2.internal   Ready    <none>   10.0.1.101     3.14.159.27

# Access via any node's external IP
curl http://3.14.159.26:30080
curl http://3.14.159.27:30080   # same result — either node works

# On EKS, worker nodes are in private subnets — use internal IP from within VPC:
curl http://10.0.1.100:30080
```

---

## NodePort Port Range Configuration

The default NodePort range is 30000–32767. This range can be changed in the API Server configuration (not recommended on EKS — it is managed by AWS):

```bash
# On self-managed clusters, change the NodePort range:
# Add --service-node-port-range=20000-25000 to kube-apiserver arguments

# On EKS, the range is fixed at 30000-32767
```

You can either let Kubernetes auto-assign a NodePort (recommended) or specify one explicitly. If you specify a port that is already in use by another Service, the apply fails with a validation error.

---

## NodePort with an External Load Balancer (Manual)

In non-cloud environments, NodePort is often combined with an external load balancer (like HAProxy or Nginx) configured outside Kubernetes:

```
Internet → External Load Balancer (HAProxy)
               │
               ├─ Node 1 IP:30080
               ├─ Node 2 IP:30080
               └─ Node 3 IP:30080
                       │
               kube-proxy → backend Pods
```

The external LB distributes traffic across all node IPs on the NodePort. This is the on-premise equivalent of a cloud LoadBalancer Service.

On AWS, do not use this pattern — use the LoadBalancer Service type which provisions an AWS NLB automatically.

---

## The Problems with NodePort in Production

### Problem 1: Port Range Limitation

NodePort ports are in the 30000–32767 range. You cannot use standard ports like 80, 443, or 3306. External clients must access your application on an unusual port number.

### Problem 2: Node IP Changes

Worker node IPs can change (during node replacement, auto-scaling events). Your external load balancer must be updated. On AWS, auto-scaling replaces nodes regularly.

### Problem 3: No AWS Load Balancer Integration

NodePort does not automatically provision an AWS NLB or ALB. You manage traffic routing to nodes manually.

### Problem 4: Security Risk

NodePort opens a port on **every** worker node's security group. If your nodes have a public IP, this port is reachable from the internet — a potential security risk if not properly secured.

### Problem 5: External Traffic Policy

By default, when external traffic hits a node via NodePort, kube-proxy may route it to a Pod on a **different** node. This adds a network hop and changes the source IP (due to SNAT):

```yaml
spec:
  type: NodePort
  externalTrafficPolicy: Local    # only route to Pods on THIS node
  # Pros: preserves client IP, eliminates cross-node hop
  # Cons: traffic imbalance if Pods are unevenly distributed
```

With `externalTrafficPolicy: Local`, if a node has no Pods for this Service, traffic to that node's NodePort is dropped. Use this only with a health-check-aware load balancer that avoids nodes without Pods.

---

## When NodePort Is the Right Choice

| Scenario | Use NodePort? |
|----------|--------------|
| On-premise cluster, no cloud LB | ✅ Yes |
| Development environment, quick access | ✅ Yes |
| Production on AWS | ❌ Use LoadBalancer or Ingress |
| Exposing standard port 80/443 | ❌ Not possible |
| Cluster on a budget (no LB cost) | ✅ Acceptable compromise |
| Multi-tenant SaaS production | ❌ |

NodePort is also used internally by the LoadBalancer Service type — a LoadBalancer Service creates a NodePort as its backend and puts an AWS NLB in front of it.

---

## Summary

- NodePort exposes a Service on a fixed port (30000–32767) on every worker node in the cluster
- External traffic reaches the Service by hitting any node's IP on the NodePort — kube-proxy routes it to a healthy Pod regardless of which node is hit
- The `PORT(S)` column in `kubectl get service` shows `80:30080/TCP` — the ClusterIP port and the NodePort
- `externalTrafficPolicy: Local` eliminates cross-node hops and preserves the client source IP — but requires a health-check-aware load balancer to avoid routing to nodes without Pods
- NodePort is appropriate for on-premise clusters and development environments
- For production on AWS EKS, use the LoadBalancer Service type (which automatically provisions an AWS NLB) or Ingress (which provisions an ALB)

In the next file, `04-loadbalancer.md`, you will learn about the LoadBalancer Service type — how it automatically provisions an AWS NLB via the AWS Load Balancer Controller and how to configure it for production traffic.
