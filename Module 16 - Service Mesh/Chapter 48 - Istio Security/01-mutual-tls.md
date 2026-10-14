# Mutual TLS in Istio

## Why Security Needs a New Story

Your services may already be inside the cluster.

That does not mean they are automatically safe.

If one pod is compromised, plain service-to-service traffic can be read or abused unless you protect it.

Istio uses mutual TLS, or mTLS, to make that traffic safer.

---

## What mTLS Means

Normal TLS proves the server is real.

Mutual TLS goes one step further.

It proves both sides are real:

- The client proves who it is
- The server proves who it is

That gives you stronger identity inside the mesh.

### Easy Analogy

Think of it like a locked gate where both people must show a badge before the door opens.

---

## Why mTLS Matters

If a service is compromised, an attacker may try to move sideways to another service.

mTLS helps because:

- Traffic is encrypted
- Services prove identity
- Fake internal traffic becomes harder
- Trust is no longer based only on being inside the cluster

That is a big step toward a zero-trust style network.

---

## How Istio Handles Identity

Istio gives each workload an identity.

That identity is tied to the service account and the mesh certificate.

The proxy uses that identity when talking to other services.

### What This Means in Practice

You do not just say:

> "This request came from some pod."

You say:

> "This request came from a trusted workload with a known mesh identity."

That is much stronger.

---

## mTLS Modes

| Mode | Meaning | When to Use |
|------|---------|-------------|
| Disabled | No mTLS | Not recommended for secure clusters |
| Permissive | Accepts both mTLS and plain traffic | Good for migration |
| Strict | Only mTLS allowed | Best for secure production |

### Why Permissive Helps

If you are adding Istio to an existing cluster, permissive mode helps you move gradually.

That avoids breaking every service on day one.

### Migration Path

You can move from permissive to strict in steps:

1. Enable Istio injection
2. Turn on permissive mTLS
3. Watch traffic and logs
4. Fix services that still talk plain HTTP
5. Move to strict mode after the app is ready

That is the safest way to protect a live cluster.

---

## Example PeerAuthentication

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

This says every workload in the namespace must use mTLS.

If you want to start gently, change `STRICT` to `PERMISSIVE` first and test the namespace.

---

## The mTLS Flow

```text
Service A
	↓ encrypted traffic
Istio proxy on A
	↓ certificate identity check
Istio proxy on B
	↓ encrypted traffic
Service B
```

The proxies handle the security work while the app stays focused on business logic.

### What the App Sees

From the app's point of view, nothing special is happening.

It still makes HTTP calls.

The proxy handles the secure transport around those calls.

---

## Why DevOps Teams Care

Security is easier when it is standard.

With Istio mTLS, you can make encrypted service traffic the default instead of a special project.

That helps with:

- Internal trust
- Audit requirements
- Production hardening
- Safer platform design

---

## Common Problems

### Problem 1: Traffic Breaks After Enabling Strict mTLS

That usually means one service is still talking plain HTTP.

### Problem 2: Old App Cannot Handle mTLS Yet

Use permissive mode first, then move to strict later.

### Problem 3: Proxy or Certificate Issue

Check the sidecar injection and mesh health.

---

## AWS EKS Example

In AWS EKS, mTLS is helpful when teams want a shared cluster but still need strong service identity.

That means your internal services can trust mesh identity instead of just network location.

### What You Gain in Practice

- Better internal trust boundaries
- Less chance of lateral movement
- Clearer identity for audit and policy
- Easier migration to stronger security later

---

## Practical Verification Commands

```bash
kubectl get peerauthentication -n production
kubectl describe peerauthentication default -n production
kubectl logs -n istio-system deploy/istiod --tail=20
```

If strict mode is not working, those commands are the first place to look.

---

## Example Namespace Rollout

```bash
kubectl label namespace staging istio-injection=enabled
kubectl apply -f peerauth-permissive.yaml
kubectl apply -f sample-workload.yaml
```

Use a small namespace first, then move the pattern to production once the traffic is stable.

---

## Summary

Mutual TLS makes Istio traffic safer by encrypting it and proving identity on both sides. It is one of the biggest reasons people adopt a service mesh in production.

Start with permissive mode if you are migrating, then move to strict mode when your services are ready. Next, Chapter 48 moves to authorization policies so you can control who is allowed to talk to whom inside the mesh.
