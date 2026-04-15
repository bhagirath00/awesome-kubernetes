# Target Group Binding

## The Last Mile Problem

You have an ALB.

You have a Kubernetes Service.

Now you need the AWS load balancer to know which targets should receive the traffic.

That is what target group binding solves.

---

## What a Target Group Does

An AWS target group is the set of places the ALB can send traffic.

Those targets may be:

- EC2 instances
- Pod IPs
- Nodes

The controller chooses and manages those targets for you.

---

## Instance Mode vs IP Mode

| Mode | Meaning | Good For |
|------|---------|----------|
| instance | Traffic goes to nodes | Simpler setups |
| IP | Traffic goes directly to pod IPs | More direct and often cleaner |

### Story Example

If you send traffic to the node, the node forwards it to the pod.

If you send traffic directly to the pod IP, you skip one layer.

That can make routing feel cleaner.

---

## Binding Flow

1. You define an Ingress
2. The controller creates a target group
3. The controller registers the right targets
4. The ALB sends traffic to the target group
5. Traffic reaches the Service and then the pods

That chain is the real moving part behind the scenes.

---

## Example Target Type Annotation

```yaml
metadata:
	annotations:
		alb.ingress.kubernetes.io/target-type: ip
```

This tells the controller to send traffic to pod IPs.

---

## Why This Matters

If target binding is wrong, your ALB may look healthy but traffic still fails.

That makes it a classic production troubleshooting issue.

---

## Health Checks

Target groups rely on health checks.

If the health check path is wrong, AWS may think healthy pods are dead.

### Example

```yaml
metadata:
	annotations:
		alb.ingress.kubernetes.io/healthcheck-path: /health
```

This tells AWS to check the correct endpoint.

---

## Troubleshooting Target Binding

### Problem 1: Targets Not Registered

Check the controller logs and the Service selector.

### Problem 2: Health Checks Fail

Check the health check path and the backend response code.

### Problem 3: Traffic Hits ALB But Not Pods

Look at security groups, target type, and pod readiness.

---

## AWS EKS Story

You deploy a new version of the app.

The controller updates the target group.

Healthy pods stay in rotation.

Broken pods are removed.

That is how AWS and Kubernetes work together to keep traffic moving.

---

## Summary

Target group binding is the step that connects the ALB to real Kubernetes targets. It controls whether traffic goes to nodes or pod IPs, and it depends on correct health checks and controller settings.

That completes Chapter 46 and closes the module with the full AWS ingress story.
