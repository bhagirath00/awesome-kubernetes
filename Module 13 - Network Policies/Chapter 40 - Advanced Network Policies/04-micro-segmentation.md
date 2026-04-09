# Micro-Segmentation

## What Micro-Segmentation Means

Micro-segmentation means splitting your network into very small security zones.

Instead of saying, “The whole cluster can talk to everything,” you say, “Only this service can reach that service on this port.”

That is a much safer design for modern Kubernetes applications.

### Simple Analogy

Think of a large office building.

- Bad design: every employee can enter every room
- Better design: people can enter only the rooms they need

Micro-segmentation brings that same idea to Kubernetes traffic.

---

## Why It Matters in DevOps

DevOps teams run many small services.

If one service is compromised, you want the damage to stay small.

Micro-segmentation helps you:

- Reduce blast radius
- Limit lateral movement
- Meet compliance requirements
- Make security rules easier to audit

---

## Example Application Layout

Imagine an order platform:

- `frontend`
- `orders-api`
- `payments-api`
- `inventory-api`
- `postgres`

Without segmentation, any service might reach any other service.

With segmentation, you can decide:

- frontend can call orders-api
- orders-api can call payments-api and inventory-api
- payments-api can call postgres only
- nothing else is allowed

That is much safer.

---

## Micro-Segmentation Policy Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
	name: orders-api-policy
	namespace: production
spec:
	podSelector:
		matchLabels:
			app: orders-api
	policyTypes:
		- Ingress
		- Egress
	ingress:
		- from:
				- podSelector:
						matchLabels:
							app: frontend
			ports:
				- protocol: TCP
					port: 8080
	egress:
		- to:
				- podSelector:
						matchLabels:
							app: payments-api
			ports:
				- protocol: TCP
					port: 8081
		- to:
				- podSelector:
						matchLabels:
							app: inventory-api
			ports:
				- protocol: TCP
					port: 8082
```

This policy gives `orders-api` only the access it truly needs.

---

## Segmenting by Function

You can segment a cluster in several ways:

| Segment Type | Example | Why It Helps |
|-------------|---------|--------------|
| By app | frontend, api, db | Limits service movement |
| By team | team-a, team-b | Separates ownership |
| By environment | dev, stage, prod | Prevents cross-environment traffic |
| By sensitivity | public, internal, secret | Protects sensitive data |

### Good Segmentation Rule

Start with business functions, not with technical complexity.

If you can explain the rule to a junior engineer in one sentence, it is probably a good rule.

---

## Small Zones, Small Risk

The main benefit of micro-segmentation is reduced risk.

If a pod is attacked:

- It cannot scan the whole cluster
- It cannot reach admin services
- It cannot reach databases unless allowed

That limits the damage.

### Security Value

Micro-segmentation is useful for:

- PCI-DSS environments
- Finance systems
- Healthcare workloads
- Internal platform teams

---

## Operational Tradeoffs

Micro-segmentation is powerful, but it adds work.

### Benefits

- Stronger security
- Clear service ownership
- Better audit trails
- Lower blast radius

### Costs

- More policies to manage
- More testing needed
- More risk of blocking valid traffic
- More documentation required

### What to Do

Use automation.

If your policies are created by Terraform, Helm, or GitOps, they become easier to maintain.

---

## Testing Segmentation Safely

Before you apply policies to production:

1. Test in a dev namespace
2. Add one policy at a time
3. Check DNS, service calls, and health probes
4. Watch app logs
5. Watch network policy events

### Useful Commands

```bash
kubectl get networkpolicy -n production
kubectl describe networkpolicy orders-api-policy -n production
kubectl get pods --show-labels -n production
kubectl logs deploy/orders-api -n production
```

---

## Real AWS EKS Scenario

In AWS EKS, you may run:

- Public services in one namespace
- Internal services in another
- Shared tools in a third

Micro-segmentation makes it possible to keep them in one cluster while still controlling access.

That saves money compared to running separate clusters for every team.

---

## When to Use Micro-Segmentation

Use it when:

- You have many microservices
- You need strict security control
- You are handling sensitive data
- You want to reduce blast radius

Skip it for very small test clusters where the policy overhead is not worth it.

---

## Summary

Micro-segmentation breaks a Kubernetes cluster into very small security zones. It helps you control service-to-service access, reduce blast radius, and improve compliance.

The tradeoff is policy management overhead, so keep rules simple, test them carefully, and automate them in your DevOps workflow.

That completes Chapter 40 and sets you up for Chapter 41, where the focus shifts from network policy to service mesh basics.
