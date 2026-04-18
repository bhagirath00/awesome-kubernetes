# Istio Authorization Policies

## Why Identity Alone Is Not Enough

mTLS tells you who the caller is.

That is good, but you still need to decide whether the caller is allowed to do the thing it wants to do.

That is what authorization policies are for.

---

## The Security Story

Think of mTLS as the badge check at the front door.

Authorization is the decision to let that person enter a specific room.

You may trust the caller enough to talk to the mesh.

But that does not mean it should reach every service.

---

## What Authorization Policies Control

Istio authorization policies can control access based on:

- Source identity
- Namespace
- Service account
- Request path
- HTTP method

That gives you very fine-grained control.

---

## Basic Example

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
	name: allow-frontend-to-api
	namespace: production
spec:
	selector:
		matchLabels:
			app: api
	rules:
		- from:
				- source:
						principals:
							- cluster.local/ns/production/sa/frontend
```

This means only the `frontend` service account can talk to the API.

---

## Why Service Account Identity Is Useful

Pod names change.

Pod IPs change.

Service account identity is much more stable.

That makes it a much better fit for policy rules.

---

## Request-Based Authorization

You can also make policy decisions based on request details.

### Example Story

You want the API service to allow:

- `GET /orders`
- `POST /orders`

But not:

- `DELETE /orders`

That keeps dangerous actions limited.

### Example

```yaml
rules:
	- to:
			- operation:
					methods: ["GET", "POST"]
					paths: ["/orders"]
```

---

## Default Deny Mindset

The safest model is to deny by default and allow only what is needed.

That reduces accidental access.

It also gives you a clean security story when someone asks why one service can talk to another.

---

## Policy Layers

| Layer | What It Controls |
|------|------------------|
| mTLS | Who the caller is |
| AuthorizationPolicy | What the caller may do |
| NetworkPolicy | Whether traffic can move at all |

Use them together for stronger control.

---

## Common Mistakes

### Mistake 1: Selector Does Not Match

If the policy selector does not match a workload, it does nothing.

### Mistake 2: Too Broad Rules

If you allow entire namespaces without thinking, you may open too much.

### Mistake 3: Forgetting Human Workflows

Admin tools, metrics, and health checks may need separate rules.

---

## AWS EKS Use Case

In AWS EKS, you may have:

- frontend service account
- API service account
- billing service account

Authorization policies let you decide which service account may call which backend.

That is a very clean fit for enterprise security.

---

## Summary

Istio authorization policies decide who may do what once identity has already been established. They let you build a strong security model using service accounts, request paths, and method rules.

Next, Chapter 48 moves to PeerAuthentication, which shows how Istio controls whether services must use mTLS or can still accept plain traffic during migration.
