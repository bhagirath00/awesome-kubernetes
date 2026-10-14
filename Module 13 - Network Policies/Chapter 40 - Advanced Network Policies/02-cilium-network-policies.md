# Cilium Network Policies

## Why Cilium Is Different

Cilium is another network policy tool for Kubernetes, but it goes beyond simple allow and deny rules.

It uses eBPF to observe and control network traffic more efficiently. That makes it interesting for clusters that need better performance and better visibility.

If Calico is like a gatekeeper, Cilium is like a gatekeeper with a full camera system and activity log.

### What You Get With Cilium

- Network policy enforcement
- Better traffic visibility
- Layer 7 policy control
- Service awareness
- Lower overhead in many environments

---

## eBPF in Simple Terms

eBPF stands for extended Berkeley Packet Filter.

You do not need to know the kernel internals to use it, but the idea is simple:

- It lets the kernel run small, safe programs fast
- Those programs can inspect packets and system behavior
- Cilium uses that power to manage traffic with less overhead

### Why DevOps Teams Care

In a busy Kubernetes cluster, every small efficiency gain matters.

If you have many services and many network policies, you want the policy engine to be fast and reliable.

---

## Basic Cilium Policy Example

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
	name: frontend-to-api
	namespace: production
spec:
	endpointSelector:
		matchLabels:
			app: api
	ingress:
		- fromEndpoints:
				- matchLabels:
						app: frontend
			toPorts:
				- ports:
						- port: "8080"
							protocol: TCP
```

This policy means:

- The API pod can receive traffic only from frontend pods
- The traffic must go to port 8080
- Other sources are blocked

The structure is similar to other Kubernetes policies, but Cilium gives you more advanced matching options.

---

## Layer 7 Policy Control

One powerful feature of Cilium is application-aware policy.

That means you can control traffic based not only on IP and port, but also on what the request is trying to do.

### Example Use Case

Suppose your API has these paths:

- `GET /health`
- `GET /orders`
- `POST /orders`
- `DELETE /admin`

You may want:

- Frontend users to call `GET /orders`
- Admin tools to call `DELETE /admin`
- Everything else to be blocked

That kind of control is stronger than simple port filtering.

### Why This Helps

If an attacker gets into one pod, they may still be blocked from calling dangerous endpoints.

That gives you defense in depth.

---

## Service-to-Service Visibility

Cilium can help you see which service talks to which other service.

That is very useful when you are debugging a production outage.

### Real Debugging Scenario

You see:

- Frontend latency is rising
- The API seems healthy
- The database looks fine

With visibility tools, you can ask:

- Which service started calling the wrong endpoint?
- Which pod is generating too much traffic?
- Is the failure on ingress, egress, or DNS?

This shortens incident response time.

---

## Why Cilium Helps in AWS EKS

In AWS EKS, Cilium can fit well when you need stronger observability and tighter service controls.

Common reasons to choose it:

- You want advanced policy logic
- You want better packet-level visibility
- You want policy and telemetry in one place
- You have many microservices and need better troubleshooting

### DevOps Use Case

A platform team may use Cilium to enforce:

- Only service A can call service B
- Only internal tools can reach admin endpoints
- Only selected namespaces can talk to shared services

That keeps the cluster cleaner and easier to secure.

---

## Policy Layers in Cilium

Cilium can work at different layers:

| Layer | What It Controls | Example |
|------|-------------------|---------|
| L3 | IP-level access | Allow one pod network to another |
| L4 | Port/protocol | Allow TCP 8080 |
| L7 | Application rules | Allow only GET on a path |

### Why This Matters

If you only control L3 and L4, you may still allow too much.

If you also control L7, you can block risky HTTP methods or paths.

---

## Identity-Based Security

Cilium often uses workload identity instead of just raw IPs.

That helps because pod IPs change all the time.

### Example Thinking

Instead of saying:

- Allow traffic from `10.2.1.25`

You say:

- Allow traffic from pods with `app=frontend`

That is much better for Kubernetes automation.

---

## Layered Security Example

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
	name: api-restricted-access
	namespace: production
spec:
	endpointSelector:
		matchLabels:
			app: api
	ingress:
		- fromEndpoints:
				- matchLabels:
						app: frontend
			toPorts:
				- ports:
						- port: "8080"
							protocol: TCP
					rules:
						http:
							- method: "GET"
								path: "/orders"
							- method: "POST"
								path: "/orders"
```

This means:

- Only frontend pods can reach the API
- Only TCP 8080 is allowed
- Only specific HTTP methods and paths are allowed

That is much tighter than a simple open port.

---

## Troubleshooting Cilium Policies

### Problem 1: Traffic Blocked After Policy Is Applied

Check:

```bash
kubectl get ciliumnetworkpolicy -n production
kubectl describe ciliumnetworkpolicy api-restricted-access -n production
kubectl get pods --show-labels -n production
```

Look for:

- Wrong labels
- Wrong namespace
- Missing DNS access
- Missing return path

### Problem 2: Policy Works but App Still Fails

Maybe the app depends on:

- External APIs
- DNS
- Metrics scraping
- Sidecar communication

If you block those by mistake, the app may break even if the main service path is allowed.

### Problem 3: Hard to Understand What Is Happening

Use Cilium visibility tools to inspect traffic paths and policy matches.

That is one of Cilium’s major strengths.

---

## When to Choose Cilium

Choose Cilium when you want:

- Fast, efficient policy enforcement
- Better observability
- Fine-grained traffic rules
- A more advanced security model

For teams running larger Kubernetes platforms on AWS, Cilium can become a very useful part of the security stack.

---

## Summary

Cilium gives you advanced network policy control with eBPF, better visibility, and stronger service-to-service security. It is a good fit when you need more than basic port-based rules.

Use it when your cluster has many services, strict security needs, or a strong need for traffic debugging. Start simple, test carefully, and add Layer 7 rules only where they really help.

Next, Chapter 40 moves to namespace isolation and shows how to keep teams separated inside one cluster.
