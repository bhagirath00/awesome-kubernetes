# Creating Deployments

## Introduction

Creating a Deployment correctly from the start means setting the right resource requests, health checks, and update parameters — not just the container image. A Deployment without proper readiness probes and resource limits is a production incident waiting to happen. This file covers how to create production-ready Deployments from scratch.

---

## Creating Declaratively — YAML (Recommended)

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
  revisionHistoryLimit: 5     # keep last 5 ReplicaSets for rollback
  progressDeadlineSeconds: 300
  
  template:
    metadata:
      labels:
        app: api-server
        version: "3.0.1"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: api-server-sa    # dedicated ServiceAccount
      
      containers:
      - name: api
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:v3.0.1
        imagePullPolicy: IfNotPresent      # pull from ECR if not on node
        
        ports:
        - name: http
          containerPort: 8080
        - name: metrics
          containerPort: 9090
        
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        
        # Readiness: don't send traffic until the app is ready
        readinessProbe:
          httpGet:
            path: /healthz/ready
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
          successThreshold: 1
          failureThreshold: 3
          timeoutSeconds: 3
        
        # Liveness: restart if the app is stuck/dead
        livenessProbe:
          httpGet:
            path: /healthz/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
          timeoutSeconds: 5
        
        # Startup: give slow-starting apps time to initialize
        startupProbe:
          httpGet:
            path: /healthz/live
            port: 8080
          failureThreshold: 30      # 30 × 10s = 5 minutes max startup time
          periodSeconds: 10
        
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]  # drain in-flight requests
        
        envFrom:
        - configMapRef:
            name: api-config
        - secretRef:
            name: api-secrets
        
        securityContext:
          runAsNonRoot: true           # do not run as root
          runAsUser: 1000
          readOnlyRootFilesystem: true # immutable container filesystem
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]             # drop all Linux capabilities
      
      # Security context at Pod level
      securityContext:
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      terminationGracePeriodSeconds: 30
      
      # Spread across AZs
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api-server
      
      # Anti-affinity — prefer different nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels:
                  app: api-server
```

```bash
kubectl apply -f deployment.yaml

kubectl get deployment api-server -n production
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# api-server   3/3     3            3           30s
```

---

## Creating Imperatively — Quick Debugging

```bash
# Create a deployment quickly (not for production)
kubectl create deployment web-test --image=nginx:1.25 --replicas=2 -n production

# Generate YAML from the imperative command (useful for templates)
kubectl create deployment web-test \
  --image=nginx:1.25 \
  --replicas=2 \
  --dry-run=client \
  -o yaml > deployment-template.yaml

cat deployment-template.yaml
# Then edit the file and add probes, resources, etc.
```

---

## Scaling a Deployment

```bash
# Imperative scale (not tracked in Git)
kubectl scale deployment api-server --replicas=5 -n production

# Declarative scale (update YAML and apply)
# Edit deployment.yaml: spec.replicas: 5
kubectl apply -f deployment.yaml -n production

# Scale to zero (stops all Pods but keeps the Deployment)
kubectl scale deployment api-server --replicas=0 -n production
# Use case: pause a service for maintenance without deleting it

# Scale back up
kubectl scale deployment api-server --replicas=3 -n production
```

---

## Inspecting a Deployment

```bash
# Quick status
kubectl get deployment api-server -n production

# Detailed status including events
kubectl describe deployment api-server -n production

# See all Pods managed by the Deployment
kubectl get pods -n production -l app=api-server

# See the ReplicaSets for this Deployment (current + history)
kubectl get replicasets -n production -l app=api-server
# NAME                      DESIRED   CURRENT   READY   AGE
# api-server-abc123         3         3         3       2d   ← current
# api-server-def456         0         0         0       5d   ← old (for rollback)
# api-server-ghi789         0         0         0       10d  ← older

# See rollout history
kubectl rollout history deployment/api-server -n production
# REVISION  CHANGE-CAUSE
# 1         Initial deployment
# 2         Update to v3.0.0
# 3         Update to v3.0.1

# See details of a specific revision
kubectl rollout history deployment/api-server -n production --revision=2

# Watch Pod events in real time
kubectl get events -n production --field-selector involvedObject.name=api-server --sort-by='.lastTimestamp'
```

---

## Pod Disruption Budget — Protect Deployments During Node Maintenance

A PodDisruptionBudget (PDB) prevents Kubernetes from terminating too many Pods at once during voluntary disruptions (node drains, cluster upgrades):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  minAvailable: 2               # at least 2 Pods must be available at all times
  # OR: maxUnavailable: 1       # at most 1 Pod can be unavailable
  selector:
    matchLabels:
      app: api-server
```

```bash
kubectl apply -f pdb.yaml

# Test: drain a node — PDB prevents all Pods from being evicted at once
kubectl drain ip-10-0-1-100.ec2.internal --ignore-daemonsets --delete-emptydir-data

# Kubernetes evicts Pods one at a time, respecting the PDB
# It will not evict the 3rd Pod until a replacement is Ready
```

---

## Deployment Annotations for CI/CD Traceability

Add annotations to track which CI/CD pipeline created the Deployment:

```yaml
metadata:
  annotations:
    # Set by CI/CD pipeline during deployment
    deployment.kubernetes.io/change-cause: "v3.0.1: Fix memory leak in session handler"
    ci.pipeline.url: "https://github.com/myorg/myapp/actions/runs/12345678"
    ci.commit.sha: "abc1234def5678"
    ci.deploy.user: "github-actions"
    ci.deploy.timestamp: "2024-01-15T10:00:00Z"
```

The `deployment.kubernetes.io/change-cause` annotation appears in `kubectl rollout history` — giving you a human-readable description of what each revision changed.

```bash
kubectl rollout history deployment/api-server -n production

# REVISION  CHANGE-CAUSE
# 1         Initial deployment
# 2         v3.0.0: Add Redis caching layer
# 3         v3.0.1: Fix memory leak in session handler  ← comes from annotation
```

---

## Summary

- Production Deployments need: readiness probes (traffic control), liveness probes (auto-restart), startup probes (slow apps), resource requests/limits, and `preStop` sleep for graceful shutdown
- Use `securityContext` to enforce least-privilege: `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, drop all capabilities
- `topologySpreadConstraints` distributes Pods across AZs; `podAntiAffinity` spreads across nodes within an AZ
- `PodDisruptionBudget` protects against excessive Pod eviction during node drains and cluster upgrades
- Use `--dry-run=client -o yaml` to generate a YAML template from an imperative command
- Annotate deployments with `deployment.kubernetes.io/change-cause` — appears in rollout history for human-readable change tracking

In the next file, `03-rolling-updates.md`, you will learn how rolling updates work — how Kubernetes gradually replaces old Pods with new ones, the `maxSurge` and `maxUnavailable` parameters, and how to monitor a rollout in progress.
