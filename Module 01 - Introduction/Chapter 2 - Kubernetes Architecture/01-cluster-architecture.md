# Kubernetes Cluster Architecture

## Introduction

To use Kubernetes effectively, you need to understand what is actually running when you "have a Kubernetes cluster." A cluster is not a single machine or a single process — it is a collection of machines, each running specific components that work together to schedule, run, and manage your containers.

This file gives you the complete architectural picture of a Kubernetes cluster. You will understand every layer — from the machines themselves down to how components communicate — before you go deep on individual components in the files that follow.

---

## The Two-Layer Architecture

Every Kubernetes cluster is divided into exactly two layers:

```
┌─────────────────────────────────────────────────────────────────┐
│                      KUBERNETES CLUSTER                         │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    CONTROL PLANE                          │  │
│  │                                                           │  │
│  │   API Server  │  Scheduler  │  Controller Manager  │ etcd│  │
│  │                                                           │  │
│  │   The brain — makes decisions, stores state               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐    │
│  │ WORKER NODE 1│   │ WORKER NODE 2│   │  WORKER NODE 3   │    │
│  │              │   │              │   │                  │    │
│  │ kubelet      │   │ kubelet      │   │ kubelet          │    │
│  │ kube-proxy   │   │ kube-proxy   │   │ kube-proxy       │    │
│  │ containerd   │   │ containerd   │   │ containerd       │    │
│  │              │   │              │   │                  │    │
│  │ Pod Pod Pod  │   │ Pod Pod Pod  │   │ Pod Pod Pod      │    │
│  │              │   │              │   │                  │    │
│  │  The muscle  │   │  The muscle  │   │  The muscle      │    │
│  └──────────────┘   └──────────────┘   └──────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

**Control Plane** — The brain of the cluster. It makes all the decisions: where to run containers, what the desired state should be, and how to respond when something fails. You interact with the control plane through `kubectl`.

**Worker Nodes** — The muscle of the cluster. They actually run your application containers. Worker nodes do exactly what the control plane tells them to do. Your Pods always run on worker nodes, never on control plane nodes (in a standard setup).

---

## What is a Node

A **node** is any machine — physical server, virtual machine, or cloud instance — that is part of the Kubernetes cluster.

On AWS EKS:
- **Control plane nodes** are EC2 instances managed entirely by AWS. You never see them in your account. AWS handles their availability, patching, and scaling.
- **Worker nodes** are EC2 instances in your AWS account. You control the instance type, the number, and the configuration.

```bash
# List all nodes in your cluster
kubectl get nodes

# Output:
# NAME                         STATUS   ROLES    AGE   VERSION
# ip-10-0-1-100.ec2.internal   Ready    <none>   5d    v1.29.0
# ip-10-0-1-101.ec2.internal   Ready    <none>   5d    v1.29.0
# ip-10-0-2-100.ec2.internal   Ready    <none>   5d    v1.29.0

# Get detailed information about a node
kubectl describe node ip-10-0-1-100.ec2.internal
```

In EKS output, worker nodes show `<none>` for ROLES because EKS manages the control plane separately — you only see worker nodes.

---

## Control Plane Components

The control plane runs four core components. Each has a specific job:

### API Server (`kube-apiserver`)
The front door of Kubernetes. Every single operation goes through the API server — `kubectl` commands, scheduler decisions, controller updates, kubelet reports. It exposes a REST API and enforces authentication, authorization, and validation.

### etcd
The cluster's database. A distributed key-value store that holds the complete state of the cluster — every resource object (Pods, Deployments, Services, ConfigMaps, Secrets). If etcd loses data, the cluster loses its memory of what should exist.

### Scheduler (`kube-scheduler`)
Watches for newly created Pods that have not been assigned to a node yet, and picks the best node for them. It considers CPU/memory availability, node taints, Pod affinity rules, and topology constraints.

### Controller Manager (`kube-controller-manager`)
Runs a set of controllers in a single process. Each controller watches the cluster state and works to make the actual state match the desired state. The Deployment controller ensures the right number of replica Pods exist. The Node controller detects when a node goes offline.

---

## Worker Node Components

Every worker node runs three components:

### kubelet
The node agent. It runs on every worker node and is responsible for starting, stopping, and monitoring containers on that node. The kubelet receives Pod specifications from the API server and tells the container runtime to start the right containers.

### kube-proxy
Handles network rules on each node. It maintains iptables or IPVS rules that allow Pods and Services to communicate across the cluster. When you create a Service, kube-proxy is what makes traffic actually reach the right Pods.

### Container Runtime
The software that actually runs containers. Kubernetes does not run containers directly — it delegates to a container runtime. The current standard is **containerd**. Docker was previously used but was deprecated as a runtime in Kubernetes 1.24.

---

## How These Components Talk to Each Other

All communication in a Kubernetes cluster goes through the API server. Components do not talk directly to each other:

```
kubectl (your terminal)
    │
    ▼  HTTPS REST API
┌────────────────┐
│   API Server   │
└──┬─────────────┘
   │ read/write
   ▼
┌────────────────┐
│      etcd      │  (only API server talks to etcd directly)
└────────────────┘
   ▲                    ▲                    ▲
   │ watches            │ watches            │ watches + reports
   │                    │                    │
Scheduler         Controller Manager      kubelet (on each node)
```

This design is intentional. The API server is the single point of truth. Everything reads from and writes to it. No component reaches into etcd directly (except the API server). This makes the system auditable and secure.

---

## The Desired State vs Actual State Loop

The most important concept in Kubernetes architecture is the **reconciliation loop**. Every controller runs an infinite loop that:

1. Reads the **desired state** from the API server (what you declared in YAML)
2. Reads the **actual state** from the real world (what actually exists)
3. Takes action to make actual match desired
4. Repeats

Example with a Deployment:

```
Desired state (your YAML): replicas: 3
Actual state (running Pods): 2 Pods running

Deployment Controller sees the gap → creates 1 new Pod → schedules it → kubelet starts it
New actual state: 3 Pods running → matches desired → no action needed

One Pod crashes → actual state: 2 Pods again
Deployment Controller sees the gap → creates 1 new Pod
```

This loop runs constantly. It is why Kubernetes is self-healing — it does not rely on you to notice and fix problems manually.

---

## Cluster Sizes — A Realistic Picture

| Cluster Size | Worker Nodes | Pods | Typical Use |
|-------------|-------------|------|------------|
| Small (dev) | 2–3 nodes | < 100 Pods | Development, testing |
| Medium | 5–10 nodes | 100–500 Pods | Startup production |
| Large | 20–50 nodes | 500–2000 Pods | Mid-size company |
| Very Large | 100+ nodes | 5000+ Pods | Enterprise (Kubernetes supports up to 5000 nodes) |

On AWS EKS, you start with a minimum of 2 worker nodes for high availability (so if one node fails, your Pods have somewhere to move to).

---

## High Availability Architecture on AWS EKS

A production EKS cluster uses multiple Availability Zones (AZs) for worker nodes:

```
                AWS Region: us-east-1
                
    ┌───────────────────────────────────────────────┐
    │            EKS Control Plane (AWS Managed)    │
    │  Runs across 3 AZs automatically              │
    └───────────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
  us-east-1a         us-east-1b        us-east-1c
  ┌──────────┐      ┌──────────┐      ┌──────────┐
  │ Worker   │      │ Worker   │      │ Worker   │
  │ Node     │      │ Node     │      │ Node     │
  │ t3.large │      │ t3.large │      │ t3.large │
  └──────────┘      └──────────┘      └──────────┘
```

If `us-east-1a` becomes unavailable, Pods on that node are rescheduled to nodes in `us-east-1b` and `us-east-1c`. Your application stays running.

---

## Setting Up kubectl to Talk to Your EKS Cluster

When you work with EKS, you need to configure `kubectl` to point to your cluster:

```bash
# Update kubeconfig to point to your EKS cluster
aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-production-cluster

# Verify kubectl is connected
kubectl cluster-info

# Output:
# Kubernetes control plane is running at https://ABC123.gr7.us-east-1.eks.amazonaws.com
# CoreDNS is running at https://ABC123.gr7.us-east-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# See the cluster version
kubectl version

# Output:
# Client Version: v1.29.0
# Server Version: v1.29.0
```

The `kubeconfig` file (`~/.kube/config`) stores the cluster endpoint URL, credentials, and context. You can have multiple clusters in kubeconfig and switch between them:

```bash
# List all configured clusters
kubectl config get-contexts

# Switch to a different cluster
kubectl config use-context my-staging-cluster

# See which cluster you are currently talking to
kubectl config current-context
```

---

## Summary

- A Kubernetes cluster has two layers: the **Control Plane** (brain) and **Worker Nodes** (muscle)
- The Control Plane runs four components: API Server, etcd, Scheduler, and Controller Manager
- Worker Nodes run three components: kubelet, kube-proxy, and a container runtime (containerd)
- All communication goes through the API server — components do not talk to each other directly
- Kubernetes runs a constant **reconciliation loop** — comparing desired state (your YAML) with actual state (what is running) and fixing differences automatically
- On AWS EKS, the control plane is fully managed by AWS; you only manage worker nodes
- Production clusters spread worker nodes across multiple Availability Zones for high availability
- Use `aws eks update-kubeconfig` to configure `kubectl` to talk to your EKS cluster

In the next file, `02-control-plane-components.md`, you will go deep into each control plane component — understanding exactly what the API Server, etcd, Scheduler, and Controller Manager do and how they interact.
