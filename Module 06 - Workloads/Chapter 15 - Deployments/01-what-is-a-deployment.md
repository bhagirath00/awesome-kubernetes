# What is a Deployment

## Introduction

A **Deployment** is the standard Kubernetes object for running stateless application replicas. It manages a fleet of identical Pods and ensures the desired number of replicas is always running — even if nodes fail, Pods crash, or you update the application image.

Deployments are the most-used workload object in Kubernetes. Your web servers, REST APIs, gRPC services, background workers, and microservices all run as Deployments.

---

## The Problem Deployments Solve

Without Deployments, you would create Pods manually:

```bash
kubectl run web-pod --image=nginx:1.25
```

This creates one Pod. Problems:
- If the Pod crashes, nobody recreates it
- You cannot easily scale to 5 Pods without 5 separate commands
- Updating the image means deleting the old Pod and creating a new one — downtime
- No rollback if the new version is broken

Deployments solve all of these:

```
Deployment "web-app" (desired replicas: 3)
    │
    ▼
ReplicaSet (manages Pod copies)
    │
    ├─ Pod: web-pod-abc123 (Running)
    ├─ Pod: web-pod-def456 (Running)
    └─ Pod: web-pod-ghi789 (Running)

If a Pod crashes → Deployment detects it → creates a replacement
If you change the image → Deployment creates new Pods with new image → removes old ones
If the new version breaks → Deployment can rollback to the previous ReplicaSet
```

---

## The Relationship: Deployment → ReplicaSet → Pods

A Deployment does not directly manage Pods. It creates and manages **ReplicaSets**, which in turn manage Pods:

```
Deployment (desired state: 3 replicas of web-app:v2.0)
    │
    ├─ ReplicaSet v1 (web-app:v1.0) — old version, scaled to 0
    └─ ReplicaSet v2 (web-app:v2.0) — current version, scaled to 3
           ├─ Pod: web-pod-abc (web-app:v2.0)
           ├─ Pod: web-pod-def (web-app:v2.0)
           └─ Pod: web-pod-ghi (web-app:v2.0)
```

When you update the image, the Deployment creates a new ReplicaSet (v3) and gradually shifts replicas from the old ReplicaSet to the new one. Old ReplicaSets are kept (scaled to 0) for rollback.

---

## Anatomy of a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app                    # Deployment name
  namespace: production
  labels:
    app: web-app
    environment: production

spec:
  replicas: 3                      # desired number of Pod copies
  
  # How the Deployment finds "its" Pods (and ReplicaSets)
  selector:
    matchLabels:
      app: web-app                 # MUST match template.metadata.labels
  
  # Update strategy (covered in 03-rolling-updates.md)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # max extra Pods during update
      maxUnavailable: 0            # min Pods that must stay available
  
  # How many old ReplicaSets to keep for rollback history
  revisionHistoryLimit: 10
  
  # How long to wait before considering a deployment failed
  progressDeadlineSeconds: 600
  
  # Pod template — all Pods are created from this
  template:
    metadata:
      labels:
        app: web-app               # MUST match spec.selector.matchLabels
        version: "2.1.0"           # additional labels for tracking
        environment: production
    
    spec:
      containers:
      - name: web
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/web-app:v2.1.0
        ports:
        - containerPort: 8080
          name: http
        
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        
        readinessProbe:
          httpGet:
            path: /healthz/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        
        livenessProbe:
          httpGet:
            path: /healthz/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]    # drain connections before SIGTERM
        
        env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-password
      
      # Spread Pods across AZs for HA
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-app
      
      terminationGracePeriodSeconds: 30    # 30s for graceful shutdown
```

---

## The selector Field — Critical and Immutable

The `spec.selector` is the most important field to get right. It defines which Pods belong to this Deployment. Once set, **it cannot be changed**.

```yaml
selector:
  matchLabels:
    app: web-app    # MUST exactly match template.metadata.labels

template:
  metadata:
    labels:
      app: web-app  # MUST match selector above
```

If the selector and template labels do not match, the Deployment creates Pods but cannot find them — they appear as unmanaged orphans.

---

## Deployment Conditions and Status

```bash
# Detailed status of a Deployment
kubectl get deployment web-app -n production

# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# web-app   3/3     3            3           5d
# READY: current/desired replicas
# UP-TO-DATE: replicas updated to latest desired state
# AVAILABLE: replicas passing readiness probes

# Describe for detailed conditions
kubectl describe deployment web-app -n production

# Conditions:
#   Type           Status  Reason
#   ----           ------  ------
#   Available      True    MinimumReplicasAvailable
#   Progressing    True    NewReplicaSetAvailable

# Watch a rollout
kubectl rollout status deployment/web-app -n production
# Waiting for deployment "web-app" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "web-app" rollout to finish: 1 old replicas are pending termination...
# deployment "web-app" successfully rolled out
```

---

## Deployment Use Cases — When to Use a Deployment

| Application Type | Use Deployment? |
|-----------------|----------------|
| REST API server | ✅ Yes |
| Web server (Nginx, Apache) | ✅ Yes |
| Background worker (no state) | ✅ Yes |
| ML inference server | ✅ Yes |
| Database (PostgreSQL, MySQL) | ❌ No — use StatefulSet |
| Message queue (Kafka, RabbitMQ) | ❌ No — use StatefulSet |
| Node-level daemon (log collector, monitor) | ❌ No — use DaemonSet |
| Batch job (run once and exit) | ❌ No — use Job |
| Scheduled task (run at time intervals) | ❌ No — use CronJob |

Deployments are for **stateless workloads** — applications where any replica can serve any request and Pods have no individual identity.

---

## Summary

- Deployments are the standard workload for stateless application replicas — they ensure the desired number of Pods is always running
- Deployments manage ReplicaSets which manage Pods — old ReplicaSets are retained for rollback
- The `selector.matchLabels` must exactly match `template.metadata.labels` — this link is immutable once set
- `replicas`, `strategy`, and `revisionHistoryLimit` are the three key configuration fields beyond the Pod template
- Use Deployments for stateless services (APIs, web servers, workers); use StatefulSets for databases, DaemonSets for node agents, Jobs for batch tasks

In the next file, `02-creating-deployments.md`, you will learn how to create, inspect, and manage Deployments — with production-grade patterns for health checks, affinity, and resource management.
