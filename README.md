<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://cdn.simpleicons.org/kubernetes/white" />
    <img alt="Kubernetes Logo" src="https://cdn.simpleicons.org/kubernetes/black" width="60" />
  </picture>
</p>

<h1 align="center">Awesome Kubernetes</h1>

<p align="center">
  <a href="https://awesome.re"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=awesomelists" /><img alt="awesome" src="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=awesomelists" /></picture></a>
  <a href="https://kubernetes.io/"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/Kubernetes-1.28+.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=kubernetes" /><img alt="Kubernetes" src="https://shieldcn.dev/badge/Kubernetes-1.28+.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=kubernetes" /></picture></a>
  <a href="LICENSE"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=dark&amp;font=geist" /><img alt="license" src="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=light&amp;font=geist" /></picture></a>
   <a href="https://github.com/bhagirath00/awesome-kubernetes"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/views/repo/bhagirath00/awesome-kubernetes.svg?variant=outline&amp;mode=dark&amp;font=geist" /><img alt="views" src="https://shieldcn.dev/views/repo/bhagirath00/awesome-kubernetes.svg?variant=outline&amp;mode=light&amp;font=geist" /></picture></a>
</p>

<p align="center">
  <em>A curated, professional-grade guide to mastering Kubernetes orchestration, from container primitives to production multi-cluster fleets.</em>
</p>

---

## Table of Contents

- [Introduction](#introduction)
- [Why Kubernetes for DevOps?](#why-kubernetes-for-devops)
- [Learning Modules](#learning-modules)
  - [Phase 1: Foundations & Core Workloads](#phase-1-foundations--core-workloads)
  - [Phase 2: Cluster Administration & Observability](#phase-2-cluster-administration--observability)
  - [Phase 3: Advanced Networking & Service Mesh](#phase-3-advanced-networking--service-mesh)
  - [Phase 4: Production Hardening, Security & GitOps](#phase-4-production-hardening-security--gitops)
- [Resources](#resources)
- [Contributing](#contributing)
- [Security](#security)
- [License](#license)

---

## Introduction

Welcome to **Awesome Kubernetes**! This repository serves as an exhaustive, hands-on architectural roadmap and reference manual designed for site reliability engineers (SREs), DevOps practitioners, and platform engineers.

> **"Kubernetes is a platform for building platforms. It's a better place to start; that's all."** — Kelsey Hightower

---

## Why Kubernetes for DevOps?

Kubernetes is the cloud-native operating system for modern infrastructure:

| Capability | Importance in DevOps |
| :--- | :--- |
| **Self-Healing Automation** | Automatic container restarts, liveness/readiness probes, and node failure eviction |
| **Declarative Elasticity** | Dynamic pod autoscaling (HPA/VPA) and just-in-time cloud compute provisioning (Karpenter) |
| **Decoupled Architecture** | Infrastructure-as-Code friendly; seamless workload portability across AWS, Azure, GCP, and bare-metal |
| **Zero-Downtime Rollouts** | Native rolling updates, canary deployments, and progressive delivery integration |

---

## Learning Modules

Organized into 20 progressive modules covering 62 chapters and 257 production-grade guides.

### Phase 1: Foundations & Core Workloads

| Module | Description |
| :--- | :--- |
| [**Module 01 - Introduction**](./Module%2001%20-%20Introduction/) | Architecture, Control Plane vs Worker Nodes, OS-specific Setup (Windows/macOS/Linux), and first deployment |
| [**Module 02 - Core Concepts**](./Module%2002%20-%20Core%20Concepts/) | Pods, Multi-Container Pods, ReplicaSets, Deployments, and Namespaces |
| [**Module 03 - Networking**](./Module%2003%20-%20Networking/) | Services (ClusterIP, NodePort, LoadBalancer), Ingress, and CoreDNS resolution |
| [**Module 04 - Configuration**](./Module%2004%20-%20Configuration/) | ConfigMaps, Kubernetes Secrets, and Environment Variables |
| [**Module 05 - Storage**](./Module%2005%20-%20Storage/) | Ephemeral Volumes, PersistentVolumes (PVs), PVCs, and StorageClasses |
| [**Module 06 - Workloads**](./Module%2006%20-%20Workloads/) | StatefulSets, DaemonSets, Jobs, and CronJobs |

### Phase 2: Cluster Administration & Observability

| Module | Description |
| :--- | :--- |
| [**Module 07 - Security**](./Module%2007%20-%20Security/) | RBAC Roles & RoleBindings, ServiceAccounts, and Pod Security Contexts |
| [**Module 08 - Scheduling and Scaling**](./Module%2008%20-%20Scheduling%20and%20Scaling/) | Node Affinity/Anti-Affinity, Taints & Tolerations, HPA, VPA, and Karpenter |
| [**Module 09 - Observability**](./Module%2009%20-%20Observability/) | Liveness & Readiness Probes, EFK/Loki Logging, and Prometheus & Grafana Monitoring |
| [**Module 10 - Package Management**](./Module%2010%20-%20Package%20Management/) | Helm Architecture, Chart Creation, Templates, and Release Lifecycle Management |
| [**Module 11 - Upgrades and Maintenance**](./Module%2011%20-%20Upgrades%20and%20Maintenance/) | etcd Snapshot & Restore, Worker Node Draining, and Zero-Downtime Cluster Upgrades |
| [**Module 12 - Advanced Storage**](./Module%2012%20-%20Advanced%20Storage/) | CSI Volume Plugins, Dynamic Volume Expansion, and VolumeSnapshots |

### Phase 3: Advanced Networking & Service Mesh

| Module | Description |
| :--- | :--- |
| [**Module 13 - Network Policies**](./Module%2013%20-%20Network%20Policies/) | Zero-Trust Network Isolation, Calico Global NetworkPolicies, and Egress Filtering |
| [**Module 14 - TLS and Certificate Management**](./Module%2014%20-%20TLS%20and%20Certificate%20Management/) | Kubernetes CSR API, cert-manager Automation, and Let's Encrypt Integration |
| [**Module 15 - Ingress Controllers**](./Module%2015%20-%20Ingress%20Controllers/) | NGINX Ingress Controller, Traefik, and Kubernetes Gateway API (HTTPRoute) |
| [**Module 16 - Service Mesh**](./Module%2016%20-%20Service%20Mesh/) | Istio Service Mesh, Envoy Sidecars, Strict mTLS, and Canary Traffic Shifting |

### Phase 4: Production Hardening, Security & GitOps

| Module | Description |
| :--- | :--- |
| [**Module 17 - Admission Controllers**](./Module%2017%20-%20Admission%20Controllers/) | Mutating & Validating Webhooks, Kyverno Policy Engine, and OPA Gatekeeper |
| [**Module 18 - Hardening and CIS Benchmarks**](./Module%2018%20-%20Hardening%20and%20CIS%20Benchmarks/) | CIS Kubernetes Benchmarks, kube-bench Remediation, and Falco Runtime Threat Detection |
| [**Module 19 - Secrets Management**](./Module%2019%20-%20Secrets%20Management/) | Encryption at Rest (KMS), HashiCorp Vault Injector, Sealed Secrets, and Secrets Store CSI |
| [**Module 20 - GitOps and Advanced Ops**](./Module%2020%20-%20GitOps%20and%20Advanced%20Ops/) | GitOps with ArgoCD & Flux CD, Multi-Cluster Fleets, Velero Backup/Restore, FinOps, and Incident Triage |

---

## Resources

### Official Documentation

| Resource | Link |
| :--- | :--- |
| Kubernetes Documentation | [kubernetes.io/docs](https://kubernetes.io/docs/home/) |
| kubectl Cheat Sheet | [kubernetes.io/docs/reference/kubectl/cheatsheet/](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) |
| Helm Documentation | [helm.sh/docs](https://helm.sh/docs/) |
| CNCF Cloud Native Landscape | [landscape.cncf.io](https://landscape.cncf.io/) |

### Recommended Tools

- **Kind**: Run local multi-node Kubernetes clusters using Docker containers.
- **Minikube**: Local single-node development cluster with built-in addons.
- **k9s**: Terminal UI for fast observation and management of Kubernetes clusters.
- **Kubeval / Pluto**: Manifest linting and deprecated API version discovery.
- **Velero**: Cloud-native disaster recovery, backup, and namespace migration.

---

## Contributing

Contributions are welcome! Please read our [Contributing Guidelines](CONTRIBUTING.md) before submitting a Pull Request.

---

## Security

Please review our [Security Policy](SECURITY.md) for vulnerability reporting guidelines, attribution details, and supported Kubernetes versions.

---

## License

This educational content is open source and available under the **MIT License**. See [LICENSE](LICENSE) for details.

<p align="center">
  <strong>Happy Orchestrating!</strong>
</p>
