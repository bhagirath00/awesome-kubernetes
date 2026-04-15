# Installing NGINX Ingress Controller

## Why NGINX Comes Into the Story

Ingress on its own is only a rulebook.

You still need something in the cluster that reads the rules and sends the traffic.

That something is the NGINX Ingress Controller.

If Ingress is the map, NGINX is the driver.

---

## What NGINX Ingress Controller Does

NGINX Ingress Controller watches Ingress resources and turns them into real traffic behavior.

It can handle:

- Host-based routing
- Path-based routing
- TLS termination
- Redirects
- Rate limiting and other controller settings

### Why DevOps Teams Use It

It is common, well known, and works nicely in many Kubernetes setups.

That makes it a good fit when you want predictable ingress behavior without too much surprise.

---

## How It Usually Gets Installed

Most teams install it with Helm or manifests.

### Helm Style

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
	--namespace ingress-nginx \
	--create-namespace
```

This creates the controller deployment, service, and supporting objects.

---

## The Main Pieces

| Piece | Job |
|------|-----|
| Controller Pod | Reads Ingress rules |
| Service | Exposes the controller |
| ConfigMap | Stores controller settings |
| Ingress Resource | Holds routing rules |

The controller is the part that actually matters at runtime.

---

## Checking the Install

After installation, check the controller pods.

```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

You want to see the controller running and the service exposed.

---

## AWS EKS Example

In AWS EKS, the NGINX controller often sits behind a cloud load balancer.

That means:

- AWS handles the external entry point
- NGINX handles the routing inside Kubernetes

This is a common pattern when teams want simple and flexible ingress rules.

---

## Common Installation Mistakes

### Mistake 1: Wrong Namespace

If the controller is installed but you are looking in the wrong namespace, it looks broken even when it is not.

### Mistake 2: No External IP

The service may take time to receive an external address from the cloud provider.

### Mistake 3: Missing RBAC

The controller needs permission to watch Ingress resources and related objects.

---

## A Small Story

You install the controller on Friday.

On Monday, the app team creates an Ingress rule.

Traffic starts flowing through the controller.

No custom code.

No special routing app.

Just Kubernetes objects and a controller doing the work.

That is the kind of repeatable platform behavior DevOps teams like.

---

## Summary

The NGINX Ingress Controller is the engine that makes Ingress rules real. It reads the rules, exposes the app, and handles common traffic tasks in a very practical way.

Install it once, verify the controller is healthy, and then let your Ingress resources do the routing work. Next, Chapter 45 moves into NGINX configuration so you can see how to tune its behavior for real apps.
