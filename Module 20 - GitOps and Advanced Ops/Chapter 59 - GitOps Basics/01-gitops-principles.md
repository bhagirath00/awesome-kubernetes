# GitOps Principles & Architecture

## 1. Architectural Concept & Deep Dive

Coined by Alexis Richardson (CEO of Weaveworks) in 2017 and formalized by the CNCF **OpenGitOps Working Group**, **GitOps** is an operating model for cloud-native applications and infrastructure that uses **Git as the single source of truth** for declarative state.

In traditional DevOps CI/CD pipelines, automated build agents execute commands (`kubectl apply`, `helm upgrade`, `terraform apply`) pushing changes into production clusters. This traditional push model suffers from configuration drift, credential sprawl, and un-audited manual modifications (`kubectl edit` in production).

GitOps reverses this model: **an in-cluster software agent continuously pulls the desired state from Git and reconciles differences in real time.**

```
+─────────────────────────────────────────────────────────────────────────+
|                           THE GITOPS RECONCILIATION LOOP                |
|                                                                         |
|  [ Developer / CI System ]                                              |
|         │                                                               |
|         ├── git commit & git push (Pull Request Reviewed & Merged)      |
|         ▼                                                               |
|  [ Git Repository (Desired State: Version 2.4) ]                        |
|         │                                                               |
|         │ (Continuous Pull / Webhook Notification)                      |
|         ▼                                                               |
|  [ In-Cluster GitOps Operator (ArgoCD / FluxCD) ]                       |
|         │                                                               |
|         ├── 1. Observes Desired State from Git: Replicas=5, Image=v2.4  |
|         ├── 2. Observes Actual Live State from Cluster: Replicas=3      |
|         ├── 3. Detects DRIFT!                                           |
|         ▼                                                               |
|  [ Auto-Self-Healing / Reconciliation Pass ]                            |
|         │                                                               |
|         ▼                                                               |
|  [ Live Kubernetes Cluster (Actual State Reconciled to Match Git) ]     |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. The Four Principles of OpenGitOps (CNCF Standard)

To qualify as true GitOps according to the CNCF OpenGitOps standard, a system must satisfy four non-negotiable principles:

| Principle | Technical Definition | Production Value |
| :--- | :--- | :--- |
| **1. Declarative** | The entire desired state of the system must be described declaratively (e.g. Kubernetes YAML, Kustomize, Helm). | Eliminates procedural scripts; manifests express *what* the system should be, not *how* to build it. |
| **2. Versioned and Immutable** | Desired state is stored in an immutable, version-controlled store (Git) that preserves complete version history. | Every single deployment is a Git commit; instant zero-overhead rollback via `git revert`. |
| **3. Pulled Automatically** | Software agents automatically pull the desired state from Git on a continuous cadence. | Cluster credentials never leave the cluster; external CI servers have zero direct access to production APIs. |
| **4. Continuously Reconciled** | Software agents continuously observe actual cluster state and automatically reconcile drift back to desired state. | If an engineer manually deletes a deployment with `kubectl`, the GitOps operator immediately recreates it. |

---

## 3. The Core Problem Solved: Configuration Drift & Self-Healing

### What is Configuration Drift?
An on-call engineer experiences a midnight outage and runs `kubectl edit deployment/payment-api` to manually increase replicas or patch an environment variable. 
- In traditional CI/CD, this change is unrecorded. The next scheduled build overwrites the fix, triggering a recurring outage.
- In GitOps, **manual out-of-band changes are strictly prohibited or instantly overwritten**.

### Automated Self-Healing in Action
```
1. Operator runs: kubectl delete service payment-service
2. Cluster state: payment-service is deleted (Actual State != Desired State)
3. GitOps Operator (ArgoCD): Detects OutOfSync condition
4. Operator applies auto-sync: Automatically re-applies payment-service manifest from Git within 3 seconds!
```

---

## 4. GitOps Repository Strategies: Monorepo vs Polyrepo

| Strategy | Structure | Advantages | Challenges |
| :--- | :--- | :--- | :--- |
| **Monorepo** | Single repository containing all cluster configs, microservices, and environment folders (`/dev`, `/staging`, `/prod`). | Central visibility, easy cross-service PRs, atomic infrastructure updates. | Requires granular CODEOWNERS and branch protection to prevent cross-team collisions. |
| **Polyrepo** | Separate repository per application team (`payment-manifests`, `auth-manifests`) + dedicated cluster repo. | Strict team boundary isolation, independent RBAC, reduced merge conflicts. | Harder to view full infrastructure state at a glance; cross-team changes require multiple PRs. |

### Recommended Directory Structure (Kustomize Overlays)
```
infrastructure-gitops/
├── apps/
│   ├── base/               # Universal baseline manifests
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── overlays/
│       ├── staging/        # Environment-specific patches
│       │   ├── kustomization.yaml
│       │   └── replicas-patch.yaml
│       └── production/
│           ├── kustomization.yaml
│           └── resources-patch.yaml
└── clusters/
    ├── staging-us-east/
    └── prod-us-west/
```

---

## 5. Security Posture of GitOps

1. **Firewall Inversion (Pull over Push)**:
   - In traditional CI/CD (Jenkins, GitLab CI, GitHub Actions), the CI server requires full `cluster-admin` credentials and an open firewall port into your Kubernetes API.
   - In GitOps, the in-cluster operator initiates outbound read-only HTTPS connections to GitHub. **The cluster API can be 100% private with zero inbound ports open!**
2. **Cryptographic Signatures & Approvals**:
   - All production deployments must pass through Git Pull Requests.
   - Branch protection rules enforce required reviews, signed GPG commits, and successful CI testing before merges.
   - The Git commit history forms an immutable, tamper-proof SOC 2 audit log.

---

## 6. Key Takeaways
- GitOps establishes Git as the single source of truth for both infrastructure and applications.
- Built on four CNCF principles: Declarative, Versioned, Pulled Automatically, and Continuously Reconciled.
- In-cluster operators continuously monitor and eliminate configuration drift via automated self-healing.
- Inverts traditional security by pulling changes outbound, allowing Kubernetes API servers to remain entirely private.
