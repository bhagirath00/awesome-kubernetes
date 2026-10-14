# Linkerd Overview

## What Linkerd Is

Linkerd is another service mesh for Kubernetes.

It is often chosen when a team wants service mesh features but prefers a simpler setup than some other platforms.

Linkerd focuses on:

- Security
- Observability
- Reliability
- Lower operational complexity

---

## Why Teams Choose Linkerd

Some service meshes are powerful but heavy.

Linkerd is attractive when you want mesh benefits without making the platform too hard to run.

### Common Reasons

- Easier to understand
- Less configuration overhead
- Good default behavior
- Practical for smaller platform teams

---

## How Linkerd Works

Like other service meshes, Linkerd uses proxies in the data path.

The proxy handles traffic between services, and the control plane manages the configuration.

### Main Idea

You should not think of Linkerd as an app feature.

Think of it as network behavior wrapped around your services.

---

## Traffic Encryption

Linkerd can encrypt traffic between services with mutual TLS.

That gives you:

- Encrypted traffic in transit
- Service identity
- More trust inside the cluster

### Why It Matters

If one workload is compromised, encrypted and identity-based communication makes lateral movement harder.

---

## Observability

Linkerd gives you useful traffic metrics.

You can see:

- Success rate
- Request latency
- Service-to-service traffic paths
- Failure patterns

That helps you debug production issues faster.

### DevOps Example

If users say the checkout page is slow, you can look at mesh metrics and see whether the API, payment service, or database path is the real problem.

---

## Linkerd Example

```bash
linkerd check
linkerd viz install | kubectl apply -f -
```

Those commands show the style of Linkerd setup.

### Example Policy Thinking

With Linkerd, you may:

- Put all app traffic through the mesh
- Encrypt service calls by default
- Watch latency and error rates from one dashboard

---

## Linkerd vs Istio

| Area | Linkerd | Istio |
|------|---------|-------|
| Complexity | Lower | Higher |
| Features | Focused | Very broad |
| Learning curve | Easier | Harder |
| Traffic control | Good | Very strong |
| Best for | Simpler platform teams | Advanced mesh use cases |

### How to Choose

- Choose Linkerd if you want a simpler mesh with good defaults
- Choose Istio if you need advanced routing and policy features

---

## Linkerd in AWS EKS

Linkerd fits AWS EKS well when you want mesh security and visibility without a very large operational burden.

### Good Fit Scenarios

- Small platform team
- Moderate number of services
- Need for encrypted service calls
- Need for clear traffic metrics

### Example

A team running an internal SaaS platform may use Linkerd to:

- Encrypt traffic between APIs
- See request failures quickly
- Reduce the amount of custom code for retries and timeouts

---

## Common Problems

### Problem 1: Proxy Not Injected

Check namespace labels and installation state.

### Problem 2: Traffic Metrics Missing

Make sure the viz components are installed and the services are in the mesh.

### Problem 3: Slow Requests

Look at the app, the proxy, and the backend service separately.

Sometimes the issue is not the mesh itself but the backend latency that the mesh reveals.

---

## Summary

Linkerd is a service mesh that gives you encryption, observability, and traffic control with a simpler operational model than many alternatives.

It is a good option when your team wants mesh benefits without adding too much complexity. Keep the install small, watch the metrics, and use it to improve service reliability in your AWS EKS environment.

Next, Chapter 41 ends with service mesh use cases and where mesh tools fit in real DevOps work.
