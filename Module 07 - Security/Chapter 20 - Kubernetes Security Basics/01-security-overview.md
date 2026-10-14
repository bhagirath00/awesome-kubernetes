# Kubernetes Security Overview

## Introduction

Imagine you just deployed your first Kubernetes cluster on AWS. It is running a payment application processing customer transactions. Everything looks good — the pods are running, traffic is flowing, the dashboard is green.

But here is what most teams don't realize: **by default, your cluster is wide open**.

Any pod in your cluster can read any secret. Pods can talk to any other pod. If someone compromises a single pod, they can potentially access everything — databases, secrets, other applications. The blast radius is your entire cluster.

This is not Kubernetes's fault. Kubernetes gives you the *tools* to secure your cluster, but it does not force you to use them. Security is something you must actively build in from the start — not something you add later.

This chapter teaches you Kubernetes security from the ground up. We will show you what can go wrong, where the weak points are, and step-by-step how to lock down your cluster. By the end, you will understand how to deploy a production-grade secure Kubernetes environment.

---

## What Happens When Security Goes Wrong

Let me tell you a story that happens more often than you would think.

A startup launched a Kubernetes cluster on AWS to run their SaaS application. They followed the quick-start guides, got it running fast, and shipped it. Performance was great. They were proud of themselves.

Two months later, a researcher discovered a vulnerability in their application code. Before they could patch it, an attacker exploited it and gained access to one pod. From that single compromised pod, the attacker:

- Read all Kubernetes secrets (database passwords, API keys)
- Connected directly to the database and exfiltrated customer data
- Created new admin pods to maintain access
- Moved laterally through the cluster
- Deleted logs to cover their tracks
- By the time anyone noticed, the damage was done

**The cost? Millions of dollars in breach notification, compliance fines, and lost customers.**

Now let me tell you a different story about a company that did it right.

They deployed similar architecture, but with security controls:

- The compromised pod had minimal permissions (couldn't read other namespaces)
- Network policies prevented it from talking to the database
- The pod was running as non-root with read-only filesystem
- When the attacker tried to escalate privileges, the SecurityContext blocked it
- Audit logs captured every suspicious action
- The incident was contained to a single pod and detected within minutes

**The cost? A quick patch deployment and a blog post about their secure practices.**

The difference between these two stories is not luck. It is security.

---

## Kubernetes Security: Three Layers of Defense

Kubernetes security is like a bank. Banks don't rely on a single lock on the front door. They have multiple security layers:

1. Security guards at the entrance (authentication — are you who you say you are?)
2. Tellers who check your authorization (what are you allowed to do?)
3. Vaults with additional controls (even if someone gets past the guards, the vault is still locked)
4. Cameras and monitoring everywhere (audit trails showing who did what)

Kubernetes works the same way. Let me walk you through the layers:

### Layer 1: Gate Keeper (Authentication & Authorization)

**Authentication** = "Who are you?" — verified by checking credentials
**Authorization** = "What are you allowed to do?" — verified by checking roles

When you run `kubectl apply -f deployment.yaml`:

```
Kubernetes receives your request
    ↓
Authentication: "Are you really patel@company.com?"
    (Check certificate, token, password)
    ↓
If authentication fails → Request rejected
    ↓
Authorization: "Does patel@company.com have permission to create deployments?"
    (Check RBAC rules)
    ↓
If authorization fails → Request rejected
    ↓
Request proceeds
```

If either check fails, Kubernetes says no. Even if you have valid credentials but lack permission, you are blocked.

### Layer 2: Policy Enforcer (Admission Control)

After a request passes authentication and authorization, **admission controllers** apply additional rules. Think of them as policy checkers.

Example: Your company policy says "all pods must have CPU and memory limits." An admission controller:
- Checks if the pod has limits
- If yes → let it through
- If no → either automatically add limits or reject the pod

Admission controllers can:
- Validate requests ("does this make sense?")
- Mutate requests ("I am automatically adding security controls")
- Reject requests ("this violates policy")

### Layer 3: Runtime Prison (Pod Security)

Even after a pod is running, Kubernetes restricts what it can do:

**SecurityContext** — runs the pod with limited privileges:
- Must run as non-root user (UID 1000, not 0)
- Cannot escalate to root
- Root filesystem is read-only (cannot modify code or install tools)
- Has no special Linux capabilities

**Network Policies** — controls which pods can talk to which:
- By default, pods cannot communicate with each other
- Only explicitly allowed pods can exchange traffic

**Resource Limits** — caps resource usage:
- Pod can use at most 512MB RAM
- Pod can use at most 0.5 CPU cores
- If it exceeds limits, Kubernetes kills it (before it hurts other pods)

**Pod Security Standards** — enforces security baselines:
- Restricted, baseline, or permissive
- Think of them like "strict", "moderate", or "relaxed" security policies

---

## The Most Common Security Mistakes

Let me show you what I see teams get wrong constantly. These are the biggest holes in Kubernetes security:

### Mistake 1: Everything Runs as Root

A new Kubernetes user asks: "Can I just run my container without specifying a user?"

Yes. It will run as root (UID 0) by default. This is the security equivalent of giving the attacker a master key.

```yaml
# WRONG — nobody specified who should run this
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: myapp:latest
    # No securityContext = runs as root
```

If this application gets hacked, the attacker:
- Is root inside the container
- Can modify any file
- Can install backdoors and tools
- Might escape to the host node

Compare to:

```yaml
# CORRECT — explicitly run as non-root
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
```

Now even if hacked, the attacker is a regular user. They cannot escalate to root. The filesystem is read-only, so they cannot install tools. The damage is contained.

### Mistake 2: Every Pod Can Talk to Every Other Pod

Here is how most Kubernetes clusters are deployed:

```
Pod A (frontend)
    ↓ can talk to
Pod B (backend)
    ↓ can talk to
Pod C (database)
```

But also:

```
Pod A (frontend)
    ↓ CAN ALSO talk to
Pod C (database) — directly! Skipping backend!
```

This is the **default behavior**. By default, any pod can communicate with any other pod.

Now imagine Pod A gets compromised. The attacker:
- Immediately finds the database connection details from environment variables
- Connects directly to the database
- Exfiltrates all customer data
- Nobody stops them

**This is prevented by Network Policies** — they are like firewall rules for pods:

```yaml
# Step 1: Block all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

---
# Step 2: Allow frontend to receive from internet
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      tier: frontend
  ingress:
  - from:
    - podSelector: {}  # Anyone
    ports: [{protocol: TCP, port: 80}]

---
# Step 3: Allow backend to receive from frontend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports: [{protocol: TCP, port: 8080}]

---
# Step 4: Allow database to receive from backend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database
spec:
  podSelector:
    matchLabels:
      tier: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports: [{protocol: TCP, port: 5432}]
```

Now if frontend is compromised, the attacker cannot reach the database. No network path exists. They can only move to backend (which you allowed).

### Mistake 3: No Limits on Pod Resources

A pod gets deployed with no resource limits. It starts consuming memory and CPU. Before you know it, it has eaten all available resources on the node. Other pods start crashing.

This is called a **denial of service** — one misbehaving pod takes down everything.

```yaml
# WRONG — pod can use unlimited resources
apiVersion: v1
kind: Pod
metadata:
  name: runaway-app
spec:
  containers:
  - name: app
    image: memory-leak:latest
    # No resource limits
```

Versus:

```yaml
# CORRECT — pod is capped
apiVersion: v1
kind: Pod
metadata:
  name: well-behaved-app
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:  # "I need at least this much"
        memory: "256Mi"
        cpu: "100m"
      limits:    # "I cannot use more than this"
        memory: "512Mi"
        cpu: "500m"
```

If the pod tries to use more than 512Mi of RAM, Kubernetes kills it. Better one pod dies than all pods crash.

### Mistake 4: Secrets in Plain Text in Pod Definitions

A developer does this:

```yaml
# WRONG — database password visible to anyone
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_PASSWORD
      value: "SuperSecretPassword123"
```

Now anyone who can run `kubectl describe pod app` can read the password.

Instead, use Secrets:

```yaml
# Store the secret separately
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  password: "SuperSecretPassword123"

---
# Pod references it (doesn't contain it)
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

Now the password is stored separately and can be encrypted at rest.

---

## The Security Checklist

Every Kubernetes cluster should implement:

| Layer | Control | Purpose |
|-------|---------|---------|
| **Authentication** | RBAC | Restrict who can do what |
| **Admission** | Pod Security Standards | Enforce security baselines |
| **Runtime** | SecurityContext | Restrict container capabilities |
| **Runtime** | Network Policies | Control pod-to-pod communication |
| **Runtime** | Resource Limits | Prevent resource starvation |
| **Storage** | Encryption at Rest | Protect secrets and etcd |
| **Monitoring** | Audit Logging | Track who did what when |
| **Compliance** | Network Segmentation | Isolate teams/applications |

---

## What This Chapter Covers

We will go deep on each of these security layers:

1. **Chapter 20 (this file)** — Security overview and fundamentals
2. **Chapter 21** — RBAC (Role-Based Access Control)
3. **Chapter 22** — Pod Security (SecurityContext, Pod Security Standards)
4. **Chapter 23** — Network Policies (controlling pod communication)

By the end of this module, you will be able to:
- Design and implement RBAC roles with least privilege
- Configure SecurityContext to restrict container capabilities
- Set up Network Policies to control traffic flow
- Audit and monitor security in your cluster

---

## Summary

Kubernetes security is about **defense in depth** — multiple layers so that if one fails, others still protect you. Authentication and authorization prevent unauthorized users from accessing the cluster. Admission control enforces policies. SecurityContext and Network Policies restrict what running pods can do.

The difference between a secure Kubernetes cluster and a compromised one often comes down to whether these controls were configured properly. In the next file, we will dive deep into **RBAC (Role-Based Access Control)** — the foundation of who can do what in your cluster.
