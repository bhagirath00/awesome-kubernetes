# Ingress Rules

## Why Rules Matter

An Ingress resource is only useful if it knows where to send traffic.

That routing logic lives in Ingress rules.

The rules are what turn a generic front door into a useful traffic planner.

---

## The Two Main Rule Types

| Rule Type | What It Matches |
|----------|------------------|
| Host-based | Matches the domain name |
| Path-based | Matches the URL path |

You can use one or both.

---

## Host-Based Routing Example

Suppose your app has two domains:

- `shop.example.com`
- `api.example.com`

You can route each one to a different Service.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: host-routing
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
		- host: api.example.com
			http:
				paths:
					- path: /
						pathType: Prefix
						backend:
							service:
								name: api
								port:
									number: 80
```

---

## Path-Based Routing Example

Now imagine one domain but several paths:

- `/` goes to frontend
- `/api` goes to API
- `/admin` goes to admin

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: path-routing
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
					- path: /api
						pathType: Prefix
						backend:
							service:
								name: api
								port:
									number: 80
					- path: /admin
						pathType: Prefix
						backend:
							service:
								name: admin
								port:
									number: 80
```

This is a common setup for web apps.

---

## How the Controller Uses Rules

The Ingress Controller reads the Ingress object and turns the rules into real network behavior.

So the rule itself is not magic.

The controller is the part doing the work.

### Story Version

You tell the receptionist:

> If someone says `shop.example.com`, send them to the frontend room.

The receptionist remembers that instruction and acts on it when people arrive.

---

## Path Types

Kubernetes supports path types like `Prefix`.

That means a rule can match more than one exact path.

For example:

- `/api` matches `/api/users`
- `/admin` matches `/admin/settings`

This gives you more flexible routing.

---

## Common Rule Mistakes

### Mistake 1: Wrong Hostname

If DNS points to the wrong name, the rule will not catch the traffic.

### Mistake 2: Wrong Path Order

If a broad path matches before a narrow one, traffic may go to the wrong backend.

### Mistake 3: Wrong Service Port

If the Service listens on a different port, the request fails.

### Mistake 4: Missing Ingress Controller

The rules mean nothing if the controller is not installed and running.

---

## AWS Example

In AWS EKS, rules are often used to:

- Route traffic from one domain to multiple services
- Keep admin paths separate
- Split public and internal traffic

That makes one cluster feel much more organized.

---

## Summary

Ingress rules decide where incoming traffic goes based on hostnames and paths. They are the part that turns a generic Ingress into a real router for your app.

Next, Chapter 44 moves to annotations, which are often the extra settings controllers need for TLS, redirects, and cloud-specific behavior.
