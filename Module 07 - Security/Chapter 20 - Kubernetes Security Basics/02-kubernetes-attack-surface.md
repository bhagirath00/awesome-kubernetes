# Kubernetes Attack Surface

## Introduction

An **attack surface** is the sum of all possible entry points where an attacker could gain unauthorized access to your system. Kubernetes has a large attack surface because it is distributed, complex, and runs untrusted workloads (user applications).

Understanding the attack surface is critical because you cannot defend against threats you do not know exist. This file walks through every major attack vector in Kubernetes — from network exposure to RBAC misconfiguration to container escape — so you can systematically protect against each one.

---

## The External Attack Surface

### 1. The Kubernetes API Server

The API server is the central control plane. It is how `kubectl` commands are executed, how deployments are created, how everything in Kubernetes is managed.

**The problem:** The API server is exposed over HTTPS on port 6443 (by default). If authentication fails or is misconfigured, an attacker can:
- List all resources (pods, secrets, ConfigMaps)
- Create new pods (and run arbitrary code)
- Modify existing resources
- Access service account tokens
- Potentially access etcd data

**Real-world example:**
Many teams configure their Kubernetes cluster with the API server accessible from the internet for remote access. They use weak credentials or no authentication at all. Attackers scan the internet for open Kubernetes API servers and compromise them within hours.

**How to defend:**
- Never expose the API server to the internet without strong authentication
- Use RBAC to restrict API access
- Enable audit logging to track who accessed what
- Use network policies to restrict API access to authorized users only

### 2. Kubelet API

The kubelet runs on every node. It exposes an HTTP API on port 10250 (by default) that allows you to:
- Read logs from running pods
- Execute commands inside containers
- Get pod and node metrics

**The problem:** By default, the kubelet API is accessible from within the cluster without authentication. If an attacker gains access to any node, they can:
- Access logs from all pods running on that node
- Execute arbitrary commands inside any container
- Potentially compromise other pods

**How to defend:**
- Enable kubelet authentication and authorization
- Use node firewall rules to restrict kubelet API access
- Monitor and audit kubelet API access

### 3. Container Registry

Your container images are stored in a registry (DockerHub, ECR, GCR, etc.). If an attacker gains access to your registry, they can:
- Modify your application images (inject malware)
- Read proprietary code
- Access credentials stored in images

**How to defend:**
- Use private registries with access control
- Scan images for vulnerabilities before pushing
- Sign images with digital signatures
- Restrict who can push images (only CI/CD pipelines)
- Never store credentials in images

---

## The Internal Attack Surface

Once an attacker has access to *any* pod inside your cluster, they have gained a foothold. Now the internal attack surface matters.

### 1. Service-to-Service Communication

By default, every pod can communicate with every other pod inside the cluster. If an attacker compromises your frontend pod, they can immediately connect to your database pod, cache pod, or any other service.

**Example attack:**
```
1. Attacker compromises the frontend pod
2. Attacker discovers environment variables (find $SERVICE_HOST, $SERVICE_PORT)
3. Attacker connects to the backend service
4. Attacker discovers database connection details
5. Attacker connects to the database and exfiltrates data
```

**How to defend:**
- Implement Network Policies (explicit allow-list model)
- By default, deny all pod-to-pod communication
- Only allow specific pods to talk to specific services
- Segment your cluster by network zones

### 2. Secrets Exposure

Secrets in Kubernetes are base64-encoded by default. This is **not encryption** — it is just encoding. Anyone who can read the Secret resource can read your database passwords, API keys, and credentials.

**How secrets can be exposed:**
- `kubectl get secret -o yaml` (if user has permission)
- Pod crashes and logs show environment variables
- Logs from application that prints environment variables
- etcd is compromised (contains all secrets)

**Real-world breach:**
A developer accidentally added a Slack webhook secret to a ConfigMap. The ConfigMap was readable by all pods in the cluster. An attacker who compromised any pod could read all Slack integration credentials.

**How to defend:**
- Encrypt secrets at rest (use EBS encryption, KMS, Vault)
- Restrict who can read Secrets (RBAC)
- Use external secret managers (HashiCorp Vault, AWS Secrets Manager)
- Enable audit logging for Secret access
- Never log or print secrets

### 3. Service Account Tokens

Every pod gets a service account token (mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`). This token allows the pod to authenticate to the Kubernetes API.

**The problem:** If a pod is compromised, the attacker has the service account token. They can now use the Kubernetes API to:
- List all resources
- Create new pods
- Modify deployments
- Delete resources
- Access other pods' secrets

**How to defend:**
- Use restrictive RBAC roles for service accounts (least privilege)
- Disable automounting of service account tokens if not needed
- Use Workload Identity / Pod Identity to authenticate to cloud services (avoid long-lived tokens)

### 4. Etcd Compromise

Etcd is the key-value store that holds *all* Kubernetes state — every resource, every secret, every ConfigMap. If etcd is compromised, the entire cluster is compromised.

**How etcd can be compromised:**
- Attacker gains access to the control plane node
- Etcd API is exposed without authentication
- Etcd backup is stored insecurely
- Credentials for accessing etcd are leaked

**What an attacker can do:**
- Read all secrets cluster-wide
- Read all ConfigMaps
- Modify any resource (deployments, services, RBAC roles)
- Shut down the cluster (delete all resources)

**How to defend:**
- Encrypt etcd at rest
- Restrict etcd API access (firewall, TLS)
- Isolate control plane nodes in a separate network
- Backup etcd encrypted and store securely
- Enable audit logging for etcd access

---

## Container Escape

A **container escape** is when an attacker breaks out of the container's isolation and gains access to the host kernel or other containers.

### 1. Privilege Escalation from Root

If a container runs as root (UID 0) and has access to the host kernel, an attacker can potentially:
- Exploit kernel vulnerabilities
- Access all mounted volumes
- See all processes on the host
- Potentially access other containers

**Example:** A container running as root can see the host's `/proc` filesystem. An attacker might find credentials or sensitive information there.

**How to defend:**
- Run containers as non-root users (SecurityContext: runAsUser, runAsNonRoot)
- Remove unnecessary Linux capabilities
- Use read-only root filesystem

### 2. Privileged Containers

A privileged container has access to the host's devices and kernel. This is essentially "full host access."

```yaml
# WRONG — privileged container
apiVersion: v1
kind: Pod
metadata:
  name: privileged-app
spec:
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      privileged: true  # DANGEROUS!
```

An attacker in a privileged container can:
- Access `/dev` (direct hardware access)
- Load kernel modules
- Reboot the host
- Access all host processes

**When to use:** Only for system daemons like kubelet, docker daemon, or monitoring agents. Never for application pods.

### 3. Host Path Mounts

```yaml
# WRONG — container can access host filesystem
apiVersion: v1
kind: Pod
metadata:
  name: risky-app
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /  # Mounts the entire host filesystem!
```

If this container is compromised, the attacker has read/write access to the entire host filesystem. They can:
- Read host SSH keys
- Modify host configurations
- Access other pods' data
- Escalate to host compromise

**When to use:** Only when absolutely necessary (e.g., logging agents that need to read host logs). Restrict the path to the minimum needed.

---

## RBAC Misconfiguration

Weak RBAC is one of the most common security failures in Kubernetes.

### The Over-Permissive Admin

```yaml
# WRONG — too much power
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: my-app-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]  # Can do ANYTHING
```

An attacker who compromises a pod with this role can:
- Delete all resources
- Create new pods with elevated privileges
- Modify RBAC rules
- Take over the entire cluster

### Service Account with Wildcard Permissions

```yaml
# WRONG
apiVersion: v1
kind: ServiceAccount
metadata:
  name: risky-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: risky-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin  # Dangerous!
subjects:
- kind: ServiceAccount
  name: risky-sa
  namespace: default
```

Every pod using this service account can do anything in the cluster.

---

## The Complete Attack Chain

Here is a realistic attack chain showing how multiple vulnerabilities can be exploited:

```
1. Attacker scans the internet
   → Finds Kubernetes API server exposed

2. Attacker probes for weak authentication
   → Default credentials still enabled
   → Attacker logs in as admin

3. Attacker creates a malicious pod
   → Uses ClusterAdmin service account (overly permissive RBAC)
   → Runs with privileged: true (unnecessary)

4. Attacker's pod is now inside the cluster
   → Can communicate with all other pods (no Network Policies)
   → Reads pod logs and finds database credentials
   → Connects to database and exfiltrates data

5. Attacker escalates
   → Uses privilege escalation to escape container
   → Gains root on the host node
   → Installs backdoor for persistent access
   → Moves laterally to other nodes

6. Attacker pivots to other systems
   → Uses found credentials to access AWS account
   → Spins up their own cluster
   → Deploys mining software
```

**Each step could have been prevented:**
1. Don't expose API server to internet
2. Use strong authentication + RBAC
3. Use least privilege RBAC + Pod Security Standards
4. Implement Network Policies
5. Run non-root, remove capabilities, use read-only filesystems
6. Rotate credentials, use short-lived tokens

---

## Summary

Kubernetes has a large attack surface spanning the external API server, container runtime, etcd backend, and internal pod-to-pod communication. Attackers can exploit RBAC misconfigurations, weak secrets management, container escapes, and lateral movement between pods.

The good news: **most attacks are preventable** through layered security controls. In the next file, we will dive deep into **security best practices** — the concrete, actionable steps you take to harden your cluster against each attack vector covered here.
