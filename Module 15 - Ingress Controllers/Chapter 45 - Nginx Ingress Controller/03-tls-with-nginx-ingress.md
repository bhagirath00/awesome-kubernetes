# TLS With NGINX Ingress

## Bringing TLS Into the Story

You already have routing.

Now you want the traffic to be safe.

That is where TLS comes in.

NGINX Ingress can terminate TLS so users connect with HTTPS.

---

## TLS Termination

TLS termination means the secure connection ends at the ingress controller.

The browser talks HTTPS to NGINX.

NGINX then sends traffic to the backend service, often over internal HTTP.

### Why This Is Useful

- Easier certificate handling
- One place for TLS setup
- Less work for app teams

---

## Basic TLS Ingress Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: shop-ingress
	namespace: production
spec:
	tls:
		- hosts:
				- shop.example.com
			secretName: shop-tls
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

The `secretName` is where the certificate lives.

---

## What Happens During HTTPS

1. Browser connects to NGINX
2. NGINX presents the certificate
3. Browser checks trust
4. Secure session begins
5. Requests go to the backend service

That means the public edge is protected.

---

## Common TLS Choices

| Choice | Meaning | When to Use |
|-------|---------|-------------|
| Termination | TLS ends at NGINX | Most common public apps |
| Re-encryption | NGINX sends TLS to backend | Extra security needs |
| Passthrough | NGINX passes TLS through | Backend manages its own certs |

### Practical Rule

For many beginner projects, termination is enough.

For sensitive internal paths, you may want re-encryption later.

---

## Redirect HTTP to HTTPS

Users may still type `http://` by habit.

You can force them onto HTTPS.

```yaml
metadata:
	annotations:
		nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

That keeps the app safer and avoids mixed behavior.

---

## Where TLS Secrets Come From

The secret can come from:

- Manual creation
- cert-manager
- Another certificate automation workflow

In a real DevOps setup, cert-manager is often the cleaner choice because it renews certs automatically.

---

## Troubleshooting TLS

### Problem 1: Browser Warning

Check the certificate name, expiry, and secret reference.

### Problem 2: Ingress Works on HTTP But Not HTTPS

Check the TLS block and secret name.

### Problem 3: Wrong Hostname

Make sure the certificate SAN matches the host in the Ingress rule.

### Problem 4: Secret Missing

If the secret is missing or in the wrong namespace, TLS fails.

---

## AWS EKS Example

In AWS EKS, TLS is often used with an ingress controller plus a certificate source.

That gives you a clean public endpoint for your app while keeping the backends inside the cluster.

---

## Summary

TLS with NGINX Ingress lets you serve HTTPS at the cluster edge. It protects user traffic, keeps certificates in a Secret, and gives you a clean path for secure public apps.

Next, Chapter 45 moves to rate limiting, which helps you keep one noisy client from hurting the whole app.
