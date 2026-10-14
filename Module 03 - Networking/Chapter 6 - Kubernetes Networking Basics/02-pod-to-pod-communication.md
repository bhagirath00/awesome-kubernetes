# Pod-to-Pod Communication

## Introduction

When Pod A on Node 1 needs to talk to Pod B on Node 2, the traffic must travel from one node's network to another. This seems straightforward, but the implementation is specific to the CNI plugin your cluster uses. On AWS EKS, the VPC CNI plugin makes this remarkably clean — Pod traffic uses the VPC routing fabric directly, with no overlay network.

This file explains exactly how Pod-to-Pod communication works on EKS, what happens at each hop, and how to observe and debug Pod connectivity issues.

---

## Same-Node Pod Communication

When two Pods are on the same node, communication is handled by a **virtual ethernet pair (veth)** and a **Linux bridge** on the node:

```
Pod A (10.0.1.45)          Pod B (10.0.1.46)
     │                           │
   eth0                        eth0
   veth-a ─────────────────── veth-b
              Linux Bridge
              (cbr0 / docker0)
```

Each Pod has a network interface `eth0` inside its network namespace. This is paired with a `veth` (virtual ethernet) interface on the node's network namespace. All Pod veth interfaces connect to a bridge on the node.

When Pod A sends traffic to `10.0.1.46`:
1. Traffic leaves Pod A's `eth0`
2. Goes through `veth-a` to the Linux bridge on the node
3. Bridge has a forwarding table entry for `10.0.1.46` → `veth-b`
4. Traffic arrives at Pod B's `eth0`

This is pure Layer 2 switching — very fast, no routing needed for same-node communication.

---

## Cross-Node Pod Communication — AWS VPC CNI

On EKS with the VPC CNI plugin, the implementation is different from most other CNI plugins. Instead of an overlay network (where Pod traffic is encapsulated and tunneled), the VPC CNI assigns **real VPC IP addresses** to Pods:

```
Node 1                                Node 2
VPC subnet: 10.0.1.0/24               VPC subnet: 10.0.2.0/24
Node IP: 10.0.1.100                    Node IP: 10.0.2.200

Pod A: 10.0.1.45  ─────────────────►  Pod B: 10.0.2.67
       (real VPC IP)                          (real VPC IP)
                    VPC routing table
                    10.0.2.0/24 → 10.0.2.200
```

The traffic path for Pod A (10.0.1.45) reaching Pod B (10.0.2.67):

1. **Pod A sends packet** destined for 10.0.2.67
2. **Node 1's routing table** — packet exits via the node's ENI (Elastic Network Interface)
3. **VPC routing** — VPC knows 10.0.2.0/24 is in subnet us-east-1b, routes to Node 2
4. **Node 2 receives packet** on its ENI
5. **Node 2's routing table** — sees 10.0.2.67 is a secondary IP on one of its ENIs
6. **Traffic delivered to Pod B** via veth pair

No encapsulation. No tunnel headers. The packets look like regular VPC traffic.

---

## How the AWS VPC CNI Assigns Pod IPs

The VPC CNI plugin runs as a DaemonSet (`aws-node`) on every worker node. It manages IP address assignment to Pods:

```bash
# The VPC CNI DaemonSet
kubectl get pods -n kube-system -l k8s-app=aws-node

# NAME             READY   STATUS    RESTARTS   AGE
# aws-node-4xkqp   1/1     Running   0          30d
# aws-node-7bmpv   1/1     Running   0          30d
# aws-node-9ztrl   1/1     Running   0          30d
```

How IP assignment works:

**Step 1: ENI Attachment**
The VPC CNI attaches additional ENIs to the worker node EC2 instance. Each ENI can hold multiple secondary private IPs.

**Step 2: IP Pre-allocation**
The VPC CNI pre-allocates a pool of secondary private IPs on those ENIs — ready to assign to new Pods without delay.

**Step 3: Pod IP Assignment**
When a new Pod is created on the node, the VPC CNI assigns one of the pre-allocated IPs to the Pod.

```bash
# See the secondary IPs assigned to a node's ENIs
aws ec2 describe-instances \
  --instance-ids i-0abc123def456789 \
  --query 'Reservations[].Instances[].NetworkInterfaces[].PrivateIpAddresses[]'

# Output:
# [
#   {"Primary": true,  "PrivateIpAddress": "10.0.1.100"},   ← node IP
#   {"Primary": false, "PrivateIpAddress": "10.0.1.45"},    ← Pod A IP
#   {"Primary": false, "PrivateIpAddress": "10.0.1.46"},    ← Pod B IP
#   {"Primary": false, "PrivateIpAddress": "10.0.1.47"}     ← available for new Pod
# ]
```

### IP Address Limits Per Node

Each EC2 instance type supports a maximum number of ENIs and secondary IPs. This directly limits how many Pods can run on a node:

| Instance Type | Max ENIs | Max IPs per ENI | Max Pods (approx) |
|--------------|----------|-----------------|-------------------|
| t3.small | 3 | 4 | 11 |
| t3.medium | 3 | 6 | 17 |
| t3.large | 3 | 12 | 35 |
| m5.large | 3 | 10 | 29 |
| m5.xlarge | 4 | 15 | 58 |
| m5.4xlarge | 8 | 30 | 234 |

Formula: `(Max ENIs × (Max IPs per ENI - 1)) + 2`

```bash
# Check the max Pods for your node type
kubectl describe node ip-10-0-1-100.ec2.internal | grep "pods"
# Allocatable:
#   pods: 35
```

When a node hits its Pod limit, new Pods stay `Pending` even if the node has free CPU and memory. This is a common scaling gotcha on EKS — you need more nodes, not bigger nodes.

> 💡 **TIP:** For clusters that need to run many small Pods, consider using **prefix delegation** with the VPC CNI — each secondary IP can be a /28 prefix, allowing up to 16 IPs per prefix and dramatically increasing Pod density.

---

## Verifying Pod-to-Pod Connectivity

Test Pod-to-Pod communication directly:

```bash
# Get IP addresses of two Pods in different nodes
kubectl get pods -o wide -n production

# NAME              READY   STATUS    IP           NODE
# web-pod-abc123    1/1     Running   10.0.1.45    ip-10-0-1-100.ec2.internal
# api-pod-def456    1/1     Running   10.0.2.67    ip-10-0-2-200.ec2.internal

# Test connectivity from web-pod to api-pod
kubectl exec -it web-pod-abc123 -n production -- \
  sh -c "wget -T 5 -O- http://10.0.2.67:8080/healthz"

# Test with ping (if the container has ping)
kubectl exec -it web-pod-abc123 -n production -- ping 10.0.2.67

# Run a temporary test Pod with networking tools
kubectl run nettest --image=busybox --rm -it --restart=Never -- sh

# Inside the test Pod:
# wget -T 5 http://10.0.2.67:8080
# ping 10.0.1.45
# nslookup kubernetes.default
```

---

## Common Pod Connectivity Issues

### Issue 1: NetworkPolicy Blocking Traffic

If you applied a default-deny NetworkPolicy and forgot to allow specific traffic, Pods cannot communicate even though the network route exists.

```bash
# Test if NetworkPolicy is the issue — run connectivity test from an allowed namespace
kubectl exec -it test-pod -n production -- wget -T 5 http://10.0.2.67:8080

# Timeout? Check NetworkPolicies
kubectl get networkpolicy -n production

# Describe the relevant policy
kubectl describe networkpolicy default-deny-all -n production
```

### Issue 2: Security Group Rules

On EKS, worker nodes have AWS Security Groups. If your Security Group blocks traffic on the application port between nodes, Pod-to-Pod communication fails at the VPC level even though Kubernetes thinks it should work.

```bash
# Check worker node Security Group rules
aws ec2 describe-security-groups \
  --group-ids sg-abc123 \
  --query 'SecurityGroups[].IpPermissions[]'

# The worker node Security Group must allow all traffic from itself (the cluster SG)
# Look for a rule like:
# Protocol: All, Source: sg-abc123 (the same security group)
```

### Issue 3: Pods on Different Subnets

If your EKS nodes span multiple subnets and VPC routing is not configured correctly between subnets, cross-subnet Pod traffic may fail.

```bash
# Check if the VPC route table covers all your subnets
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-abc123" \
  --query 'RouteTables[].Routes[]'
```

---

## Watching Pod Network Traffic

For debugging, you can use `tcpdump` inside a Pod or on the node:

```bash
# Run tcpdump on the node to see Pod traffic
# SSH into the node via AWS Systems Manager
aws ssm start-session --target i-0abc123def456789

# Find the veth interface for a specific Pod
# First get the Pod's network namespace
kubectl exec web-pod-abc123 -- cat /sys/class/net/eth0/ifindex
# Output: 12

# Find the veth pair with that index on the node
ip link | grep "^12:"
# 12: eth0@if13: ...

# Capture traffic on that interface
tcpdump -i veth-abc123 -n
```

---

## Summary

- Same-node Pod communication uses virtual ethernet pairs (veth) and a Linux bridge — pure Layer 2 switching, very fast
- Cross-node Pod communication on EKS uses the VPC CNI — each Pod gets a real VPC secondary IP on the node's ENI, no encapsulation or tunneling needed
- The VPC CNI (`aws-node` DaemonSet) pre-allocates IP pools on worker nodes and assigns them to new Pods
- Each EC2 instance type has a hard limit on Pod density based on ENI count and secondary IPs per ENI — choose instance types appropriately for your Pod count
- Test Pod-to-Pod connectivity with `kubectl exec -- wget` or a temporary `busybox` test Pod
- Common connectivity failures: NetworkPolicy blocking traffic, Security Group rules too restrictive, or missing VPC route table entries

In the next file, `03-cluster-networking-model.md`, you will learn the complete Kubernetes network model in depth — how iptables rules implement Service routing, how the Linux kernel handles traffic, and how to read and interpret routing tables.
