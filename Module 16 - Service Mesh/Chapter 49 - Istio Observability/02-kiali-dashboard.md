# Kiali Dashboard

## Why a Dashboard Helps

Metrics are useful, but sometimes you want to see the whole mesh as a picture.

That is what Kiali gives you.

Kiali is the dashboard that helps you understand Istio traffic, services, and policies.

---

## The Story

You are on call.

Someone says the checkout flow is broken.

Instead of reading logs from ten places, you open Kiali and look at the traffic graph.

Now you can see what is talking to what.

That saves time.

---

## What Kiali Shows

Kiali can show:

- Service graph
- Traffic flow
- Health status
- VirtualService and DestinationRule details
- Policy coverage

### Why This Is Helpful

It turns a big mesh into something a human can understand quickly.

---

## Service Graph Example

Think of this chain:

- frontend
- orders
- payments
- inventory

Kiali can show arrows between them so you can see the path the request is taking.

That is much easier than guessing from raw metrics alone.

---

## Common Troubleshooting Use

### Story Example

The frontend service is healthy.

The orders service is healthy.

The payments service has a red warning.

Now you know where to look first.

That is the power of a visual dashboard.

---

## Kiali and Policy Awareness

Kiali can also help you see whether your traffic policies are correct.

For example, if a VirtualService exists but the traffic graph looks wrong, you can spot that much faster in a dashboard than in YAML alone.

---

## Example Workflow

1. Open Kiali
2. Find the namespace
3. Look at service traffic graph
4. Check health indicators
5. Inspect routing and policies

That is a very practical debugging flow.

---

## What Kiali Is Not

Kiali is not the mesh itself.

It is the window into the mesh.

That distinction matters because the real traffic logic still lives in Istio and the proxies.

---

## AWS EKS Story

In AWS EKS, Kiali is especially helpful when many teams share one cluster.

The platform team can use it to see if traffic is behaving the way the mesh policy says it should.

That helps reduce surprises.

---

## Common Problems

### Problem 1: No Graph Data

Metrics or telemetry may not be fully enabled.

### Problem 2: Wrong Namespace

You may be looking at the wrong part of the mesh.

### Problem 3: Missing Traffic

If sidecars are missing, Kiali cannot show the whole story.

---

## Summary

Kiali turns Istio data into a visual story. It helps you see service graphs, traffic flow, and policy health so you can debug faster and understand the mesh at a glance.

Next, Chapter 49 moves to distributed tracing, where one request becomes a trail you can follow across many services.
