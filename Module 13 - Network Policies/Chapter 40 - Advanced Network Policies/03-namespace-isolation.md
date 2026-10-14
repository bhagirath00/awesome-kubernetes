# Namespace Isolation

## Why Namespace Isolation Exists

Namespaces are one of the simplest ways to separate workloads in Kubernetes.

They let you group resources by team, app, or environment. But a namespace alone does not automatically stop traffic. If you want true separation, you need network policies too.

That is why namespace isolation is a mix of organization and security.

### Real-World Example

In an AWS EKS cluster, you might have:

- `dev` namespace for developers
- `staging` namespace for testing
- `production` namespace for live traffic

If those namespaces are not isolated, a mistake in dev can affect production.

---

## The Two Jobs of Namespaces

Namespaces do two important things:

1. They organize resources
2. They help you enforce boundaries

### Organization

You can keep each team’s resources separate.

### Security

You can say which namespaces are allowed to talk to each other.

That makes it easier to control blast radius.

---

## Namespace Isolation Pattern

The safest pattern is:

- Deny traffic by default
- Allow only required namespace-to-namespace paths
- Keep shared services in a controlled namespace

### Example Traffic Map

| Source Namespace | Destination Namespace | Allowed? | Reason |
|------------------|----------------------|----------|--------|
| frontend | api | Yes | App needs it |
| api | database | Yes | Backend access |
| dev | production | No | Unsafe |
| staging | production | No | Prevent accidental impact |
| monitoring | all | Limited | Metrics only |

---

## Basic Isolation Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: deny-all-in-namespace
	namespace: production
spec:
	podSelector: {}
	policyTypes:
		- Ingress
		- Egress
```

This policy says:

- Apply to all pods in production
- Block incoming traffic
- Block outgoing traffic

You then add specific allow rules.

### Allow Only Frontend Pods From Another Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: allow-frontend-to-api
	namespace: production
spec:
	podSelector:
		matchLabels:
			app: api
	policyTypes:
		- Ingress
	ingress:
		- from:
				- namespaceSelector:
						matchLabels:
							name: frontend
					podSelector:
						matchLabels:
							app: web
			ports:
				- protocol: TCP
					port: 8080
```

This rule allows only a specific workload in a specific namespace.

---

## Using Namespace Labels

Namespace labels make policies easier to manage.

### Example

```bash
kubectl label namespace frontend name=frontend
kubectl label namespace production name=production
kubectl label namespace dev name=dev
```

Then your policy can match on labels.

That is better than hardcoding names everywhere.

### Why This Matters in CI/CD

If your namespace is created by automation, labels let you apply the right policy consistently.

---

## DevOps Workflow for Isolation

In a DevOps team, namespace isolation usually follows this flow:

1. Create namespace for the environment
2. Apply default deny policy
3. Add required allow rules
4. Deploy workloads
5. Test connectivity
6. Monitor traffic and logs

### Example Pipeline Step

```bash
kubectl apply -f namespaces.yaml
kubectl apply -f default-deny.yaml
kubectl apply -f allow-rules.yaml
kubectl rollout status deployment/api -n production
```

That keeps the cluster repeatable and safer.

---

## Common Isolation Design

### Frontend Namespace

- Receives public traffic
- Talks only to API namespace

### API Namespace

- Receives traffic only from frontend namespace
- Talks only to database namespace

### Database Namespace

- Receives traffic only from API namespace
- No public access

### Monitoring Namespace

- Reads metrics from workloads
- Should not send traffic to application ports

---

## Problems You May Hit

### Problem 1: Everything Breaks After Deny Policy

That usually means you forgot an allow rule for:

- DNS
- Metrics
- Service-to-service calls
- Health probes

### Problem 2: Isolation Is Too Loose

If you allow whole namespaces without checking pod labels, you may open more than you meant to.

### Problem 3: Teams Use the Wrong Namespace

This is an organization problem as much as a network problem.

Fix it with:

- Clear namespace naming
- RBAC
- Admission controls
- CI/CD checks

---

## Namespace Isolation and AWS EKS

In AWS, namespace isolation helps you keep shared clusters practical.

You do not always need a separate cluster for every app.

Instead, you can use:

- One EKS cluster
- Multiple namespaces
- Network policies
- Strong RBAC

That gives you a balance between cost and safety.

---

## Summary

Namespace isolation helps you separate environments and teams inside one Kubernetes cluster. It works best when you combine namespaces with default deny policies and small allow rules.

Use namespaces for organization, then use network policies for real traffic control. In AWS EKS, this is a practical way to reduce risk without creating too many clusters.

Next, Chapter 40 moves to micro-segmentation and shows how to break the cluster into very small security zones.
