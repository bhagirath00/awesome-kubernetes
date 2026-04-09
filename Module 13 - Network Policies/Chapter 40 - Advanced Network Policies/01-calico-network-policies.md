# Calico Network Policies

## Why Network Policies Matter

When you build Kubernetes apps, every pod can talk to every other pod by default. That is easy at the start, but it is dangerous in real systems.

If one container gets compromised, the attacker may move sideways to databases, internal APIs, or admin tools. Network policies help you reduce that risk by letting you control which pods can talk to each other.

Calico is one of the most common tools for this job. It gives Kubernetes strong network policy support and works well in AWS EKS clusters.

### A Real DevOps Example

Imagine this setup:

- A frontend pod serves users
- An API pod handles business logic
- A database pod stores orders

Without policies, the frontend pod could try to reach the database directly. That is not a good design. You want the frontend to talk only to the API, and the API to talk only to the database.

That is the kind of rule Calico helps you enforce.

---

## What Calico Does

Calico adds a policy layer to Kubernetes networking. It can work with several networking modes and is often used with AWS EKS, on-prem clusters, and hybrid setups.

Calico helps you:

- Allow traffic only from trusted pods
- Block traffic from other namespaces
- Control ingress and egress
- Add default deny rules
- Build safer cluster boundaries

### Core Idea

Calico does not replace Kubernetes networking. It sits on top of it and applies rules before packets are allowed to reach a pod.

Think of it like a security guard at the gate of each service.

---

## Basic Policy Concepts

Before you write a policy, you need to know these parts:

| Term | Meaning |
|------|---------|
| PodSelector | Chooses which pods the policy applies to |
| NamespaceSelector | Chooses which namespace the traffic comes from or goes to |
| Ingress | Incoming traffic to a pod |
| Egress | Outgoing traffic from a pod |
| Policy Type | Whether the rule controls ingress, egress, or both |

### Default Behavior

If you do not define a policy, traffic is usually open inside the cluster.

That means:

- Any pod can talk to any pod
- Any namespace can reach other namespaces
- A compromised app can scan the cluster

Calico lets you change that behavior.

---

## Default Deny Pattern

The safest starting point is a default deny policy. This means no traffic is allowed until you explicitly allow it.

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
	name: default-deny
	namespace: production
spec:
	selector: all()
	types:
		- Ingress
		- Egress
```

This policy says:

- Apply to all pods in the production namespace
- Block all incoming traffic
- Block all outgoing traffic

That sounds harsh, but it is the cleanest way to build secure access rules from scratch.

### Why Default Deny Helps

It makes traffic paths explicit.

If an application breaks after you add default deny, that is useful. It means you now know the app was relying on hidden network access that should have been documented.

---

## Allowing Only Trusted Traffic

Once default deny is in place, you add small allow rules.

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
	name: allow-api-from-frontend
	namespace: production
spec:
	selector: app == 'api'
	ingress:
		- action: Allow
			source:
				selector: app == 'frontend'
			protocol: TCP
			destination:
				ports:
					- 8080
	types:
		- Ingress
```

This means:

- Only pods labeled `app=frontend` can reach the API pods
- Only TCP traffic on port 8080 is allowed
- All other traffic stays blocked

### Why Labels Matter

Calico policies usually match labels, not pod names.

That is useful because pod names change when pods restart. Labels are stable and fit DevOps automation better.

---

## Egress Control

Ingress protects what comes in. Egress protects what goes out.

That matters because a bad container may try to call an attacker-controlled endpoint or scan internal services.

### Example: Allow a Pod to Talk Only to DNS and the API

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
	name: allow-app-egress
	namespace: production
spec:
	selector: app == 'web'
	egress:
		- action: Allow
			destination:
				selector: app == 'api'
			protocol: TCP
			destination:
				ports:
					- 8080
		- action: Allow
			destination:
				nets:
					- 10.0.0.10/32
			protocol: UDP
			destination:
				ports:
					- 53
	types:
		- Egress
```

This policy allows:

- App traffic to the API
- DNS lookups to the cluster DNS server

Everything else is blocked.

---

## Calico in AWS EKS

In AWS EKS, Calico is often used when teams need more control than the basic network policy support gives them.

You may use Calico when you want:

- Strong namespace isolation
- Better traffic visibility
- More flexible policy design
- Security controls that fit enterprise compliance

### Common AWS Use Case

You run:

- A public frontend in one namespace
- Internal services in another namespace
- A database in a private namespace

With Calico, you can allow only the exact traffic paths you want.

That helps you keep the database private even inside the same cluster.

---

## Policy Order and Evaluation

Calico evaluates policies based on selectors and rules. The most specific allowed rule wins only if it matches the traffic.

That means you should design policies carefully.

### Good Practice

- Start with a default deny policy
- Add small allow rules
- Use clear labels like `app=frontend` and `tier=backend`
- Test each rule in a non-production namespace first

### Bad Practice

- Using one huge policy for every pod
- Writing rules without labels
- Allowing entire namespaces without need
- Forgetting egress rules when you lock down ingress

---

## Troubleshooting Calico Policies

### Problem 1: Application Suddenly Cannot Reach Another Service

Check:

```bash
kubectl get networkpolicy -n production
kubectl describe networkpolicy allow-api-from-frontend -n production
kubectl get pods --show-labels -n production
```

What to look for:

- Wrong label on the pod
- Wrong namespace
- Port mismatch
- Missing egress rule

### Problem 2: DNS Stops Working

This usually means your egress rules are too strict.

If a pod cannot resolve service names, it cannot talk to other services by name.

Fix:

- Allow traffic to CoreDNS or your cluster DNS service
- Allow UDP/TCP port 53

### Problem 3: Policy Looks Correct but Traffic Still Passes

This can happen if:

- The policy is in the wrong namespace
- The selector does not match any pod
- Another policy is allowing traffic

Use labels and namespace checks to confirm the real match.

---

## When Calico Is a Good Choice

Calico fits well when you need:

- Clear security boundaries
- Simple label-based policy rules
- Production-grade cluster segmentation
- AWS EKS security hardening

It is a strong match for DevOps teams that want network rules to be part of the platform design, not a manual afterthought.

---

## Summary

Calico helps you control pod-to-pod traffic in Kubernetes. It is useful when you want default deny behavior, namespace isolation, ingress control, and egress control.

Start with a simple model: deny everything, then allow only the traffic your app really needs. Use labels, test in staging first, and always include DNS and required service paths in your rules.

Next, Chapter 40 moves to Cilium and shows how it adds deeper traffic control and observability.
