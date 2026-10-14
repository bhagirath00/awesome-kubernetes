# Pod YAML Structure

## Introduction

Every Kubernetes object — Pod, Deployment, Service, ConfigMap — is defined in YAML. The Pod YAML is the most fundamental of all because Deployments, StatefulSets, and DaemonSets all contain a Pod template inside them. Master the Pod YAML and you understand every workload resource in Kubernetes.

This file is a complete field-by-field reference of the Pod YAML specification. Every important field is explained and demonstrated with real examples.

---

## The Four Top-Level Fields

Every Kubernetes object YAML starts with these four fields:

```yaml
apiVersion: v1          # which API to use
kind: Pod               # what kind of object
metadata:               # name, namespace, labels, annotations
  ...
spec:                   # the actual specification of what you want
  ...
```

---

## `metadata` — Identity and Organization

```yaml
metadata:
  name: web-pod                     # required — unique name in the namespace
  namespace: production             # which namespace (default: "default")
  
  labels:                           # key-value pairs for selection and grouping
    app: web-app
    version: "2.1.0"
    environment: production
    team: platform
    tier: frontend
  
  annotations:                      # key-value pairs for metadata (not for selection)
    kubernetes.io/description: "Web application frontend pod"
    deploy-timestamp: "2024-01-15T10:00:00Z"
    git-commit: "a1b2c3d4e5f6"
    jenkins-build: "https://jenkins.myco.com/build/142"
    contact: "platform-team@myco.com"
```

**Labels** are queryable — Services and Deployments use them to find Pods. **Annotations** are not queryable — they store documentation, build info, and tool-specific configuration. You can store up to 256KB of data in annotations.

---

## `spec.containers` — The Container List

The most important section. Each item defines one container:

### Image

```yaml
containers:
- name: web                                    # container name — unique within the Pod
  image: nginx:1.25                            # image:tag
  image: nginx                                 # no tag = latest (avoid in production)
  image: 123456789.dkr.ecr.us-east-1.amazonaws.com/web-app:v2.1.0  # full ECR path
  
  imagePullPolicy: IfNotPresent               # IfNotPresent | Always | Never
  # IfNotPresent: use cached image if available (best for tagged images)
  # Always: always pull from registry (use with 'latest' tag)
  # Never: fail if not cached (air-gapped environments)
```

### Ports

```yaml
  ports:
  - name: http                # optional name — referenced by Services and probes
    containerPort: 8080       # port the container process listens on
    protocol: TCP             # TCP (default) or UDP
  - name: metrics
    containerPort: 9090
    protocol: TCP
```

> ⚠️ **Note:** `ports` in a Pod spec is purely **informational**. Declaring a port here does not open or close it. A container can accept traffic on any port regardless of what is declared here. The purpose is documentation and to allow other tools to discover which ports the app uses.

### Environment Variables

```yaml
  env:
  # Literal value
  - name: ENVIRONMENT
    value: "production"
  
  # Value from a Secret
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials      # Secret name
        key: password             # key within the Secret
  
  # Value from a ConfigMap
  - name: APP_PORT
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: port
  
  # Value from Pod metadata (Downward API)
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name       # inject the Pod's own name
  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName      # inject the node name
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  
  # All keys from a ConfigMap as environment variables
  envFrom:
  - configMapRef:
      name: app-config              # every key in app-config becomes an env var
  - secretRef:
      name: app-secrets             # every key in app-secrets becomes an env var
```

### Resources — CPU and Memory

```yaml
  resources:
    requests:                       # guaranteed minimum (used by scheduler for placement)
      cpu: "500m"                  # 500 millicores = 0.5 CPU cores
      memory: "512Mi"              # 512 mebibytes
    limits:                         # hard maximum (enforced by Linux cgroups)
      cpu: "1000m"                 # 1 CPU core
      memory: "1Gi"                # 1 gibibyte
```

**CPU values:**
- `"1"` = 1 full CPU core
- `"500m"` = 0.5 CPU core (m = millicores, 1000m = 1 core)
- `"250m"` = 0.25 CPU core

**Memory values:**
- `"256Mi"` = 256 mebibytes (1 Mi = 1,048,576 bytes)
- `"1Gi"` = 1 gibibyte
- `"512M"` = 512 megabytes (M ≠ Mi — use Mi to be precise)

If a container exceeds its **memory limit**, it is killed immediately with `OOMKilled`. If it exceeds its **CPU limit**, it is throttled (slowed down) but not killed.

### Health Probes

```yaml
  # Startup probe: protect slow-starting containers
  # Kubernetes does NOT run liveness/readiness until startup probe passes
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30     # try 30 times
    periodSeconds: 10        # every 10 seconds = up to 5 minutes to start
  
  # Liveness probe: is the container still working? Restart if not.
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 0    # 0 because startupProbe runs first
    periodSeconds: 10
    timeoutSeconds: 5         # probe times out after 5 seconds
    failureThreshold: 3       # restart after 3 consecutive failures
    successThreshold: 1       # (for liveness, must always be 1)
  
  # Readiness probe: is the container ready to receive traffic?
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 3
    successThreshold: 1       # mark ready after 1 success (can be > 1)
```

**Three probe types** (pick the right one for your endpoint):

```yaml
  # HTTP GET — most common for web services
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
      httpHeaders:
      - name: Authorization
        value: "Bearer healthcheck-token"
  
  # TCP Socket — for services that don't have HTTP (databases, SMTP)
  livenessProbe:
    tcpSocket:
      port: 5432              # success if connection accepted
  
  # Exec — run a command inside the container
  livenessProbe:
    exec:
      command:
      - /bin/sh
      - -c
      - "redis-cli ping | grep PONG"   # success if exit code 0
```

### Volume Mounts

```yaml
  volumeMounts:
  - name: config-volume          # matches a volume name in spec.volumes
    mountPath: /etc/app          # path inside the container
    readOnly: true               # optional — mount as read-only
  
  - name: data-volume
    mountPath: /var/data
    subPath: myapp               # mount only this subdirectory, not the whole volume
```

### Lifecycle Hooks

```yaml
  lifecycle:
    postStart:                    # runs immediately after container starts
      exec:
        command: ["/bin/sh", "-c", "echo 'Container started' >> /var/log/startup.log"]
    
    preStop:                      # runs before SIGTERM is sent (for graceful shutdown)
      exec:
        command: ["/bin/sleep", "5"]   # wait 5 seconds for kube-proxy to drain traffic
```

---

## `spec.volumes` — Storage Definitions

Volumes are defined at the Pod level and referenced by containers:

```yaml
spec:
  volumes:
  
  # Temporary in-memory storage (deleted when Pod dies)
  - name: tmp-storage
    emptyDir:
      medium: Memory             # store in RAM for fast access (default: disk)
      sizeLimit: 256Mi
  
  # ConfigMap as a directory of files
  - name: app-config
    configMap:
      name: app-config-cm         # ConfigMap name
      items:
      - key: config.json
        path: config.json         # file name inside the mountPath directory
  
  # Secret as a directory of files
  - name: tls-certs
    secret:
      secretName: tls-secret
      defaultMode: 0400           # file permissions (readable by owner only)
  
  # EBS volume (AWS)
  - name: data-volume
    awsElasticBlockStore:
      volumeID: vol-0abc123def456789  # EBS volume ID
      fsType: ext4
  
  # PersistentVolumeClaim (the standard way to use persistent storage)
  - name: database-storage
    persistentVolumeClaim:
      claimName: postgres-pvc     # name of the PVC object
      readOnly: false
  
  # Host path (mount a directory from the node's filesystem)
  - name: host-logs
    hostPath:
      path: /var/log/pods
      type: Directory
```

---

## `spec` — Pod-Level Settings

These fields apply to the entire Pod, not individual containers:

```yaml
spec:
  restartPolicy: Always              # Always | OnFailure | Never
  
  terminationGracePeriodSeconds: 60  # seconds to wait before SIGKILL (default: 30)
  
  # Run as a specific user
  securityContext:
    runAsUser: 1000                  # UID to run all containers as
    runAsGroup: 3000                 # GID
    runAsNonRoot: true               # reject if any container runs as root
    fsGroup: 2000                    # GID for volume ownership
  
  # DNS settings
  dnsPolicy: ClusterFirst            # use cluster DNS (default)
  dnsConfig:
    nameservers:
    - 8.8.8.8
    searches:
    - production.svc.cluster.local
  
  # Node selection
  nodeSelector:
    kubernetes.io/arch: amd64        # only schedule on amd64 nodes
    eks.amazonaws.com/nodegroup: gpu-nodes  # only on specific node group
  
  # Service account for IAM permissions (EKS IRSA)
  serviceAccountName: my-app-sa     # service account with IAM role annotation
  
  # Tolerations — allow scheduling on tainted nodes
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  
  # Image pull secrets (for private registries not on EKS IRSA)
  imagePullSecrets:
  - name: ecr-pull-secret
  
  # Share the host's PID namespace (advanced — security risk)
  hostPID: false                     # default false
  
  # Share the host's network (advanced)
  hostNetwork: false                 # default false
```

---

## A Complete Production Pod Example

Putting everything together:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-service
  namespace: production
  labels:
    app: api-service
    version: "3.2.1"
    environment: production
    team: backend
  annotations:
    git-commit: "f1e2d3c4"
    deploy-by: "jenkins"
spec:
  serviceAccountName: api-service-sa   # IRSA role for AWS permissions

  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000

  terminationGracePeriodSeconds: 60

  containers:
  - name: api
    image: 123456789.dkr.ecr.us-east-1.amazonaws.com/api-service:3.2.1
    imagePullPolicy: IfNotPresent

    ports:
    - name: http
      containerPort: 8080

    env:
    - name: ENVIRONMENT
      value: "production"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name

    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "1000m"
        memory: "1Gi"

    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 12
      periodSeconds: 10

    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 10
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 3

    lifecycle:
      preStop:
        exec:
          command: ["/bin/sleep", "5"]

    volumeMounts:
    - name: config
      mountPath: /etc/app
      readOnly: true
    - name: tmp
      mountPath: /tmp

  volumes:
  - name: config
    configMap:
      name: api-service-config
  - name: tmp
    emptyDir: {}
```

---

## Summary

- Every Pod YAML has four top-level fields: `apiVersion`, `kind`, `metadata`, and `spec`
- `metadata` contains the name, namespace, labels (for selection), and annotations (for documentation)
- `spec.containers` is a list — each item specifies image, ports, env vars, resources, probes, and volume mounts
- Port declarations in Pod spec are informational only — they do not open or close ports
- Environment variables can come from literal values, Secrets (`secretKeyRef`), ConfigMaps (`configMapKeyRef`), or Pod metadata (Downward API)
- CPU is in millicores (`500m` = 0.5 cores); memory is in mebibytes (`512Mi`). Exceeding memory limit = OOMKilled. Exceeding CPU limit = throttled.
- Use `startupProbe` for slow-starting apps, `livenessProbe` for crash detection, `readinessProbe` for traffic control
- `preStop: sleep 5` is a production best practice for zero-downtime Pod termination
- `serviceAccountName` connects the Pod to an IAM role via EKS IRSA for AWS API access

In the next file, `05-init-containers.md`, you will learn how init containers run setup tasks before your main containers start — and why they are the right pattern for database migration, config fetching, and dependency waiting.
