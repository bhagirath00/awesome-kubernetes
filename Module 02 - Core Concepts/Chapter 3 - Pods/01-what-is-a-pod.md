# What is a Pod

## Introduction

A Pod is the smallest deployable unit in Kubernetes. Everything you run in Kubernetes — whether a web server, a database, a batch job, or a monitoring agent — runs inside a Pod. You cannot run a container directly in Kubernetes without a Pod wrapping it.

Understanding Pods deeply is the foundation of everything else in Kubernetes. Before you can understand Deployments, StatefulSets, Services, or health checks, you need to know exactly what a Pod is, what it contains, and how Kubernetes thinks about it.

---

## What a Pod Actually Is

A Pod is a wrapper around one or more containers that share the same:

- **Network namespace** — all containers in a Pod share the same IP address and port space
- **Storage** — containers in a Pod can share volumes (filesystem directories)
- **Lifecycle** — all containers in a Pod are created and destroyed together

The word "Pod" comes from a pod of whales or a pea pod — a group of things that naturally belong together.

In almost all cases, a Pod contains exactly **one container**. Multi-container Pods are a specific pattern used in specific situations — covered in `03-multi-container-pods.md`.

```
┌─────────────────────────────────────────────────┐
│                      POD                        │
│   IP: 10.0.1.45                                 │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │          Container: web-app             │   │
│   │          Image: nginx:1.25              │   │
│   │          Port: 80                       │   │
│   └─────────────────────────────────────────┘   │
│                                                 │
│   Volume: /data (shared between containers)     │
└─────────────────────────────────────────────────┘
```

---

## Why Pods Exist — The Design Reason

You might wonder: why does Kubernetes not just run containers directly? Why the extra layer?

The reason is that some applications need tightly-coupled helper processes that must run on the same machine and share the same network. For example:

- A web application container and a log-shipping sidecar that reads the same log files
- A service container and an Envoy proxy container that intercepts all its network traffic
- An application container and an init container that pre-populates a config file before the app starts

By grouping these into a Pod, Kubernetes guarantees they are always co-located on the same node, share the same network interface, and share volumes. You get tight coupling where you need it, without abandoning the separation that containers provide.

For 95% of use cases, you have one container per Pod, and this co-location behavior is irrelevant. But the Pod abstraction exists for the 5% where it matters greatly.

---

## Your First Pod YAML

Every Kubernetes object is described in YAML. Here is the simplest valid Pod definition:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  namespace: default
spec:
  containers:
  - name: web
    image: nginx:1.25
```

Apply it:

```bash
kubectl apply -f pod.yaml

# Output:
# pod/web-pod created
```

Check it:

```bash
kubectl get pod web-pod

# Output:
# NAME      READY   STATUS    RESTARTS   AGE
# web-pod   1/1     Running   0          30s
```

`1/1` means 1 out of 1 containers in the Pod is ready and running.

---

## The Key Fields of a Pod

### `apiVersion` and `kind`

```yaml
apiVersion: v1
kind: Pod
```

Pods use the core API group (`v1`). The `kind` tells Kubernetes what type of object this is.

### `metadata`

```yaml
metadata:
  name: web-pod           # unique name within the namespace
  namespace: production   # which namespace this Pod belongs to
  labels:                 # key-value pairs for grouping and selecting
    app: web
    environment: production
    version: "1.25"
  annotations:            # key-value pairs for storing extra information
    deploy-date: "2024-01-15"
    deployed-by: "jenkins-pipeline"
```

**Labels** are used by Services and Deployments to find Pods. **Annotations** store metadata that is not used for selection — documentation, build info, tool-specific config.

### `spec.containers`

The `containers` field is a list. Each item in the list defines one container:

```yaml
spec:
  containers:
  - name: web                          # container name (unique within the Pod)
    image: nginx:1.25                  # Docker image to run
    ports:
    - containerPort: 80               # port the container listens on (informational)
    
    env:                               # environment variables
    - name: ENVIRONMENT
      value: "production"
    - name: DB_HOST
      value: "postgres.production.svc.cluster.local"
    
    resources:                         # CPU and memory requests and limits
      requests:
        cpu: "250m"                   # 0.25 CPU cores guaranteed
        memory: "128Mi"              # 128 MB RAM guaranteed
      limits:
        cpu: "500m"                   # 0.5 CPU cores maximum
        memory: "256Mi"              # 256 MB RAM maximum
    
    volumeMounts:                      # mount volumes into the container
    - name: config-volume
      mountPath: /etc/app/config     # path inside the container
```

### `spec.volumes`

Volumes are defined at the Pod level and mounted into containers:

```yaml
spec:
  volumes:
  - name: config-volume
    configMap:
      name: app-config              # mounts data from a ConfigMap
  
  - name: data-volume
    emptyDir: {}                    # temporary directory, lives as long as the Pod
  
  - name: logs-volume
    emptyDir: {}
  
  containers:
  - name: web
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app/config
    - name: logs-volume
      mountPath: /var/log/app
```

---

## A Complete, Production-Style Pod

Here is what a real-world Pod specification looks like:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server-pod
  namespace: production
  labels:
    app: api-server
    version: "2.1.0"
    environment: production
  annotations:
    git-commit: "a1b2c3d4"
    build-number: "jenkins-build-142"
spec:
  containers:
  - name: api-server
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:2.1.0
    ports:
    - name: http
      containerPort: 8080
      protocol: TCP

    env:
    - name: PORT
      value: "8080"
    - name: ENVIRONMENT
      value: "production"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials    # read from a Kubernetes Secret
          key: password
    - name: APP_CONFIG
      valueFrom:
        configMapKeyRef:
          name: api-config       # read from a ConfigMap
          key: config.json

    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1000m"
        memory: "1Gi"

    livenessProbe:               # is the container alive?
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30   # wait 30s after start before first check
      periodSeconds: 10
      failureThreshold: 3       # restart after 3 consecutive failures

    readinessProbe:              # is the container ready to serve traffic?
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3

    volumeMounts:
    - name: tmp-dir
      mountPath: /tmp

  volumes:
  - name: tmp-dir
    emptyDir: {}

  restartPolicy: Always         # Always (default), OnFailure, Never

  terminationGracePeriodSeconds: 30  # give the container 30s to shut down cleanly
```

---

## Pods Are Ephemeral — This is Critical to Understand

Pods are designed to be temporary. They can be created, destroyed, and replaced at any time. This is intentional and important:

- When you update a Deployment, old Pods are deleted and new ones are created
- When a node fails, its Pods are terminated and new Pods are scheduled on other nodes
- When the cluster auto-scales down, Pods on removed nodes are terminated

**A Pod does not have a permanent identity.** Every time a Pod is created, it gets a new random name (like `web-deployment-7d6f8b-x2k9p`) and a new IP address.

This means:
- **Never connect to a Pod directly by IP** — use a Service instead
- **Never store permanent data in a Pod's local filesystem** — it will be lost when the Pod is deleted. Use PersistentVolumes.
- **Never assume a Pod will always be on the same node** — Kubernetes can reschedule it anywhere

---

## Interacting With Pods

```bash
# Get all pods in all namespaces
kubectl get pods --all-namespaces

# Get pods in a specific namespace with more detail
kubectl get pods -n production -o wide

# Output:
# NAME              READY   STATUS    RESTARTS   AGE   IP            NODE
# api-server-pod    1/1     Running   0          5m    10.0.1.45     ip-10-0-1-100.ec2.internal

# See everything about a Pod
kubectl describe pod api-server-pod -n production

# Get Pod YAML (current state including computed fields)
kubectl get pod api-server-pod -n production -o yaml

# View container logs
kubectl logs api-server-pod -n production

# Follow logs in real-time
kubectl logs -f api-server-pod -n production

# Logs for a specific container (in multi-container Pods)
kubectl logs api-server-pod -n production -c api-server

# Execute a command inside the running container
kubectl exec -it api-server-pod -n production -- bash

# Run a quick command and exit
kubectl exec api-server-pod -n production -- env | grep DB

# Copy a file from a Pod to your local machine
kubectl cp production/api-server-pod:/app/logs/error.log ./error.log

# Delete a Pod (Deployment will recreate it automatically)
kubectl delete pod api-server-pod -n production
```

---

## Pod Status Values

```bash
kubectl get pods
# NAME           READY   STATUS             RESTARTS   AGE
```

| Status | Meaning |
|--------|---------|
| `Pending` | Pod accepted by API Server, waiting to be scheduled or for image pull |
| `Running` | At least one container is running |
| `Succeeded` | All containers completed successfully (for Jobs) |
| `Failed` | All containers terminated, at least one with non-zero exit code |
| `Unknown` | Node cannot be reached (network issue) |
| `CrashLoopBackOff` | Container keeps crashing and restarting with increasing delays |
| `ImagePullBackOff` | Cannot pull the container image (wrong name, wrong credentials) |
| `ContainerCreating` | Pod scheduled on a node, containers starting |
| `Terminating` | Pod is being deleted |
| `OOMKilled` | Container was killed because it exceeded its memory limit |

---

## Summary

- A Pod is the smallest deployable unit in Kubernetes — every container runs inside a Pod
- Pods share network (same IP), storage (volumes), and lifecycle across all their containers
- The Pod YAML has four key sections: `apiVersion`, `kind`, `metadata`, and `spec`
- The `spec.containers` list defines each container with its image, ports, environment variables, resources, health probes, and volume mounts
- Pods are **ephemeral** — they are created and destroyed constantly. Never use a Pod's IP directly. Never store permanent data in a Pod's local filesystem.
- Common Pod statuses: Running, Pending, CrashLoopBackOff, ImagePullBackOff, OOMKilled
- Core commands: `kubectl get pods`, `kubectl describe pod`, `kubectl logs`, `kubectl exec`

In the next file, `02-pod-lifecycle.md`, you will learn the exact phases a Pod goes through from creation to termination — including what happens during graceful shutdown and why `terminationGracePeriodSeconds` matters for zero-downtime deployments.
