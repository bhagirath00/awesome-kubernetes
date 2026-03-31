# Helm Architecture

## High-Level Architecture

```
Helm Client (your laptop)
        ↓
Helm Repository (online)
        ↓ Downloads chart
Helm Library (templates)
        ↓ Renders
Kubernetes manifests (YAML)
        ↓ kubectl apply
Kubernetes Cluster
        ↓ Creates resources
Deployments, Services, Secrets, etc.
```

---

## Helm Components

### 1. Helm CLI

Command-line tool you run:

```bash
helm install
helm upgrade
helm list
helm status
helm delete
```

Runs on your machine or CI/CD pipeline.

### 2. Tiller (Helm 2 only)

In-cluster server component (Helm 2 architecture).

**Helm 2 (old, deprecated):**
```
Your machine
    ↓
Helm Client
    ↓
Tiller (server in cluster)
    ↓ Creates resources
Kubernetes
```

**Helm 3 (current):**
```
Your machine
    ↓
Helm Client
    ↓ Directly talks to Kubernetes API
Kubernetes
```

Helm 3 removed Tiller (simpler, safer).

### 3. Chart Repository

Hosted charts:

```
Bitnami repo
  ├─ postgresql
  ├─ redis
  ├─ mysql
  └─ ...

Official Helm repo
  ├─ prometheus
  ├─ grafana
  └─ ...

Your private repo
  ├─ internal-api
  ├─ internal-web
  └─ ...
```

---

## Chart Structure

```
my-app-chart/
├── Chart.yaml           # Chart metadata
├── values.yaml          # Default values
├── templates/
│   ├── deployment.yaml  # Kubernetes manifest templates
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl     # Template helpers
├── charts/              # Dependency charts
│   └── mysql/
├── values-prod.yaml     # Production override
└── README.md            # Documentation
```

---

## Chart.yaml

Metadata about the chart:

```yaml
apiVersion: v2
name: web-app
description: A Helm chart for web application
version: 1.2.3
appVersion: "1.2.3"
keywords:
  - web
  - app
maintainers:
  - name: DevOps Team
    email: devops@company.com
dependencies:
  - name: mysql
    version: 9.0.0
    repository: https://charts.bitnami.com/bitnami
```

---

## values.yaml

Default configuration values:

```yaml
# Image
image:
  repository: web-app
  tag: "1.2.3"
  pullPolicy: IfNotPresent

# Replicas
replicas: 3

# Resources
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Service
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# Ingress
ingress:
  enabled: false
  host: app.example.com
```

---

## Templates

Kubernetes manifests with Helm templating:

```yaml
# deployment.yaml template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
        version: {{ .Chart.AppVersion }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: {{ .Values.service.targetPort }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

Variables like `\{{ .Release.Name \}}` get filled from values.

---

## Real-World Example

### Chart Creation

```bash
# Create chart
helm create web-app-chart

# Edit values.yaml (development defaults)
values.yaml:
  replicas: 1
  image.tag: latest
  resources.requests.memory: 128Mi

# Edit templates/deployment.yaml
# Your specific logic

# Create environment overrides
values-prod.yaml:
  replicas: 3
  image.tag: v1.2.3
  resources.requests.memory: 512Mi
  resources.limits.memory: 1Gi
```

### Installation

```bash
# Development (use defaults)
helm install dev-app web-app-chart

# Production (override values)
helm install prod-app web-app-chart -f values-prod.yaml
```

---

## Helm Release Management

### Install

```bash
# Create release from chart
helm install my-app web-app-chart

# Install specific version
helm install my-app web-app-chart --version 1.2.3

# Install with custom namespace
helm install my-app web-app-chart -n production
```

### List

```bash
# View all releases
helm list

# View releases in specific namespace
helm list -n production

# Show detailed status
helm status my-app
```

### Upgrade

```bash
# Update to new chart version
helm upgrade my-app web-app-chart

# Override values during upgrade
helm upgrade my-app web-app-chart \
  --set replicas=5 \
  --set image.tag=v2.0.0

# Dry-run first (test)
helm upgrade my-app web-app-chart --dry-run --debug
```

### Rollback

```bash
# View release history
helm history my-app

# Rollback to previous version
helm rollback my-app 1

# Rollback specific revision
helm rollback my-app 3
```

### Delete

```bash
# Delete release
helm uninstall my-app

# Keep old data (don't delete)
helm uninstall my-app --keep-history
```

---

## Helm Hooks

Execute scripts at specific times:

```yaml
# Run migration before upgrade
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: pre-upgrade
    helm.sh/hook-weight: "-5"
spec:
  template:
    spec:
      containers:
      - name: migration
        image: web-app:v2.0.0
        command: ["python", "migrate.py"]

# Clean up after delete
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: post-delete
spec:
  template:
    spec:
      containers:
      - name: cleanup
        image: web-app:v1.0.0
        command: ["python", "cleanup.py"]
```

Hooks: pre-install, post-install, pre-upgrade, post-upgrade, pre-delete, post-delete.

---

## Summary

Helm architecture:

**Components:**
- Helm CLI (client)
- Chart Repository (online storage)
- Kubernetes API (no Tiller in v3)

**Chart Structure:**
- Chart.yaml (metadata)
- values.yaml (defaults)
- templates/ (manifest templates)

**Release Lifecycle:**
- Install (create)
- Upgrade (update)
- Rollback (revert)
- Delete (remove)

**Templates:**
- Jinja-like syntax
- Fill from values
- Conditionals and loops

**Next:** File 03 covers Helm installation and setup.
