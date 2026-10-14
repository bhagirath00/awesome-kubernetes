# What Is a Service Mesh?

## Why You Need a Service Mesh

When you run microservices in Kubernetes, each service talks to many other services.

At first, you may manage that with simple Kubernetes Services and network policies. That works for small systems. But as the number of services grows, the network layer becomes hard to manage.

A service mesh solves that problem by moving common service-to-service features into a dedicated layer.

It helps you manage:

- Traffic routing
- Retries
- Timeouts
- Security between services
- Observability

### Simple Example

Think about an order system:

- Frontend calls API
- API calls payment service
- API calls inventory service
- Payment service calls a fraud check service

Without a service mesh, each app team has to build all of this behavior into its own code.

With a service mesh, the platform layer can handle a lot of that work for you.

---

## What a Service Mesh Actually Is

A service mesh is a platform layer that manages communication between services.

It does not replace Kubernetes.

It sits on top of Kubernetes and gives you better control over traffic between pods.

### Main Parts

| Part | Job |
|------|-----|
| Data plane | Moves the traffic and applies rules |
| Control plane | Stores config and pushes rules |
| Sidecar or proxy | Sits beside each app and handles traffic |

The most common model is the sidecar model.

Each app pod gets a proxy container next to it. That proxy sees the traffic in and out of the pod.

---

## Why Developers Like It

Developers do not want to write the same networking code in every service.

They want their application code to focus on business logic.

Service mesh helps by handling common transport tasks in one place.

### Common Tasks the Mesh Can Help With

- Automatic retries when a service fails briefly
- Short timeouts when a request is too slow
- Circuit breaking when a service is unhealthy
- Mutual TLS for encrypted service-to-service traffic
- Routing canary traffic to a new version

---

## Sidecar Pattern

The sidecar is a small proxy container that runs beside your app container.

### Pod Example

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: orders-api
	namespace: production
spec:
	containers:
		- name: app
			image: myorg/orders-api:1.0
			ports:
				- containerPort: 8080
		- name: proxy
			image: mesh-proxy:latest
			args: ["--mode=sidecar"]
```

The app container handles business logic.

The proxy container handles network logic.

### Why This Helps

If you need to change traffic policy, you change the mesh config instead of changing application code.

That is a cleaner design for DevOps teams.

---

## What Problems It Solves

### Problem 1: Unreliable Calls

One service may fail for a short time.

The mesh can retry the request and recover without the app knowing all the transport details.

### Problem 2: Too Much Trust

In many clusters, services trust every other service too much.

The mesh can use encryption and identity to make that safer.

### Problem 3: Hard Debugging

If traffic fails in a microservice app, it can be hard to know where the failure happened.

The mesh can give you metrics, traces, and traffic logs.

---

## Service Mesh vs Network Policy

They are not the same thing.

| Feature | Network Policy | Service Mesh |
|---------|----------------|--------------|
| Blocks traffic | Yes | Yes, with proxies |
| Encrypts traffic | Not usually | Yes, often with mTLS |
| Observability | Limited | Strong |
| Traffic routing | Limited | Strong |
| Retries/timeouts | No | Yes |
| App-level control | No | Yes |

### Simple Rule

- Use network policy to control who can talk
- Use service mesh to control how they talk

---

## AWS EKS Use Case

In AWS EKS, a service mesh helps when you have:

- Many internal microservices
- Strict security needs
- Canary deployments
- Performance debugging needs
- Cross-team service communication

For example, a payments team may want to send only 5 percent of traffic to a new version before full rollout.

The mesh can do that without code changes.

---

## When Not to Use a Service Mesh

A service mesh is useful, but it is not free.

Avoid it when:

- Your app is very small
- You only have a few services
- You do not need retries or advanced routing
- Your team is not ready for extra operational work

It adds complexity, so you should use it for a clear reason.

---

## Summary

A service mesh is a platform layer that helps manage service-to-service traffic in Kubernetes. It gives you better routing, security, and observability than plain networking alone.

It is useful when your microservices are growing and you want to keep traffic rules out of application code. Start small, understand the proxy model, and use it only when the added control is worth the extra complexity.

Next, Chapter 41 moves to Istio and shows how one of the most common service mesh platforms works in practice.
