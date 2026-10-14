# Ingress vs Service

## A Simple Comparison Story

You have an app in Kubernetes, and you want people to reach it.

At first, you create a Service.

That works, but then you notice the app needs hostname routing, TLS, and a single external entry point.

Now Ingress starts to make more sense.

---

## What a Service Does

A Kubernetes Service gives a stable way to reach pods.

It handles:

- Pod discovery
- Stable virtual IPs
- Load balancing inside the cluster

### Service Example

```yaml
apiVersion: v1
kind: Service
metadata:
	name: frontend
	namespace: production
spec:
	selector:
		app: frontend
	ports:
		- port: 80
			targetPort: 8080
	type: ClusterIP
```

This Service is great for internal communication.

---

## What Ingress Does

Ingress sits above Services.

It decides which Service gets the traffic based on:

- Hostname
- Path
- TLS settings

That makes it useful for public web traffic.

### Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: app-ingress
	namespace: production
spec:
	rules:
		- host: shop.example.com
			http:
				paths:
					- path: /
						pathType: Prefix
						backend:
							service:
								name: frontend
								port:
									number: 80
```

---

## The Main Difference

| Feature | Service | Ingress |
|---------|---------|---------|
| Internal pod access | Yes | No |
| Stable internal address | Yes | No |
| External web entry point | Limited | Yes |
| Host routing | No | Yes |
| Path routing | No | Yes |
| TLS termination | No | Yes |

### Easy Memory Trick

- Service = reach the app
- Ingress = decide how outside traffic enters the app

---

## When You Use Only a Service

Use only a Service when:

- The app is internal only
- Pods talk only to each other
- You do not need hostname routing
- You are still testing locally

Example:

- A database service
- A queue service
- An internal metrics collector

---

## When You Add Ingress

Add Ingress when:

- You need HTTPS
- You want one public entry point
- You want to route by host or path
- You want to keep AWS load balancers under control

This is common for websites, APIs, and dashboards.

---

## AWS Example

Imagine two choices in AWS EKS:

### Choice 1: Many Public Services

- One public load balancer for frontend
- One public load balancer for API
- One public load balancer for admin

This is expensive and harder to manage.

### Choice 2: One Ingress Front Door

- One Ingress Controller
- One public endpoint
- Routes to many Services

This is cleaner and usually cheaper.

---

## Service Types Still Matter

Ingress usually sends traffic to a `ClusterIP` Service.

That Service then sends traffic to pods.

So Ingress does not replace Services.

It depends on them.

---

## Troubleshooting Difference

### If Service Fails

Check:

- Selector labels
- Target port
- Pod readiness

### If Ingress Fails

Check:

- Ingress rule
- Hostname
- Path
- Ingress controller status

---

## Practical Rule for Beginners

If you need internal traffic, think Service.

If you need external web traffic, think Ingress.

If you need both, use a Service behind an Ingress.

---

## Summary

Services give stable internal access to pods, while Ingress gives the cluster a smart public entry point. You usually need both in a real app: Service for routing inside the cluster, Ingress for traffic coming from outside.

Next, Chapter 44 moves to Ingress rules so you can see how hostname and path routing actually work in practice.
