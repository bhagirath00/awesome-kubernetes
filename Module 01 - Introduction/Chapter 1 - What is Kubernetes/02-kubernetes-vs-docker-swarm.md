# Kubernetes vs Docker Swarm

## Introduction

When container orchestration became a real need around 2015–2016, two tools competed to solve the same problem: **Docker Swarm** and **Kubernetes**. Docker Swarm was built by the Docker team and came built into Docker itself. Kubernetes came from Google and required a separate setup.

For a period, both were actively used. Today, Kubernetes has clearly won — but understanding *why* makes you a better engineer. Knowing what Docker Swarm is, where it still makes sense, and exactly where Kubernetes outperforms it will sharpen your understanding of both tools.

---

## What is Docker Swarm

Docker Swarm is Docker's built-in container orchestration feature. When you have multiple Docker hosts and you want to run containers across them, you can connect those hosts into a **Swarm** — a cluster managed by Docker itself.

The key advantage of Docker Swarm is that it requires no additional installation. If you have Docker, you have Swarm. You enable it with one command:

```bash
# Initialize a Swarm on the first machine (becomes the manager node)
docker swarm init --advertise-addr 192.168.1.10

# Output:
# Swarm initialized: current node (abc123) is now a manager.
# To add a worker to this swarm, run the following command:
#
#     docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377
```

Then you add worker nodes:

```bash
# On each worker machine:
docker swarm join --token SWMTKN-1-xxx 192.168.1.10:2377
```

And you deploy services across the swarm:

```bash
# Run 3 replicas of nginx across the swarm
docker service create \
  --name web \
  --replicas 3 \
  --publish 80:80 \
  nginx:1.25
```

Docker Swarm handles scheduling those 3 containers across your nodes, restarts failed containers, and load-balances traffic across them. For simple use cases, this works well.

---

## What is Kubernetes (Recap in This Context)

Kubernetes does everything Docker Swarm does — and significantly more. It requires more setup, has a steeper learning curve, but provides capabilities that Docker Swarm simply does not have.

The equivalent Kubernetes deployment:

```yaml
# kubernetes-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "250m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

More verbose — but notice what the Kubernetes version has that the Swarm version does not: resource limits, readiness probes, namespaces. These are not optional extras — they are critical for running production workloads safely.

---

## Side-by-Side Feature Comparison

| Feature | Docker Swarm | Kubernetes |
|---------|-------------|------------|
| **Setup complexity** | Very simple — built into Docker | Complex — multiple components |
| **Learning curve** | Low — familiar Docker CLI | High — new concepts and APIs |
| **Scaling** | Manual or basic auto-scaling | Horizontal Pod Autoscaler (CPU/memory/custom) |
| **Rolling updates** | Basic | Advanced — with health checks and automatic rollback |
| **Self-healing** | Restarts failed containers | Restarts + reschedules + replaces unhealthy nodes |
| **Load balancing** | Built-in, basic (L4 only) | Built-in L4 + Ingress for L7 (HTTP routing) |
| **Resource limits** | Basic | Fine-grained CPU/memory requests and limits per container |
| **Storage** | Basic volume support | PersistentVolumes, StorageClasses, dynamic provisioning |
| **Secrets management** | Docker Secrets | Kubernetes Secrets + integration with Vault, AWS Secrets Manager |
| **Networking** | Overlay networks | CNI plugins (Calico, Flannel, Cilium) — highly configurable |
| **Health checks** | Basic health checks | Liveness probes, readiness probes, startup probes |
| **RBAC** | Basic | Full role-based access control |
| **Namespaces** | No true namespaces | Full namespace isolation |
| **Custom resources** | No | Custom Resource Definitions (CRDs) — extend Kubernetes with your own objects |
| **Ecosystem** | Small | Enormous — Helm, Istio, Argo CD, Prometheus, and thousands more |
| **Cloud managed service** | No | AWS EKS, GKE, AKS — fully managed control planes |
| **Community size** | Small, declining | Massive, growing |
| **CNCF project** | No | Yes — graduated project |

---

## Where Docker Swarm Wins

Docker Swarm is not a failed tool. It is genuinely better than Kubernetes in specific situations:

### Small Teams with Simple Needs
If you have a small application with 5–10 containers and a team of 1–3 engineers, the operational overhead of Kubernetes is not worth it. Docker Swarm gives you 80% of what you need with 20% of the complexity.

### Rapid Prototyping
Standing up a Swarm takes minutes. Standing up a production-grade Kubernetes cluster takes hours (or days if you are doing it manually). When you need container orchestration fast, Swarm wins on speed.

### Legacy Docker Compose Environments
Docker Swarm can deploy Docker Compose files directly with `docker stack deploy`. If your team already uses Docker Compose for development, the transition to Swarm is nearly zero effort:

```bash
# Deploy a docker-compose.yml directly to Swarm
docker stack deploy -c docker-compose.yml myapp
```

No YAML translation needed. No new concepts. This is a real advantage.

### Teams Not on a Major Cloud
If you are running on bare metal or in an environment without a managed Kubernetes service, Swarm is much simpler to operate and maintain.

---

## Where Kubernetes Wins (and Why it Won)

### Auto-Scaling Based on Real Metrics
Kubernetes has the **Horizontal Pod Autoscaler (HPA)** — it watches CPU usage, memory usage, or custom metrics and automatically scales Pods up or down:

```yaml
# Automatically scale between 2 and 20 replicas based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

When CPU across your Pods exceeds 70%, Kubernetes adds more Pods automatically. When traffic drops, it scales back down. Docker Swarm has no equivalent.

### True HTTP Load Balancing with Ingress
Docker Swarm only does Layer 4 (TCP) load balancing. Kubernetes supports **Ingress** controllers that do Layer 7 (HTTP) routing:

```yaml
# Route traffic to different services based on URL path
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

This routes `/api` requests to one service and all other requests to another — all in Kubernetes, no extra load balancer config needed.

### Sophisticated Health Checking
Kubernetes has three types of health probes:

```yaml
containers:
- name: web
  image: myapp:1.0
  
  # Liveness probe: is the container alive? If not, restart it.
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
    failureThreshold: 3
  
  # Readiness probe: is the container ready to serve traffic? If not, remove from load balancer.
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5
  
  # Startup probe: give the container extra time to start before liveness checks begin.
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30
    periodSeconds: 10
```

Docker Swarm only has a basic health check. Kubernetes has three separate probes, each with different retry and timing behavior. This makes a huge difference in production — your load balancer never sends traffic to a Pod that is not ready.

### AWS EKS — Managed Control Plane
When you use Kubernetes on AWS, Amazon runs the entire control plane for you. You pay a small hourly fee ($0.10/hour per cluster) and AWS handles:
- High availability of the API server
- etcd backups and replication
- Automatic Kubernetes version upgrades
- Security patches for control plane components

There is no equivalent managed service for Docker Swarm from any major cloud provider.

---

## Why Kubernetes Won the Orchestration War

In 2017, Docker Swarm was a serious competitor. By 2019, the contest was effectively over. Here is what happened:

1. **Google contributed Kubernetes to the CNCF** — it immediately had the backing of the entire cloud native ecosystem
2. **Every major cloud launched a managed K8s service** — AWS EKS (2018), GKE (2014), AKS (2017). No one launched a managed Swarm service
3. **The ecosystem exploded** — Helm, Istio, Prometheus, Argo CD, Fluentd all became Kubernetes-native tools
4. **Docker Inc itself switched** — In 2019, Docker Desktop started shipping Kubernetes as a first-class option alongside Swarm
5. **Enterprise adoption** — Large companies needed the advanced features (RBAC, namespaces, network policies) that only Kubernetes provided

Today, Docker Swarm is in maintenance mode. Docker Inc is not actively developing new features for it.

---

## When to Use Each in 2024

| Scenario | Use Docker Swarm | Use Kubernetes |
|----------|-----------------|---------------|
| 5–10 containers, small team | ✅ | Overkill |
| 50+ containers, multiple services | ❌ Too limited | ✅ |
| AWS production workload | ❌ No managed service | ✅ EKS |
| Need auto-scaling | ❌ | ✅ HPA |
| Need HTTP routing (Ingress) | ❌ | ✅ |
| Quick internal tool | ✅ | Slower setup |
| DevOps job requirement | ❌ Rarely asked | ✅ Standard requirement |
| Microservices architecture | ❌ | ✅ |
| Legacy Docker Compose migration | ✅ Easiest path | More effort |

---

## Summary

- Docker Swarm is Docker's built-in orchestration, simple to set up but limited in capability
- Kubernetes is more complex to learn but provides auto-scaling, advanced health checks, Ingress, RBAC, namespaces, and a massive ecosystem
- Docker Swarm wins for small teams, simple apps, and quick setups where operational simplicity matters more than features
- Kubernetes wins for production workloads, microservices, cloud deployments, and anywhere auto-scaling and advanced routing are needed
- Kubernetes has won the orchestration market — it is the industry standard and what every DevOps role expects you to know
- AWS EKS provides a fully managed Kubernetes control plane — this is what you will use in this course

In the next file, `03-kubernetes-in-devops.md`, you will see exactly where Kubernetes fits inside a complete DevOps pipeline — from code commit to production deployment.
