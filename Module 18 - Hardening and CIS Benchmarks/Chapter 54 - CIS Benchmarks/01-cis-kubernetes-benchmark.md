# The CIS Kubernetes Benchmark Deep Dive

## 1. Architectural Concept & Deep Dive

The **Center for Internet Security (CIS)** is an internationally recognized, non-profit organization that develops consensus-based best practices for cybersecurity. 

The **CIS Kubernetes Benchmark** is the global gold standard for securing Kubernetes clusters. Maintained by industry practitioners, cloud providers, and security researchers, it provides declarative, prescriptive hardening specifications with precise verification and remediation commands for every component of the architecture.

Major regulatory frameworks—including **PCI-DSS 4.0, SOC 2 Type II, HIPAA, and ISO 27001**—explicitly reference the CIS Kubernetes Benchmark as acceptable proof of container orchestration security controls.

```
+─────────────────────────────────────────────────────────────────────────+
|                    CIS KUBERNETES BENCHMARK HIERARCHY                   |
|                                                                         |
|  Section 1: Control Plane Components                                    |
|  ├── 1.1 Control Plane Node Configuration Files (Permissions & Owner)   |
|  ├── 1.2 API Server (Flags, Authentication, AuthZ, Auditing)            |
|  ├── 1.3 Controller Manager (Flags, Service Account Keys)               |
|  ├── 1.4 Scheduler (Flags, Profiling)                                   |
|  └── 1.5 etcd (Permissions, mTLS, Client Certs)                         |
|                                                                         |
|  Section 2: Control Plane Configuration                                 |
|  └── 2.1 Encryption at Rest (EncryptionConfiguration, KMS)              |
|                                                                         |
|  Section 3: Worker Node Configuration                                   |
|  ├── 3.1 Worker Node Configuration Files (kubelet.conf, pki permissions)|
|  └── 3.2 Kubelet (Anonymous auth, Webhook mode, Port 10255)             |
|                                                                         |
|  Section 4: Policies & Workloads                                        |
|  ├── 4.1 RBAC & Service Accounts                                        |
|  ├── 4.2 Pod Security Standards (Privileged, Root, Capabilities)        |
|  └── 4.3 Network Policies & CNI                                         |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Benchmark Profiles: Level 1 vs Level 2

Every recommendation in the CIS Benchmark is assigned a profile level based on its operational impact:

| Benchmark Profile | Definition | Operational Impact | Recommended Usage |
| :--- | :--- | :--- | :--- |
| **Level 1 (Basic)** | Practical baseline security recommendations. Intended to be applied broadly with minimal risk of breaking workloads. | Zero to Low. Standard production posture. | **Mandatory across all production and staging clusters.** |
| **Level 2 (Hardened)**| Strict defense-in-depth recommendations. Restricts advanced capabilities, disables profiling, enforces mandatory network encryption. | Medium to High. Can inhibit developer productivity or break legacy applications. | Financial services, healthcare, defense, zero-trust environments. |

---

## 3. Scored vs Automated vs Manual Checks

- **Scored / Automated**: The recommendation has a deterministic pass/fail condition that can be evaluated programmatically via CLI or automated scanners (e.g., verifying file permissions on `/etc/kubernetes/manifests/kube-apiserver.yaml` are `0600`).
- **Manual**: The recommendation requires architectural review and human contextual judgment (e.g., verifying that developers are not assigned excessive `cluster-admin` RBAC privileges).

---

## 4. Key Sections & Landmark Recommendations

### Section 1: Master Node & Control Plane
- **1.1.1**: Ensure that the API server pod specification file permissions are set to `600` or more restrictive.
- **1.2.1**: Ensure that the `--anonymous-auth` argument is set to `false`.
- **1.2.18**: Ensure that the `--insecure-bind-address` argument is not set.
- **1.2.22**: Ensure that the `--authorization-mode` argument includes `Node`.
- **1.2.25**: Ensure that the `--enable-admission-plugins` argument includes `NodeRestriction`.
- **1.2.30**: Ensure that the `--audit-log-path` argument is set.

### Section 2: etcd Hardening
- **2.1**: Ensure that the `--cert-file` and `--key-file` arguments are appropriately set.
- **2.2**: Ensure that the `--client-cert-auth` argument is set to `true`.
- **2.4**: Ensure that the `--peer-cert-file` and `--peer-key-file` arguments are appropriately set.

### Section 3: Worker Nodes & Kubelet
- **3.1.1**: Ensure that the `kubelet.conf` file permissions are set to `600`.
- **3.2.1**: Ensure that the `anonymous-auth` argument is set to `false`.
- **3.2.2**: Ensure that the `authorization.mode` argument is set to `Webhook`.
- **3.2.4**: Ensure that the `readOnlyPort` argument is set to `0`.

### Section 4: Policies & Cluster Governance
- **4.1.1**: Ensure that the `cluster-admin` role is only used where strictly necessary.
- **4.2.1**: Minimize the admission of privileged containers (enforce Pod Security Standards).
- **4.3.1**: Ensure that Network Policies are enabled and enforced across all namespaces.

---

## 5. Self-Managed vs Cloud-Managed Benchmarks (EKS, GKE, AKS)

The standard CIS Kubernetes Benchmark assumes you manage the bare-metal or virtual control plane nodes directly (kubeadm / custom installs).

For cloud-managed clusters, the CIS consortium publishes dedicated vendor benchmarks:
- **CIS Amazon EKS Benchmark**: Disregards Section 1 (control plane is managed by AWS under shared responsibility), focusing heavily on AWS IAM OIDC roles, worker node AMIs, security groups, and AWS Secrets Manager.
- **CIS Google GKE Benchmark**: Focuses on Workload Identity, Shielded GKE Nodes, Datapath V2 (Cilium), and Cloud Monitoring audit logging.
- **CIS Microsoft AKS Benchmark**: Focuses on Azure AD integration, Azure CNI, and Azure Key Vault CSI.

---

## 6. Verification & Troubleshooting Runbook

### How Auditors Evaluate Compliance
Auditors run verification scripts against every control plane and worker node:

```bash
# Example Audit for CIS 1.1.1 (API Server Manifest Permissions)
stat -c %a /etc/kubernetes/manifests/kube-apiserver.yaml
# Must return: 600 or 640

# Example Audit for CIS 1.2.1 (Anonymous Auth)
ps -ef | grep kube-apiserver | grep -o -- "--anonymous-auth=[a-zA-Z]*"
# Must return: --anonymous-auth=false
```

---

## 7. Key Takeaways
- The CIS Kubernetes Benchmark is the recognized industry standard for Kubernetes security compliance (PCI-DSS, SOC 2).
- Level 1 provides practical baseline security for all production clusters; Level 2 introduces strict zero-trust controls.
- The benchmark covers 4 core zones: Control Plane, etcd, Worker Nodes, and Cluster Policies.
- Managed clusters (EKS/GKE) follow specialized cloud provider benchmarks that reflect the cloud shared responsibility model.
