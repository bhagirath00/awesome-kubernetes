# Rate Limiting

## Why Rate Limiting Matters

Sometimes a request is not a problem.

A flood of requests is the problem.

If one client keeps hammering your API, it can slow down real users or even create an outage.

Rate limiting helps you put a cap on that behavior.

---

## The Story

Imagine a store opening its doors.

One customer buys normally.

Then another person keeps pushing the door fifty times a second.

The store cannot work like that.

Rate limiting is the bouncer at the door.

---

## What Rate Limiting Does

Rate limiting controls how many requests a client can make in a given time.

That helps you:

- Protect backend services
- Stop accidental overload
- Reduce abuse
- Keep the app stable during traffic spikes

---

## NGINX Rate Limit Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: api-ingress
	namespace: production
	annotations:
		nginx.ingress.kubernetes.io/limit-rps: "10"
spec:
	rules:
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

This limits requests per second.

---

## When You Use It

Rate limiting is useful for:

- Public APIs
- Login endpoints
- Upload endpoints
- Shared internal services

It is especially useful when one backend service is expensive to run.

---

## A Practical Example

Suppose your login API is under attack.

The attacker keeps sending requests.

Without rate limiting, the API and database may slow down.

With rate limiting, the controller starts pushing back before the whole system gets tired.

---

## Limit Settings

| Setting | What It Controls |
|--------|-------------------|
| limit-rps | Requests per second |
| limit-connections | Concurrent connections |
| limit-burst | Short bursts above the normal limit |

### Why Burst Matters

A small burst is normal.

You do not want to block all short spikes.

Burst allows quick traffic without letting abuse run wild.

---

## Good Practices

- Start with soft limits
- Watch traffic before tightening too much
- Protect login and expensive endpoints first
- Test with real app behavior

Too much rate limiting can frustrate good users.

---

## Troubleshooting Rate Limits

### Problem 1: Legit Users Get Blocked

Your limit is too low.

### Problem 2: No Effect

The annotation may not match the controller version or the route path.

### Problem 3: Backend Still Overloaded

You may need to combine rate limiting with autoscaling and caching.

---

## AWS EKS Angle

In AWS EKS, rate limiting is a simple but effective way to protect APIs exposed through NGINX.

It does not replace security tools, but it gives you one more layer of defense.

---

## Summary

Rate limiting helps protect your services from too much traffic, whether that traffic is malicious or just badly behaved. NGINX Ingress gives you a practical place to apply those limits.

That finishes Chapter 45 and prepares you for Chapter 46, where AWS ALB Ingress Controller brings the cloud provider into the story.
