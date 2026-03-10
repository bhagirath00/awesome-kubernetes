# Cluster Networking Model

## Introduction

The Kubernetes cluster networking model defines how traffic flows between all components in a cluster — Pods, Services, nodes, and external clients. At the heart of this model is how **kube-proxy** uses Linux **iptables** rules to implement Service routing. Understanding this layer explains why Kubernetes networking behaves the way it does and gives you the skills to debug even the most subtle network issues.

---

## The Service IP is Virtual — There is No Process Listening on It

This is the most important thing to understand about Services. When you create a Service with ClusterIP `172.20.45.123`, there is no process anywhere in the cluster listening on that IP. It does not exist as a real network interface.

The ClusterIP only exists as iptables rules in the kernel on every node. When a Pod sends traffic to `172.20.45.123`, the kernel's netfilter (iptables) intercepts the packet **before it is routed** and rewrites the destination IP to a real Pod IP.

```bash
# Prove there is no real listener on the ClusterIP
kubectl get service web-service -n production
# NAME          TYPE        CLUSTER-IP      PORT(S)
# web-service   ClusterIP   172.20.45.123   80/TCP

# From any node — no process listening on this IP:
ss -tnlp | grep 172.20.45.123
# (no output — no process is listening)

# But traffic to this IP works because iptables intercepts it
curl http://172.20.45.123:80
# (gets a response from one of the backend Pods)
```

---

## How kube-proxy Creates iptables Rules

When you create a Service, kube-proxy on every node creates a chain of iptables rules:

```
New request to 172.20.45.123:80
         │
         ▼
PREROUTING chain → KUBE-SERVICES chain
         │
         ▼
KUBE-SVC-XXXXXXXX chain (one per Service)
         │
    ┌────┴─────────────────┐
    │                      │
    ▼ (33% probability)    ▼ (67% probability)
KUBE-SEP-AAAA             KUBE-SVC-XXXXXXXX (continue)
(endpoint 10.0.1.45:8080)      │
                               ▼ (50% of 67% = 33%)
                          KUBE-SEP-BBBB
                          (endpoint 10.0.1.46:8080)
                               │
                               ▼ (remaining = 33%)
                          KUBE-SEP-CCCC
                          (endpoint 10.0.2.67:8080)
```

Each `KUBE-SEP-*` rule is a DNAT (Destination NAT) rule — it rewrites the destination IP from the ClusterIP to the actual Pod IP.

```bash
# View iptables rules kube-proxy creates (on a worker node)
# SSH into a node first
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers

# Find the rules for your Service
sudo iptables -t nat -L -n | grep 172.20.45.123

# See the endpoint rules for a specific Service
sudo iptables -t nat -L KUBE-SVC-XXXXXXXX -n
```

---

## The Complete Traffic Flow — Pod to Service

Let us trace a complete network request from Pod A to a Service with 3 Pods:

```
Pod A (10.0.1.45) sends HTTP request to web-service:
  Source IP:      10.0.1.45
  Destination IP: 172.20.45.123 (Service ClusterIP)
  Destination Port: 80

Step 1: Packet leaves Pod A's eth0

Step 2: Packet enters Node 1's network namespace
        Linux netfilter (iptables) checks PREROUTING chain

Step 3: KUBE-SERVICES chain matches destination 172.20.45.123:80
        Selects KUBE-SVC-XXXXXXXX chain

Step 4: Random selection picks endpoint 10.0.2.67:8080 (Pod C on Node 2)

Step 5: DNAT rule rewrites the packet:
        Source IP:      10.0.1.45  (unchanged)
        Destination IP: 10.0.2.67  (Pod C's real IP)
        Destination Port: 8080      (Pod C's port)

Step 6: Packet is now routed normally — VPC routes it to Node 2

Step 7: Node 2 receives packet, delivers to Pod C via veth pair

Step 8: Pod C responds:
        Source IP: 10.0.2.67:8080
        Destination IP: 10.0.1.45

Step 9: Response goes back through iptables (POSTROUTING chain)
        Conntrack table remembers the DNAT translation
        Source IP rewritten back to 172.20.45.123:80

Step 10: Pod A receives response from 172.20.45.123:80
         Pod A is unaware of Pod C — it only ever saw the ClusterIP
```

The **conntrack** (connection tracking) table is critical — it remembers which DNAT translation was applied to which connection, so the response packet gets the correct reverse translation.

---

## iptables Load Balancing — How It Distributes Traffic

kube-proxy achieves load balancing through **probabilistic rules**. For a Service with 3 endpoints:

- Rule 1: 33.3% probability → forward to Pod 1
- Rule 2: 50% probability of remaining → forward to Pod 2 (50% of 66.7% = 33.3%)
- Rule 3: 100% of remaining → forward to Pod 3 (33.3%)

Net result: each Pod receives approximately equal traffic. This is random round-robin, not strict round-robin — the actual distribution varies slightly but averages to 1/N per Pod.

The load balancing is **per connection**, not per request. Once a TCP connection is established to a specific Pod, all requests on that connection go to the same Pod (because the DNAT rule is established and conntrack tracks the session).

```bash
# Watch which Pod endpoints are receiving traffic
kubectl get endpoints web-service -n production -w

# NAME          ENDPOINTS                                    AGE
# web-service   10.0.1.45:8080,10.0.1.46:8080,10.0.2.67:8080  5m

# When a Pod is deleted, its IP is removed from endpoints automatically
# kube-proxy updates iptables rules — no more traffic to that IP
```

---

## IPVS Mode — An Alternative to iptables

For very large clusters (thousands of Services), iptables has a performance problem: it does linear rule traversal. A cluster with 10,000 Services creates hundreds of thousands of iptables rules, and every new connection must scan through all of them.

**IPVS (IP Virtual Server)** solves this by using a hash table instead of linear rules. Lookups are O(1) regardless of Service count:

```bash
# Check if IPVS mode is active
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# Enable IPVS mode (requires kernel modules)
# Edit the kube-proxy ConfigMap:
kubectl edit configmap kube-proxy -n kube-system

# Change:
# mode: ""
# To:
# mode: "ipvs"
# ipvs:
#   scheduler: "rr"   # round-robin (rr), least-connection (lc), or random (rnd)
```

IPVS also supports more load balancing algorithms: round-robin, least connection, destination hashing, source hashing, and more.

---

## Network Namespaces — The Isolation Mechanism

Linux **network namespaces** are what give each Pod its own isolated network environment. Each Pod runs in its own network namespace — it has its own routing table, iptables rules, and network interfaces.

```bash
# On a worker node, list all network namespaces
ip netns list

# Output:
# 87654321-4321-4321-4321 (id: 5)   ← Pod 1's network namespace
# 12345678-1234-1234-1234 (id: 4)   ← Pod 2's network namespace
# cni-abcdefgh-1234-5678-abcd (id: 3)  ← CNI created namespace

# Run a command inside a Pod's network namespace
kubectl exec web-pod-abc123 -- ip addr show
# Inside the Pod's namespace, only the Pod's eth0 and loopback are visible
# Node interfaces are not visible

# Verify the Pod's IP
kubectl exec web-pod-abc123 -- ip addr show eth0
# inet 10.0.1.45/32 scope global eth0
```

The `/32` prefix (on EKS with VPC CNI) means each Pod IP is a host route — the Pod sees its IP as a single host address, not a subnet.

---

## Checking Cluster Networking Health

```bash
# Test DNS resolution (CoreDNS must be working)
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local

# Test Pod-to-Service connectivity
kubectl run conn-test --image=busybox --rm -it --restart=Never -- \
  wget -T 5 -O- http://web-service.production.svc.cluster.local

# Check that kube-proxy is running on every node
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide

# Check kube-proxy logs for errors
kubectl logs -n kube-system -l k8s-app=kube-proxy --tail=50

# Verify Service endpoints are populated
kubectl get endpoints -n production

# NAME          ENDPOINTS
# web-service   10.0.1.45:8080,10.0.1.46:8080,10.0.2.67:8080

# If endpoints are empty (<none>), the Service selector doesn't match any Pods
# Check labels:
kubectl get pods -n production --show-labels
kubectl get service web-service -n production -o yaml | grep -A 3 selector
```

---

## Summary

- Service ClusterIPs are **virtual** — no process listens on them. They exist only as iptables rules in the Linux kernel
- kube-proxy creates KUBE-SERVICES and KUBE-SEP-* iptables chains that perform DNAT — rewriting Service ClusterIPs to real Pod IPs before routing
- iptables load balancing is probabilistic per-connection, not per-request — connections are sticky to one Pod while they remain open
- Linux **conntrack** tracks DNAT translations so that response packets are correctly reverse-NATted back to the ClusterIP
- IPVS mode replaces iptables with a hash table for O(1) Service lookups — preferred for large clusters with thousands of Services
- Each Pod runs in its own Linux network namespace — it has isolated routing tables and network interfaces
- Debug networking issues by checking: endpoints (`kubectl get endpoints`), kube-proxy logs, and iptables rules on the node

In the next file, `04-cni-plugins.md`, you will learn about CNI (Container Network Interface) plugins — what they are, how they implement the Kubernetes network model, and how AWS VPC CNI compares to Calico, Flannel, and Cilium.
