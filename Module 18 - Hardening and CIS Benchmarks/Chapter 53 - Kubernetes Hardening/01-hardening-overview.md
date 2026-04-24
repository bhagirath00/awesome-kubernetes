# Kubernetes Hardening Overview & The 4Cs Security Model

## 1. Architectural Concept & Deep Dive

Securing a Kubernetes cluster requires recognizing that Kubernetes is not a single monolith, but a distributed orchestration system spanning physical infrastructure, kernel virtualization, API authorization, and container runtimes.

The Cloud Native Computing Foundation (CNCF) and the National Security Agency (NSA / CISA) define the standard defense-in-depth model known as the **4Cs of Cloud Native Security**:
1. **Cloud / Infrastructure**: Physical data centers, hypervisors, cloud IAM, VPC networks, and firewall rules.
2. **Cluster**: Kubernetes Control Plane (API server, etcd, scheduler, controller-manager) and Worker Node daemons (kubelet, kube-proxy, CRI).
3. **Container**: Container image registries, base image CVEs, non-root users, Linux capabilities, and runtime sandboxes.
4. **Code**: Application source code, third-party libraries, TLS endpoints, and business logic.

```
+─────────────────────────────────────────────────────────────────────────+
|                    THE 4Cs DEFENSE-IN-DEPTH MODEL                       |
|                                                                         |
|  +───────────────────────────────────────────────────────────────────+  |
|  |                           1. CLOUD                                |  |
|  |  AWS / GCP / Bare Metal: VPC, Security Groups, IAM, KMS           |  |
|  |  +─────────────────────────────────────────────────────────────+  |  |
|  |  |                        2. CLUSTER                           |  |  |
|  |  |  K8s API, etcd, mTLS, RBAC, NetworkPolicies, Admission Ctrl|  |  |
|  |  |  +───────────────────────────────────────────────────────+  |  |  |
|  |  |  |                     3. CONTAINER                      |  |  |  |
|  |  |  |  Non-root UID, Read-only Rootfs, Seccomp, Capabilities|  |  |  |
|  |  |  |  +─────────────────────────────────────────────────+  |  |  |  |
|  |  |  |  |                   4. CODE                       |  |  |  |  |
|  |  |  |  |  Static Analysis (SAST), Dependency CVEs, TLS    |  |  |  |  |
|  |  |  |  +─────────────────────────────────────────────────+  |  |  |  |
|  |  |  +───────────────────────────────────────────────────────+  |  |  |
|  |  +─────────────────────────────────────────────────────────────+  |  |
|  +───────────────────────────────────────────────────────────────────+  |
+─────────────────────────────────────────────────────────────────────────+
```

> **The Golden Rule of Cloud Native Security**:
> *A breach at an outer layer compromises all inner layers.* If the underlying Cloud IAM or physical host is compromised, no amount of in-cluster RBAC or container hardening can protect the workload.

---

## 2. Threat Modeling: MITRE ATT&CK for Kubernetes

The MITRE Corporation categorizes adversary techniques targeting Kubernetes into a standardized threat matrix:

| Attack Vector | Tactic Category | Real-World Threat Example | Primary Hardening Defense |
| :--- | :--- | :--- | :--- |
| **Initial Access** | Supply Chain / API Access | Compromised kubeconfig, exposed Dashboard, untrusted container image. | OIDC Authentication, private API endpoints, image signature verification. |
| **Execution** | Container Execution | Executing shells inside pods (`kubectl exec`), running malicious entrypoints. | Restrict RBAC `exec` verbs, enforce Falco runtime detection. |
| **Privilege Escalation**| Host Breakout | Exploiting `privileged: true`, mounting `/var/run/docker.sock` or host root `/`. | Enforce Pod Security Standards (`Restricted`), drop `CAP_SYS_ADMIN`. |
| **Defense Evasion** | Disabling Auditing | Deleting audit logs, killing Falco/Wazuh daemonsets, tampering with kube-bench. | Forward logs off-cluster immediately, protect daemonsets with admission webhooks. |
| **Credential Access** | Token Stealing | Reading ServiceAccount tokens mounted at `/var/run/secrets/kubernetes.io/serviceaccount`. | Set `automountServiceAccountToken: false` by default, use Short-Lived Bound Service Account Tokens. |
| **Lateral Movement** | Network Sniffing | Traversing flat pod network to reach cloud metadata service (`169.254.169.254`). | Default-deny NetworkPolicies, IMDSv2 enforcement on AWS. |

---

## 3. Control Plane vs Data Plane Threat Vectors

```
           CONTROL PLANE ATTACK VECTORS                 DATA PLANE ATTACK VECTORS
          (The Brain: Master Nodes)                    (The Workers: Pod Workloads)

          • Unauthenticated API Access (Port 6443)      • Malicious Container Image Pulled
          • Stolen Admin Kubeconfig Credentials         • Container Breakout to Host Kernel
          • Direct etcd Compromise (Port 2379)          • Unrestricted Pod-to-Pod Lateral Scan
          • Insecure Controller-Manager Flags           • SSRF against Cloud Metadata Service
```

### 1. Control Plane Security Priorities:
- Close public API server endpoints; use VPC private endpoints or VPN-gated ingress.
- Enforce strict mutual TLS (mTLS) with internal PKI between API server, etcd, and kubelets.
- Encrypt etcd storage at rest using Cloud KMS (AWS KMS, Google Cloud KMS, HashiCorp Vault).

### 2. Data Plane Security Priorities:
- Implement Pod Security Standards (`baseline` or `restricted`) across all application namespaces.
- Enforce default-deny egress and ingress `NetworkPolicy` rules.
- Eliminate host network (`hostNetwork: true`), host PID, and host IPC sharing.

---

## 4. The NSA / CISA Kubernetes Hardening Guidance

The United States NSA and CISA published an authoritative Kubernetes Hardening Guide. Its core technical mandates include:

1. **Pod Security**:
   - Run containers as non-root users (`runAsNonRoot: true`).
   - Use immutable read-only root filesystems (`readOnlyRootFilesystem: true`).
   - Drop all unnecessary Linux capabilities (`capabilities.drop: ["ALL"]`).
2. **Network Isolation**:
   - Never rely on default open networking; explicitly declare ingress and egress firewall rules for every microservice.
   - Restrict access to the cloud metadata service (`169.254.169.254`) to prevent AWS IAM role theft.
3. **Authentication & Authorization**:
   - Disable anonymous authentication on the API server and Kubelet (`--anonymous-auth=false`).
   - Enforce least-privilege Role-Based Access Control (RBAC); audit `cluster-admin` bindings.
4. **Log Auditing & Continuous Detection**:
   - Enable comprehensive Kubernetes API audit logging with dynamic backends.
   - Deploy eBPF-based runtime threat detection engines (Falco).

---

## 5. Security Posture Assessment Workflow

```bash
# 1. Inspect cluster-admin rolebindings across all subjects
kubectl get clusterrolebindings -o json | jq -r '
  .items[] | select(.roleRef.name=="cluster-admin") |
  "Binding: \(.metadata.name) -> Subjects: \([.subjects[]? | "\(.kind):\(.name)"] | join(", "))"
'

# 2. Identify all pods running as root or with host namespaces enabled
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] | select(.spec.hostNetwork==true or .spec.hostPID==true) |
  "\(.metadata.namespace)/\(.metadata.name) [HostNetwork: \(.spec.hostNetwork), HostPID: \(.spec.hostPID)]"
'
```

---

## 6. Key Takeaways
- The 4Cs model (Cloud, Cluster, Container, Code) mandates layered security; failure in an outer layer compromises inner layers.
- The MITRE ATT&CK matrix for Kubernetes maps out real-world adversary tactics from initial access to lateral movement.
- Modern Kubernetes hardening focuses on closing control plane exposure (etcd encryption, private endpoints) and containing data plane blast radiuses (Pod Security Standards, NetworkPolicies).
- Apply the NSA/CISA guidelines to systematically achieve compliance across production clusters.
