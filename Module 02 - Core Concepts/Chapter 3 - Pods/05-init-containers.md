# Init Containers

## Introduction

Init containers run **before** the main containers in a Pod. They run to completion one at a time, in order. Only when all init containers have successfully finished does Kubernetes start the main containers.

This sequencing is exactly what makes init containers powerful. You can run setup tasks — database migrations, configuration fetching, dependency checking — in a clean, controlled way before your application starts.

---

## Why Init Containers Exist

Imagine you have an application that:
- Needs a database schema to be migrated before it starts
- Needs to fetch a configuration file from AWS S3 before it can read its config
- Must wait for a database to be ready before attempting to connect
- Needs to set permissions on a shared volume before the main process starts

You could put all of this logic in your application's startup code. But that makes the application more complex, harder to test, and harder to replace. Init containers let you separate this concern completely — the init container handles setup, the main container handles the application.

```
Pod starts
    │
    ▼
Init Container 1: wait-for-database
    │ runs to completion (exit 0)
    │
    ▼
Init Container 2: run-migrations
    │ runs to completion (exit 0)
    │
    ▼
Main Container: api-server
    │ starts and runs continuously
```

If any init container fails, it is restarted (based on the Pod's `restartPolicy`) until it succeeds. The main container does not start until all init containers succeed.

---

## Init Container vs Sidecar Container

| | Init Container | Sidecar Container |
|---|----------------|------------------|
| **When it runs** | Before main containers, one at a time | Alongside main containers, simultaneously |
| **Lifecycle** | Runs to completion (exits) | Runs for the life of the Pod |
| **Purpose** | Setup, initialization, prerequisite checking | Ongoing helpers (logging, proxying) |
| **If it fails** | Pod cannot start | Main container may or may not be affected |
| **Image** | Can use a different image from main container | Can use any image |

---

## Use Case 1: Wait for a Dependency

The most common use case. Your application needs another service (database, cache, message broker) to be ready before it starts. An init container waits by polling the service:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
  namespace: production
spec:
  initContainers:

  # Init container 1: wait for PostgreSQL to be ready
  - name: wait-for-postgres
    image: busybox:1.35
    command:
    - /bin/sh
    - -c
    - |
      echo "Waiting for PostgreSQL to be ready..."
      until nc -z postgres-service.production.svc.cluster.local 5432; do
        echo "PostgreSQL not yet ready — sleeping 2 seconds"
        sleep 2
      done
      echo "PostgreSQL is ready!"

  # Init container 2: wait for Redis to be ready
  - name: wait-for-redis
    image: busybox:1.35
    command:
    - /bin/sh
    - -c
    - |
      until nc -z redis-service.production.svc.cluster.local 6379; do
        echo "Redis not ready yet — retrying..."
        sleep 2
      done
      echo "Redis is ready!"

  containers:
  - name: api-server
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:v2.0
    ports:
    - containerPort: 8080
```

`nc -z` checks if a TCP connection can be established. The init container keeps retrying every 2 seconds until it succeeds. Only then does the next init container (or main container) start.

This is much better than having the application itself retry and fail for 30 seconds during startup.

---

## Use Case 2: Database Migrations

Run database migrations before the application starts. This ensures the schema is always up to date before any application instance tries to use it:

```yaml
spec:
  initContainers:

  - name: run-migrations
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:v2.0   # same image as main app
    command: ["python", "manage.py", "migrate", "--noinput"]
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "1000m"
        memory: "512Mi"

  containers:
  - name: api-server
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:v2.0
    # ... main container config
```

Using the same image as the main container means the migration code is always in sync with the application code. When you deploy v2.0, the migrations for v2.0 run before any v2.0 application Pod starts serving traffic.

> ⚠️ **Warning:** In a rolling update, multiple Pods may run migrations simultaneously. Design your migrations to be **idempotent** — running them multiple times should have the same result as running once. Use `IF NOT EXISTS` in SQL migrations and tools like Flyway or Liquibase that track which migrations have already run.

---

## Use Case 3: Fetch Configuration from S3

Your application needs a large configuration file stored in AWS S3. An init container fetches it and writes it to a shared volume:

```yaml
spec:
  initContainers:

  - name: fetch-config
    image: amazon/aws-cli:latest
    command:
    - /bin/sh
    - -c
    - |
      echo "Fetching configuration from S3..."
      aws s3 cp s3://mycompany-configs/production/app-config.json /config/app-config.json
      echo "Configuration fetched successfully"
      ls -la /config/
    env:
    - name: AWS_REGION
      value: "us-east-1"
    volumeMounts:
    - name: config-volume
      mountPath: /config

  containers:
  - name: api-server
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/api-server:v2.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app         # reads the fetched config from here
      readOnly: true

  volumes:
  - name: config-volume
    emptyDir: {}                   # shared between init container and main container

  serviceAccountName: s3-reader-sa  # IAM role with s3:GetObject permission
```

The init container writes to `/config/app-config.json`. The main container reads from `/etc/app/app-config.json`. Both mount the same `config-volume` — they share the file through the `emptyDir` volume.

---

## Use Case 4: Set Permissions on Volumes

When using PersistentVolumes, the files might be owned by root. If your container runs as a non-root user, it cannot write to those files. An init container can fix the permissions:

```yaml
spec:
  initContainers:

  - name: fix-permissions
    image: busybox:1.35
    command:
    - /bin/sh
    - -c
    - |
      echo "Setting ownership on data volume..."
      chown -R 1000:2000 /data     # match the UID:GID of the main container user
      chmod 750 /data
      echo "Permissions set"
    securityContext:
      runAsUser: 0                 # run as root to change ownership
    volumeMounts:
    - name: data-volume
      mountPath: /data

  containers:
  - name: app
    image: my-app:v1.0
    securityContext:
      runAsUser: 1000             # run as non-root
    volumeMounts:
    - name: data-volume
      mountPath: /data            # now owned by UID 1000 — writable
```

---

## Use Case 5: Clone a Git Repository

Fetch application configuration or static assets from a Git repository before the main container starts:

```yaml
spec:
  initContainers:

  - name: clone-configs
    image: alpine/git:latest
    command:
    - /bin/sh
    - -c
    - |
      git clone --depth 1 \
        https://github.com/mycompany/app-configs.git \
        /configs
      echo "Cloned config repository:"
      ls /configs/
    volumeMounts:
    - name: config-repo
      mountPath: /configs

  containers:
  - name: app
    image: my-app:v1.0
    volumeMounts:
    - name: config-repo
      mountPath: /app/configs
      readOnly: true

  volumes:
  - name: config-repo
    emptyDir: {}
```

---

## Init Container Sequencing — Multiple Init Containers

When you have multiple init containers, they run in order — one at a time:

```yaml
initContainers:
- name: step-1-check-dependencies   # runs first, must exit 0
  ...
- name: step-2-fetch-config         # runs second, must exit 0
  ...
- name: step-3-run-migrations       # runs third, must exit 0
  ...

containers:
- name: main-app                    # starts only after all three succeed
  ...
```

If `step-2` fails, `step-3` does not run. The Pod goes back to `Pending` and Kubernetes retries `step-2` (with backoff). `step-1` does not re-run — only the failed step is retried.

---

## Checking Init Container Status

```bash
# See init container progress
kubectl get pod api-server -n production

# During init:
# NAME         READY   STATUS     RESTARTS   AGE
# api-server   0/1     Init:0/2   0          30s
# "Init:0/2" means 0 of 2 init containers have completed

# After init:
# NAME         READY   STATUS    RESTARTS   AGE
# api-server   1/1     Running   0          2m

# View init container logs
kubectl logs api-server -n production -c wait-for-postgres
kubectl logs api-server -n production -c run-migrations

# If init container fails, view its logs
kubectl logs api-server -n production -c run-migrations --previous

# Detailed view — shows init container states
kubectl describe pod api-server -n production

# Look for this section:
# Init Containers:
#   wait-for-postgres:
#     State:          Terminated
#       Reason:       Completed
#       Exit Code:    0
#   run-migrations:
#     State:          Running
#       Started:      Mon, 15 Jan 2024 10:00:10 +0000
```

---

## Init Containers and Resource Calculation

Kubernetes calculates Pod resource requirements differently when init containers are present:

- The effective Pod resource request = `max(largest init container request, sum of all main containers' requests)`
- This is because init containers run sequentially — only one runs at a time

```yaml
initContainers:
- name: migration
  resources:
    requests:
      cpu: "1000m"      # 1 CPU core (largest init container)
      memory: "1Gi"

containers:
- name: app
  resources:
    requests:
      cpu: "500m"       # 0.5 CPU core
      memory: "512Mi"
```

Effective request = `max(1000m, 500m)` = `1000m` CPU, `max(1Gi, 512Mi)` = `1Gi` memory.

The scheduler uses this effective request to find a node. After init containers finish and the main container starts, the node only needs `500m` CPU and `512Mi` memory.

---

## Summary

- Init containers run before main containers, one at a time, in order — all must succeed before the main container starts
- Unlike sidecar containers (which run alongside the main container), init containers run to completion and exit
- Common use cases: waiting for dependencies (`nc -z` polling), running database migrations, fetching config from S3, fixing volume permissions, cloning git repositories
- Init containers can use different images from the main container — use lightweight images (`busybox`, `alpine`, `aws-cli`) for setup tasks
- Multiple init containers run sequentially — if one fails, Kubernetes retries that one (not the whole sequence from the start)
- View init container progress: `kubectl get pod` shows `Init:0/2`, `Init:1/2`, then `Running`
- View init container logs: `kubectl logs <pod> -c <init-container-name>`
- Resource calculation: effective Pod request = max(largest init container, sum of main containers) — init containers can temporarily use more resources than the running application needs

This completes Chapter 3 — Pods. You now have a thorough understanding of what Pods are, their lifecycle, multi-container patterns, the full YAML spec, and init containers.

In Chapter 4, `01-what-are-namespaces.md`, you will learn how namespaces divide a single Kubernetes cluster into multiple virtual environments — enabling team isolation, resource quotas, and multi-tenancy.
