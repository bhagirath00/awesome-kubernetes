# Ingress Annotations

## Why Annotations Exist

Sometimes an Ingress rule is not enough.

You may also need to tell the controller how to behave:

- Redirect HTTP to HTTPS
- Use a specific load balancer class
- Set timeouts
- Enable rewrite rules

That extra behavior is often handled with annotations.

---

## What Annotations Are

Annotations are key-value pairs attached to a Kubernetes object.

They are usually used for controller-specific settings.

### Example

```yaml
metadata:
	annotations:
		nginx.ingress.kubernetes.io/rewrite-target: /
```

That tells the NGINX Ingress Controller to rewrite the path.

---

## Common Annotation Uses

| Use | Example |
|-----|---------|
| HTTPS redirect | Force users onto HTTPS |
| Path rewrite | Change `/app` to `/` before forwarding |
| Timeout control | Set request timeout length |
| Body size limit | Allow bigger uploads |
| Load balancer behavior | Tune cloud-specific settings |

---

## Example: Redirect HTTP to HTTPS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: app-ingress
	namespace: production
	annotations:
		nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
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

This makes sure users go to HTTPS.

---

## Example: Path Rewrite

Sometimes your app expects `/`, but users come in through `/shop`.

Annotations can help rewrite the path.

```yaml
metadata:
	annotations:
		nginx.ingress.kubernetes.io/rewrite-target: /
```

That can make legacy apps easier to expose without changing the app code.

---

## AWS and Controller-Specific Settings

Different controllers use different annotation names.

That means you should always check the controller docs before adding them.

### Story Example

If your team switches from NGINX to AWS ALB later, the annotation style may change.

That is normal.

The Ingress idea stays the same, but the controller-specific knobs are different.

---

## Good Practices

- Keep annotations simple
- Use them only when the controller needs extra behavior
- Document what each one does
- Test in staging before production

Too many annotations can make the manifest hard to read.

---

## Troubleshooting Annotations

### Problem 1: Annotation Ignored

Check whether the annotation belongs to your controller.

### Problem 2: Wrong Rewrite Behavior

Look at the path and verify what the backend expects.

### Problem 3: HTTPS Redirect Loop

This often happens if the redirect and TLS setup are not aligned.

---

## Summary

Annotations give Ingress controllers extra instructions for behavior that goes beyond basic routing. They are useful for redirects, rewrites, and tuning controller settings.

Use them carefully, because different controllers understand different annotations. Next, Chapter 45 moves into NGINX Ingress Controller, where these ideas become a real deployment pattern.
