# What Is the AWS Load Balancer Controller?

## The AWS Story

You have a Kubernetes app on EKS.

You want traffic to come in from the internet.

You also want AWS to do the load balancer work for you.

That is where the AWS Load Balancer Controller helps.

It watches Kubernetes Ingress and Service resources and creates AWS load balancers, target groups, and listeners.

---

## Why It Exists

Without the controller, you would need to create and manage AWS load balancers by hand.

That means:

- More clicks in the AWS console
- More manual syncing between AWS and Kubernetes
- More chance of drift

The controller removes that pain by letting Kubernetes control the AWS edge layer.

---

## The Main Job

The controller translates Kubernetes intent into AWS networking objects.

| Kubernetes Object | AWS Result |
|------------------|------------|
| Ingress | ALB and listener rules |
| Service | Target group targets |
| Annotations | AWS-specific settings |

The Kubernetes manifest stays the source of truth.

---

## How It Feels in Real Life

Think of it like this:

- You write an Ingress rule in Git
- The controller sees it
- AWS gets the load balancer details automatically
- Traffic starts flowing

That means your platform team can manage AWS edge routing through Kubernetes YAML.

---

## Why DevOps Teams Use It

This controller helps when you want:

- AWS-managed public entry points
- Path and host routing
- TLS at the load balancer
- Automatic target registration
- Less manual AWS work

It is a strong fit for teams already using EKS and AWS networking.

---

## ALB vs NGINX

| Area | AWS ALB Controller | NGINX Ingress |
|------|--------------------|---------------|
| Cloud integration | Deep AWS integration | More generic |
| Load balancer type | ALB | Usually external LB in front of NGINX |
| Best for | AWS-native setups | Flexible Kubernetes setups |
| AWS features | Strong | Depends on setup |

### Easy Rule

If you want AWS-native edge control, the ALB controller is a very natural fit.

---

## Example Flow

```text
User Browser
	↓ HTTPS
AWS ALB
	↓
AWS Load Balancer Controller
	↓
Kubernetes Ingress
	↓
Service
	↓
Pod
```

The ALB is the front door.

The controller keeps it in sync with Kubernetes.

---

## Common Use Cases

- Public websites
- Internal dashboards
- API gateways
- Multi-path apps
- TLS-enabled front doors

That makes it useful for many AWS workloads.

---

## Summary

The AWS Load Balancer Controller links Kubernetes and AWS so your Ingress rules can become real ALBs, target groups, and listeners without manual work.

It is a very practical tool for EKS teams because it makes AWS edge networking part of the Kubernetes workflow. Next, Chapter 46 moves to installation so you can see how the controller gets into the cluster.
