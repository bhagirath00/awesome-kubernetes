# Kubernetes Networking Overview

## Introduction

Networking in Kubernetes is one of the most important topics to understand deeply. Every Pod can communicate with every other Pod, every Service has a stable IP, and DNS works automatically across namespaces. But none of this happens by magic — it is built on a specific networking model enforced by every Kubernetes cluster.

This file gives you the complete mental model of Kubernetes networking before you go deep into the specific components in the files that follow.

---

## The Four Networking Problems Kubernetes Solves

Kubernetes networking addresses four distinct communication scenarios:

```
1. Container-to-Container     — containers within the same Pod
2. Pod-to-Pod                 — Pods on the same or different nodes
3. Pod-to-Service             — Pods reaching a stable Service VIP
4. External-to-Service        — traffic from outside the cluster
```

Each is handled differently, and each has a different mechanism. Let us go through each one.

---

## Problem 1: Container-to-Container Communication

Containers inside the same Pod share the same network namespace. They all have the same IP address and share all ports on that IP. They communicate via `localhost`:

```
Pod IP: 10.0.1.45

Container: app     →  binds to port 8080 on 10.0.1.45
Container: sidecar →  calls localhost:8080 to reach app

No network hop. No routing. Zero latency overhead.
```

```python
# Inside the sidecar container — calling the app container
import requests
response = requests.get("http://localhost:8080/healthz")
# 'localhost' resolves to the shared Pod IP — reaches the app container directly
```

The constraint: no two containers in the same Pod can bind to the same port (they share the port space).

---

## Problem 2: Pod-to-Pod Communication

Kubernetes requires that every Pod gets a unique, routable IP address. Any Pod must be able to reach any other Pod by IP — directly, without NAT (Network Address Translation).

This is the **flat network** requirement. It means:
- Pod A at `10.0.1.45` can reach Pod B at `10.0.2.67` directly
- No port mapping needed
- No NAT translation between source and destination
- Works whether the Pods are on the same node or different nodes

On AWS EKS, this is implemented by the **AWS VPC CNI Plugin** — each Pod gets a real VPC IP address from your subnet's IP range. Pod traffic uses the VPC routing fabric directly:

```
Node 1 (10.0.1.0/24 subnet)     Node 2 (10.0.2.0/24 subnet)
┌────────────────────────┐       ┌────────────────────────┐
│  Pod A: 10.0.1.45      │ ────► │  Pod B: 10.0.2.67      │
│  eth0: 10.0.1.100      │       │  eth0: 10.0.2.200      │
│  (node IP)             │       │  (node IP)             │
└────────────────────────┘       └────────────────────────┘
               Traffic goes through VPC routing — direct, no NAT
```

Each Pod IP is a secondary private IP address on the worker node's ENI (Elastic Network Interface) — it is a real VPC IP, routable within your VPC.

---

## Problem 3: Pod-to-Service Communication

Pods are ephemeral. Their IPs change every time they are recreated. You cannot build a reliable application that connects to specific Pod IPs.

**Services** solve this. A Service gets a stable **ClusterIP** — a virtual IP that never changes, even as the Pods behind it are replaced. When a Pod sends traffic to a Service's ClusterIP, **kube-proxy** intercepts it and redirects to one of the healthy Pod IPs:

```
Pod A wants to reach the database service

1. Pod A sends traffic to: 172.20.45.123:5432  (ClusterIP — stable)
2. kube-proxy intercepts (via iptables rules on the node)
3. kube-proxy selects a healthy backend Pod: 10.0.2.89:5432
4. Traffic is rewritten and forwarded to the actual Pod

Pod A thinks it talked to 172.20.45.123.
Actually talked to 10.0.2.89.
The database Pod's IP can change — ClusterIP never does.
```

---

## Problem 4: External-to-Service Communication

Traffic from outside the cluster (from the internet, or from other AWS services) needs a way in. Kubernetes provides three mechanisms:

**NodePort** — exposes a port on every node's external IP. External traffic hits any node on that port and is forwarded to the Service.

**LoadBalancer** — provisions a cloud load balancer (AWS NLB or ALB) that distributes traffic to the Service.

**Ingress** — an HTTP/HTTPS router that sits in front of multiple Services and routes based on hostname and URL path.

For production on AWS, the LoadBalancer type (via AWS Load Balancer Controller) and Ingress (via AWS ALB Ingress Controller) are the standard patterns.

---

## The Kubernetes Networking Model — Three Rules

The Kubernetes networking model imposes three requirements on any network implementation (CNI plugin):

**Rule 1: Every Pod gets a unique IP.**
No two Pods in the cluster share an IP address.

**Rule 2: All Pods can communicate with all other Pods without NAT.**
Pod A can reach Pod B directly by IP — no port mapping, no NAT, no proxy.

**Rule 3: Agents on a node (kubelet, kube-proxy) can communicate with all Pods on that node.**
The node's host network can reach Pod IPs.

These three rules define the contract. Any network plugin (CNI) that satisfies them is valid for Kubernetes. Different CNI plugins implement this contract differently — AWS VPC CNI, Calico, Flannel, Cilium — but all deliver the same result.

---

## IP Address Ranges in a Kubernetes Cluster

A cluster has three IP address ranges that must not overlap:

| Range | Purpose | Who Configures It |
|-------|---------|------------------|
| **Node CIDR** | IP addresses for worker nodes | Your VPC subnet configuration |
| **Pod CIDR** | IP addresses for Pods | CNI plugin / cluster configuration |
| **Service CIDR** | IP addresses for Services (ClusterIPs) | kube-apiserver configuration |

On EKS with the AWS VPC CNI:

```bash
# Check your cluster's CIDR ranges
aws eks describe-cluster --name my-cluster --region us-east-1 \
  --query 'cluster.kubernetesNetworkConfig'

# Output:
# {
#   "serviceIpv4Cidr": "172.20.0.0/16",
#   "ipFamily": "ipv4"
# }

# Pod CIDR = the VPC subnet CIDR ranges of your worker node subnets
# (because EKS VPC CNI assigns VPC IPs to Pods directly)

# See all node IPs
kubectl get nodes -o wide

# See all Pod IPs
kubectl get pods --all-namespaces -o wide
```

These ranges must not overlap. If `172.20.0.0/16` is your Service CIDR, your VPC subnets (which become Pod IPs) must use a different range like `10.0.0.0/8`.

---

## Port Concepts in Kubernetes Networking

Several different "port" concepts appear in Kubernetes. Understanding each one prevents confusion:

```yaml
spec:
  containers:
  - name: web
    ports:
    - containerPort: 8080    # port the CONTAINER process listens on
                             # purely informational — does not open/close anything
---
apiVersion: v1
kind: Service
spec:
  ports:
  - port: 80                 # port clients use to reach the SERVICE (ClusterIP:80)
    targetPort: 8080         # port on the Pod to forward traffic to
    nodePort: 30080          # port exposed on each NODE's IP (NodePort only)
    protocol: TCP
```

```
External Client → NodePort (30080 on Node IP)
                      ↓
           Service ClusterIP (port 80)
                      ↓
              Pod (targetPort 8080)
                      ↓
           Container process listening on 8080
```

---

## Network Traffic Flow Diagram — End to End

Here is the complete flow for an external HTTP request reaching a Pod in EKS:

```
Internet
    │
    ▼
AWS Application Load Balancer (ALB)
    │  (created by AWS ALB Ingress Controller)
    ▼
Ingress resource routing
    │  (host: api.mycompany.com, path: /api/*)
    ▼
Service (ClusterIP: 172.20.45.123, port 80)
    │
    ▼
kube-proxy iptables rules on the node
    │  (rewrites destination to a real Pod IP)
    ▼
Pod (10.0.1.45:8080) on Worker Node 1
    │
    ▼
Container process inside the Pod
```

Each layer adds something:
- **ALB** handles TLS termination, health checks, and cross-AZ balancing
- **Ingress** handles HTTP routing by host and path
- **Service** provides a stable VIP and load balances across Pods
- **kube-proxy** performs the actual IP rewriting on each node

---

## Summary

- Kubernetes networking handles four scenarios: container-to-container (localhost), Pod-to-Pod (flat network, no NAT), Pod-to-Service (ClusterIP via kube-proxy), and External-to-Service (LoadBalancer/Ingress)
- The core rule: every Pod gets a unique IP, and all Pods can reach each other without NAT — this is the flat network model
- On AWS EKS, the VPC CNI plugin gives each Pod a real VPC IP address — Pod traffic uses the VPC routing fabric directly
- Three non-overlapping CIDR ranges: Node CIDR (VPC subnets), Pod CIDR (Pod IPs), Service CIDR (ClusterIPs)
- `containerPort` is informational only; `port` is the Service port; `targetPort` is the Pod port; `nodePort` is the node-level port
- End-to-end flow: Internet → ALB → Ingress → Service → kube-proxy → Pod IP → Container

In the next file, `02-pod-to-pod-communication.md`, you will go deep into how two Pods on different nodes communicate — the route taken, how the AWS VPC CNI assigns IPs, and what happens at each network hop.
