# Istio Overview

## What Istio Is

Istio is a service mesh platform for Kubernetes.

It helps you control how services communicate, how traffic is routed, and how service identity is handled.

Istio is popular because it gives teams a lot of power, but it also needs careful setup.

### What Istio Adds

- Traffic control
- Security with mutual TLS
- Observability for services
- Policy enforcement
- Canary and progressive delivery

---

## Istio Architecture

Istio usually has two big parts:

| Part | Job |
|------|-----|
| Control plane | Stores configuration and sends rules |
| Data plane | Proxies traffic for each service |

The control plane tells the proxies what to do.

The data plane applies the rules while traffic moves.

### Common Components

- `istiod` for control plane logic
- Envoy proxy for traffic handling
- Sidecar injection to add the proxy into pods

---

## Sidecar Injection

Istio often uses automatic sidecar injection.

That means when a pod starts, Istio adds a proxy container to the pod.

### Why It Matters

You do not want every team to manually add mesh proxies to every deployment.

Automation keeps the setup repeatable.

### Example Namespace Label

```bash
kubectl label namespace production istio-injection=enabled
```

Now new pods in that namespace can get the sidecar automatically.

---

## Traffic Management

Istio lets you control how traffic moves between versions of a service.

That is very useful for DevOps teams doing safe releases.

### Canary Release Example

You have:

- `orders-v1`
- `orders-v2`

You want 90 percent of traffic to go to v1 and 10 percent to go to v2.

Istio can do that without changing the app code.

### Example Destination Rule

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
	name: orders-dr
spec:
	host: orders.production.svc.cluster.local
	trafficPolicy:
		loadBalancer:
			simple: ROUND_ROBIN
```

That gives Istio a rule for how to send traffic to a service.

---

## Virtual Services

A VirtualService defines traffic routing rules.

### Example

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
	name: orders-vs
spec:
	hosts:
		- orders.production.svc.cluster.local
	http:
		- route:
				- destination:
						host: orders
						subset: v1
					weight: 90
				- destination:
						host: orders
						subset: v2
					weight: 10
```

This means:

- Most traffic goes to v1
- A small part goes to v2

That is useful when you want to test a new version safely.

---

## Security With mTLS

Istio can use mutual TLS, which means both sides of the connection prove who they are.

That helps protect internal traffic.

### Why mTLS Is Useful

- Traffic is encrypted
- Services have identity
- Less chance of fake internal traffic
- Better compliance story

### Real Example

If one pod tries to pretend it is the payment service, mTLS can stop it unless it has the right identity.

---

## Observability

Istio gives you better visibility into service traffic.

You can see:

- Request rate
- Error rate
- Latency
- Traffic source and destination

That helps during incidents.

### DevOps Value

If latency rises after a deployment, Istio metrics help you see whether the problem is in the app, the network, or the traffic policy.

---

## Istio on AWS EKS

Istio works well on EKS when you need:

- Strong traffic control
- Zero-trust communication
- Canary deploys
- Better service metrics

It is often used by platform teams that need a standard service communication layer.

### Example Scenario

A retail company may use Istio to:

- Send 5 percent of checkout traffic to a new release
- Encrypt all internal service calls
- Block traffic from unknown services

That reduces risk during deployments.

---

## Operational Tradeoffs

Istio is powerful, but it has a learning curve.

### Benefits

- Rich routing rules
- Strong security
- Good visibility
- Mature ecosystem

### Costs

- More moving parts
- More resources
- More setup work
- More complexity for small teams

You should adopt it when the value is clear, not just because it is popular.

---

## Troubleshooting Istio

### Problem 1: Pods Do Not Get a Sidecar

Check:

```bash
kubectl get namespace --show-labels
kubectl get pods -n production
```

Possible causes:

- Injection label missing
- Namespace excluded
- Admission webhook problem

### Problem 2: Traffic Broken After Policy Change

Check:

```bash
kubectl get virtualservice
kubectl get destinationrule
kubectl describe virtualservice orders-vs
```

Look for:

- Wrong host name
- Wrong subset
- Route weight mistake

### Problem 3: mTLS Causes Failures

Sometimes older services are not ready for mTLS.

You may need to enable it gradually and test each service.

---

## Summary

Istio is a service mesh platform that gives you traffic control, mTLS security, and service observability. It is a strong choice for larger AWS EKS environments that need canary releases and fine-grained routing.

Use it when you need more than simple Kubernetes networking. Keep the setup automated, test sidecar injection carefully, and roll out traffic rules slowly so you do not break production.

Next, Chapter 41 moves to Linkerd and shows a simpler service mesh option.
