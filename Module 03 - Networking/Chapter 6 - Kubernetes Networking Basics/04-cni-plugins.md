# CNI Plugins

## Introduction

The **CNI (Container Network Interface)** is a standard that defines how network plugins integrate with Kubernetes. When Kubernetes creates a Pod, it calls the CNI plugin to set up that Pod's network — assigning an IP, creating interfaces, and connecting the Pod to the cluster network. Without a CNI plugin, Pods cannot communicate at all.

This file covers what CNI is, how it works, and the major CNI plugins you will encounter — with a focus on the AWS VPC CNI used by EKS.

---

## What CNI Is

CNI is a specification. It defines a simple interface between Kubernetes (the container runtime, specifically the kubelet) and a network plugin:

```
kubelet (creates Pod) → calls CNI binary → CNI sets up Pod networking
```

The CNI "call" is actually a shell execution. When a Pod needs networking:
1. kubelet executes the CNI binary at `/opt/cni/bin/<plugin>`
2. kubelet passes Pod information via environment variables and stdin JSON
3. The CNI binary creates network interfaces, assigns IPs, sets routes
4. CNI returns the assigned IP address to kubelet via stdout JSON

```bash
# CNI binaries live on each worker node
ls /opt/cni/bin/

# Output (on EKS):
# aws-cni    bandwidth  bridge  dhcp  firewall  flannel
# host-device  host-local  ipvlan  loopback  macvlan
# portmap  ptp  sbr  static  tuning  vlan  vpc-branch-eni

# CNI configuration lives here
ls /etc/cni/net.d/

# Output:
# 10-aws.conflist   ← AWS VPC CNI configuration
```

---

## The AWS VPC CNI Plugin — How EKS Handles Networking

On EKS, the AWS VPC CNI plugin (`amazon-vpc-cni-k8s`) is installed by default. It gives each Pod a real VPC IP address.

### How It Works

The VPC CNI runs as a DaemonSet (`aws-node`) on every worker node. It does three things:

**1. ENI Management** — attaches additional ENIs (Elastic Network Interfaces) to the EC2 node instance to get more secondary IP addresses:

```bash
# See the aws-node DaemonSet
kubectl get daemonset aws-node -n kube-system

# See CNI logs on a node
kubectl logs -n kube-system -l k8s-app=aws-node --tail=30
```

**2. IP Pool Management** — pre-allocates secondary IPs on the ENIs so new Pods get an IP immediately without waiting for an AWS API call:

```bash
# Configure the IP pool size
kubectl set env daemonset aws-node \
  WARM_IP_TARGET=5 \              # keep 5 IPs always available
  MINIMUM_IP_TARGET=10 \          # allocate at least 10 IPs total
  -n kube-system
```

**3. Route Setup** — when a Pod is scheduled, the VPC CNI:
- Assigns a secondary IP to the Pod
- Creates a veth pair (one end in Pod network namespace, one on node)
- Adds a host route on the node: `10.0.1.45 dev veth-abc`

```bash
# See routes added by VPC CNI on a worker node
ip route show

# Output:
# default via 10.0.1.1 dev eth0
# 10.0.1.45 dev eni-12345abc scope link   ← route to Pod A
# 10.0.1.46 dev eni-67890def scope link   ← route to Pod B
```

### VPC CNI Configuration

The VPC CNI behavior is configured via environment variables on the `aws-node` DaemonSet:

```bash
kubectl describe daemonset aws-node -n kube-system | grep -A 50 "Environment:"
```

| Variable | Default | Purpose |
|----------|---------|---------|
| `WARM_ENI_TARGET` | 1 | Keep N extra ENIs warmed up |
| `WARM_IP_TARGET` | 0 | Keep N extra IPs available |
| `MINIMUM_IP_TARGET` | 0 | Minimum IPs to maintain |
| `ENABLE_POD_ENI` | false | Enable dedicated ENIs per Pod (security groups for Pods) |
| `ENABLE_PREFIX_DELEGATION` | false | Use /28 prefixes — increases Pod density |

### Prefix Delegation — More Pods Per Node

By default, each secondary IP supports one Pod. With prefix delegation, each secondary IP is a /28 prefix (16 addresses), dramatically increasing Pod density:

```bash
# Enable prefix delegation
kubectl set env daemonset aws-node \
  ENABLE_PREFIX_DELEGATION=true \
  -n kube-system

# Now each prefix slot holds 16 Pod IPs instead of 1
# A t3.large (3 ENIs × 12 slots) goes from 35 to 110+ Pods
```

---

## Other CNI Plugins

While EKS uses the VPC CNI by default, other environments use different CNI plugins. Understanding them helps when you work with on-premise clusters or non-EKS Kubernetes:

### Flannel — Simplest, Overlay Network

Flannel uses a **VXLAN overlay network**. Pod traffic is encapsulated in UDP packets and tunneled between nodes:

```
Pod A (10.244.1.5)          Pod B (10.244.2.7)
     │                            │
     ▼                            ▼
VXLAN tunnel header           VXLAN tunnel header
     │                            │
Node A eth0 (192.168.1.10) → Node B eth0 (192.168.1.20)
```

Pros: Simple, works anywhere, no cloud provider dependency.
Cons: Overhead from encapsulation, lower performance than VPC-native networking.

```bash
# Flannel DaemonSet (self-managed clusters)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Calico — Network Policy Focus

Calico supports both overlay (VXLAN/IPIP) and native routing modes. Its key strength is **network policy enforcement**:

```bash
# Calico installation
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

Calico implements Kubernetes NetworkPolicies and also supports its own more advanced `GlobalNetworkPolicy` CRD — which can enforce cluster-wide policies that apply across all namespaces.

On EKS, you can run Calico alongside the VPC CNI — using VPC CNI for IP assignment and Calico only for NetworkPolicy enforcement:

```bash
# Install Calico for NetworkPolicy only (not for IP management)
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/calico-operator.yaml
```

### Cilium — eBPF-Based, Modern Choice

Cilium uses **eBPF (extended Berkeley Packet Filter)** instead of iptables. eBPF programs run directly in the Linux kernel, bypassing much of the networking stack for dramatically better performance:

```
Traditional (iptables):
  Packet → netfilter → iptables chains → routing
  
Cilium (eBPF):
  Packet → eBPF program (XDP/TC hook) → direct delivery
  Much faster — bypasses full netfilter evaluation
```

Cilium also provides:
- Layer 7 network policies (HTTP-aware — block specific HTTP paths or methods)
- Built-in Hubble for network flow observability
- Replaces kube-proxy entirely

On EKS, you can replace the VPC CNI with Cilium (using Cilium's ENI mode to still get VPC-native IPs).

---

## Comparing CNI Plugins for EKS

| CNI Plugin | IP Model | NetworkPolicy | Performance | Complexity |
|-----------|----------|--------------|-------------|-----------|
| **AWS VPC CNI** (default) | Native VPC IPs | Via Calico add-on | High (no encapsulation) | Low |
| **Calico + VPC CNI** | Native VPC IPs | Full Calico policies | High | Medium |
| **Cilium (ENI mode)** | Native VPC IPs | L3/L4/L7 | Highest (eBPF) | Medium-High |
| **Flannel** | Overlay (VXLAN) | No (use Calico) | Lower | Very Low |

For most EKS use cases: **AWS VPC CNI + Calico for NetworkPolicy** is the recommended combination. It gives you native VPC performance and full NetworkPolicy support.

---

## Verifying Your CNI Plugin

```bash
# What CNI plugin is running?
kubectl get pods -n kube-system | grep -E "aws-node|calico|cilium|flannel"

# Check CNI configuration on a node
cat /etc/cni/net.d/10-aws.conflist

# Check CNI binary version
/opt/cni/bin/aws-cni --version

# Verify Pod IP assignment is working
kubectl run cni-test --image=busybox --rm -it --restart=Never -- \
  sh -c "ip addr show eth0 && ip route"

# Output should show:
# eth0 with a VPC IP (e.g., 10.0.1.45/32)
# default route via the node gateway
```

---

## Summary

- CNI (Container Network Interface) is a standard that defines how network plugins integrate with Kubernetes — kubelet calls the CNI binary to set up Pod networking
- The AWS VPC CNI (`aws-node` DaemonSet) is the default on EKS — it assigns real VPC secondary IPs to Pods, enabling native VPC routing without encapsulation
- The VPC CNI manages ENI attachment, pre-allocates IP pools, and sets up host routes for each Pod
- Enable prefix delegation (`ENABLE_PREFIX_DELEGATION=true`) to dramatically increase Pod density per node
- Flannel uses VXLAN overlay networks — simple but adds encapsulation overhead; no NetworkPolicy support
- Calico adds robust NetworkPolicy enforcement — can run alongside VPC CNI on EKS for the best of both
- Cilium uses eBPF to replace both CNI and kube-proxy — highest performance, L7-aware policies, built-in observability

In Chapter 7, `01-what-is-a-service.md`, you will learn about Kubernetes Services in full depth — the resource that gives Pods a stable network identity and distributes traffic across them.
