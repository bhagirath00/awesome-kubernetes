# Peer Authentication

## The Problem It Solves

You may want one part of the mesh to accept plain traffic while another part is already locked down with mTLS.

That is what PeerAuthentication helps with.

It tells Istio how the workload should handle incoming traffic from other workloads.

---

## What PeerAuthentication Means

PeerAuthentication controls whether Istio expects mTLS for inbound traffic.

| Mode | Meaning |
|------|---------|
| STRICT | Only mTLS is allowed |
| PERMISSIVE | Plain traffic and mTLS are both accepted |
| DISABLE | mTLS is not required |

### Why That Helps

It lets you move at the speed of your real cluster instead of forcing every service to change all at once.

---

## Example

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
	name: default
	namespace: production
spec:
	mtls:
		mode: STRICT
```

This says all workloads in the namespace must use mTLS.

---

## Migration Story

If you already have live traffic, going straight to strict mode can break old services.

So many teams do this:

1. Start permissive
2. Watch traffic and logs
3. Fix old services
4. Switch to strict mode

That gives you a safer path.

---

## Namespace and Workload Level

PeerAuthentication can be applied at different levels.

| Scope | Meaning |
|------|---------|
| Mesh-wide | Applies broadly to the whole mesh |
| Namespace | Applies to one namespace |
| Workload | Applies to one service or pod set |

That flexibility is useful when some services are ready and others are not.

---

## Common Problems

### Problem 1: Service Breaks After STRICT Mode

Some client is still not speaking mTLS.

### Problem 2: Policy Applied Too Broadly

You may have turned on strict mode before testing all traffic paths.

### Problem 3: Mixed Old and New Services

Use permissive mode while you migrate, then reduce the old paths one by one.

---

## Why This Is Practical in AWS EKS

In EKS, teams often migrate a large platform in stages.

PeerAuthentication fits that reality because it lets you secure one namespace first and expand later.

That makes the rollout safer and easier to manage.

---

## Summary

PeerAuthentication controls whether Istio expects mTLS, allows both mTLS and plain traffic, or disables mTLS. It is one of the key tools for moving a cluster toward stronger security without breaking everything at once.

That completes Chapter 48 and sets up Chapter 49, where the focus shifts from security to observability and how you can see what the mesh is doing.
