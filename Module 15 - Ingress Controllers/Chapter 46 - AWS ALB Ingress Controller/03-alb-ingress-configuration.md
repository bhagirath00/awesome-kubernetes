# ALB Ingress Configuration

## Why Configuration Matters

An AWS ALB does not magically know what you want.

You tell it through an Ingress resource and annotations.

That configuration decides things like:

- Which subnets to use
- Whether the ALB is internet-facing
- Which certificate to use
- How traffic should route

---

## The Basic Ingress Shape

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: shop-alb-ingress
	namespace: production
	annotations:
		kubernetes.io/ingress.class: alb
		alb.ingress.kubernetes.io/scheme: internet-facing
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

This tells the controller to create an AWS ALB and route traffic to the Service.

---

## Important Annotations

| Annotation | Meaning |
|-----------|---------|
| alb.ingress.kubernetes.io/scheme | Public or internal load balancer |
| alb.ingress.kubernetes.io/target-type | Instance or IP targets |
| alb.ingress.kubernetes.io/certificate-arn | TLS certificate for HTTPS |
| alb.ingress.kubernetes.io/listen-ports | Which ports ALB should listen on |

### Story Example

You want the app public.

You set the scheme to `internet-facing`.

You want HTTPS.

You add a certificate ARN.

Now AWS creates the right edge behavior for you.

---

## Public vs Internal ALB

| Type | Meaning | Use Case |
|------|---------|---------|
| internet-facing | Public on the internet | Websites, APIs |
| internal | Only inside the VPC | Private apps, internal tools |

Choosing the wrong one can expose an app that should stay private.

---

## Routing With Host and Path

You can route by hostname and by path just like other Ingress controllers.

### Example

```yaml
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
```

That lets one ALB serve multiple app parts.

---

## TLS With ALB

You can attach an AWS certificate to the ALB.

That is useful when you want AWS to manage public TLS at the load balancer layer.

### Example Annotation

```yaml
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/12345678
```

This is a very common production setup in AWS.

---

## Common Configuration Mistakes

### Mistake 1: Wrong Ingress Class

If the annotation does not point to the AWS controller, the ALB may never appear.

### Mistake 2: Wrong Certificate ARN

If the certificate ARN is wrong or expired, HTTPS fails.

### Mistake 3: Wrong Scheme

If you need a private service but set `internet-facing`, you may expose too much.

### Mistake 4: Wrong Subnets

AWS needs the right subnets in the cluster VPC.

---

## AWS EKS Angle

This controller fits AWS very well because it speaks the same language as the cloud.

That means less glue work for platform teams.

---

## Summary

ALB Ingress configuration is where Kubernetes intent becomes AWS edge behavior. The annotations tell AWS whether the load balancer should be public or private, what certificate to use, and how to route traffic.

Next, Chapter 46 moves to target group binding so you can see how traffic gets from the ALB to the right pods.
