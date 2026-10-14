# What Is Ingress?

## The Problem Ingress Solves

Imagine you deployed three services in Kubernetes:

- `frontend`
- `api`
- `admin`

Each service has its own pod and its own Service object. That works fine inside the cluster, but now you want the outside world to reach them through one clean entry point.

If you do not use Ingress, you often end up with:

- One load balancer per service
- Too many public endpoints
- Higher cost
- Messy DNS

Ingress gives you one simple front door.

---

## A Simple Story

Think of Kubernetes like a big office building.

- Services are rooms inside the building
- Pods are the people in the rooms
- Ingress is the reception desk at the main door

Instead of giving every visitor a different door number, you send everyone to reception first. Reception then guides them to the correct room.

That is what Ingress does for HTTP and HTTPS traffic.

---

## What Ingress Really Is

Ingress is a Kubernetes object that defines how external traffic reaches services inside the cluster.

It works with an Ingress Controller, which actually performs the routing.

| Part | What It Does |
|------|--------------|
| Ingress resource | Describes the routing rules |
| Ingress Controller | Reads the rules and sends traffic correctly |
| Service | Sends traffic to pods |
| Pod | Runs the application |

Without the controller, the Ingress resource is only a plan.

---

## Why You Should Care in DevOps

Ingress is useful because it helps you:

- Reduce public load balancers
- Keep routing in YAML and Git
- Manage TLS in a standard way
- Route traffic by hostname and path
- Keep your platform easier to operate

That means your team can move faster without building custom networking for every app.

---

## Basic Ingress Flow

```text
User Browser
	 ↓ HTTPS
Ingress Controller
	 ↓
Ingress Rule
	 ↓
Kubernetes Service
	 ↓
Pod
```

The browser asks for `shop.example.com`.

The controller checks the host and path.

Then it forwards the request to the right Service.

---

## Host-Based Routing

You may want different apps on different hostnames.

### Example

| Host | Service |
|------|---------|
| shop.example.com | frontend |
| api.example.com | api |
| admin.example.com | admin |

### Example YAML

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

This says: if traffic comes for `shop.example.com`, send it to the frontend Service.

---

## Path-Based Routing

You may also want different paths on the same host.

### Example

| Path | Service |
|------|---------|
| `/` | frontend |
| `/api` | api |
| `/admin` | admin |

This is helpful when you want one domain for the whole app.

---

## Ingress vs Service

| Feature | Service | Ingress |
|---------|---------|---------|
| Internal access | Yes | No |
| External HTTP access | Limited | Yes |
| Host routing | No | Yes |
| Path routing | No | Yes |
| TLS termination | No | Yes |

### Easy Rule

Use a Service to reach pods inside the cluster.

Use Ingress when you want the outside world to enter through HTTP or HTTPS.

---

## A Simple Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: shop-ingress
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

This is the cleanest starting point for a beginner.

---

## What Happens If You Skip Ingress

Without Ingress, you may end up with:

- Direct service exposure
- More AWS load balancers than needed
- Hard-to-manage DNS
- Poor app separation

That is fine for tiny test clusters, but not for a real platform.

---

## Practical AWS Example

In AWS EKS, Ingress usually sits in front of services and works with either:

- NGINX Ingress Controller
- AWS Load Balancer Controller

You choose the controller based on your app style and your team’s preference.

---

## Summary

Ingress gives Kubernetes a single, clear entry point for HTTP and HTTPS traffic. It helps you route by host and path, keeps your services cleaner, and makes public access much easier to manage.

If you think of it like a receptionist for your cluster, the idea becomes easier to remember. Next, Chapter 44 moves to how Ingress compares with plain Services, so you can see when each one should be used.
