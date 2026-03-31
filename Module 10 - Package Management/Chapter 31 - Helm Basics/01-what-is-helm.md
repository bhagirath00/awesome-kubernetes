# What is Helm?

## Introduction

You just wrote a perfect Kubernetes deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: web
        image: web:v1.2.3
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
```

Great! But now your colleague needs to deploy the same app.

```
Colleague: "Give me the manifest"
You: "Here's the YAML"
Colleague: "What about database?"
You: "Oh, separate file"
Colleague: "Dev, staging, prod different?"
You: "Yeah, modify these fields"
Colleague: "This is messy"
```

**Helm** packages it all cleanly:

```bash
helm install my-app web-app-chart

# Done! Everything deployed with defaults
# Can customize if needed
```

---

## What Helm Does

Helm is a package manager for Kubernetes, like npm for Node.js or apt for Linux:

```
npm: Node.js packages        helm install: Kubernetes apps
  npm install express          helm install prometheus
  Gets dependencies
  Configures setup

apt: Linux packages          helm install: Kubernetes apps
  apt install postgresql       helm install postgres
  Automatic config
  Dependencies handled
```

---

## Helm Template Example

```yaml
# Manifest (static)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: web
        image: web:v1.2.3

# Helm template (dynamic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Values.namespace }}
spec:
  replicas: {{ .Values.replicas }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        resources:
          requests:
            cpu: {{ .Values.resources.requests.cpu }}
            memory: {{ .Values.resources.requests.memory }}
```

---

## Use Cases

### Use Case 1: Deploy Postgres

```bash
# Without Helm (manual, error-prone)
1. Create namespace
2. Create secret for password
3. Create persistent volume
4. Create persistent volume claim
5. Create StatefulSet
6. Create Service
7. Configure networking

# With Helm (one command)
helm install postgres bitnami/postgresql \
  --set postgresqlPassword=mypassword \
  --set persistence.size=50Gi

# Everything done
```

### Use Case 2: Install Prometheus Monitoring

```bash
# Without Helm (20+ manifests)
1. Deploy Prometheus server
2. Deploy Grafana
3. Deploy Alertmanager
4. Deploy Node Exporter
5. Configure ServiceMonitors
6. Configure alerts
7. Set up RBAC
... all manual

# With Helm (one command)
helm install monitoring prometheus-community/kube-prometheus-stack

# Full monitoring stack ready
```

---

## Three Core Concepts

### 1. Chart

Chart = Helm package (like deb or rpm file)

```
web-app-chart/
├── Chart.yaml          (metadata)
├── values.yaml         (defaults)
├── templates/
│   ├── deployment.yaml (template)
│   ├── service.yaml
│   └── configmap.yaml
└── README.md
```

### 2. Repository

Repository = Helm charts collection (like apt repo)

```bash
# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Find chart
helm search repo bitnami/postgresql

# Install from repo
helm install my-db bitnami/postgresql
```

### 3. Release

Release = Installed instance of a chart

```bash
# Install chart = Create release
helm install my-app web-app-chart
# Release: my-app

# Same chart, multiple releases
helm install dev-app web-app-chart
helm install prod-app web-app-chart
# Three independent deployments
```

---

## Real-World Workflow

```
Day 1: Developer creates chart
  web-app-chart/
    ├── templates (deployment, service)
    └── values.yaml (defaults)

Day 2: Deploy to dev
  helm install dev-web-app web-app-chart \
    --values dev-values.yaml

Day 3: Deploy to staging
  helm install staging-web-app web-app-chart \
    --values staging-values.yaml

Day 4: Deploy to prod
  helm install prod-web-app web-app-chart \
    --values prod-values.yaml \
    --values prod-secrets.yaml

Same chart, three different deployments
Each customized for environment
```

---

## Helm vs. Kustomize

```
Helm:
  - Full templating language
  - Package manager (chart repositories)
  - Lifecycle management (install/upgrade/rollback)
  - Popular, many charts available
  - Steeper learning curve

Kustomize:
  - Simpler, patch-based
  - No package manager
  - Part of kubectl (kubectl apply -k)
  - Lightweight
  - Less powerful
```

Helm more powerful, Kustomize simpler.

---

## Helm Commands Overview

```bash
# Search for charts
helm search repo postgres

# Install release
helm install my-db bitnami/postgresql

# View installed releases
helm list

# Check release status
helm status my-db

# Upgrade release
helm upgrade my-db bitnami/postgresql

# Rollback to previous version
helm rollback my-db 1

# Delete release
helm uninstall my-db

# View values
helm get values my-db
```

---

## Common Mistakes

### Mistake 1: No Version Control

```bash
# WRONG — manual helm install, values untracked
helm install app my-chart \
  --set replicas=3 \
  --set image.tag=v1.2.3

# Months later: Can't remember the values used
```

**Fix:** Track in Git.

```bash
# RIGHT
git commit values-prod.yaml
helm install app my-chart -f values-prod.yaml
```

### Mistake 2: Chart Not Idempotent

```yaml
# WRONG — installing twice creates issues
helm install app chart1
helm install app chart1
# Error: Release "app" already exists

# RIGHT — use upgrade
helm upgrade --install app chart1
# Creates if not exists, updates if exists
```

---

## Summary

Helm overview:

**What it is:**
- Package manager for Kubernetes
- Like npm (Node), apt (Linux), brew (Mac)

**Three concepts:**
1. Chart (package)
2. Repository (collection of charts)
3. Release (installed chart)

**Workflow:**
1. Add repository
2. Search for chart
3. Install as release
4. Customize with values
5. Upgrade when needed

**Benefits:**
- Repeatable deployments
- Environment-specific configs
- Easy upgrades and rollbacks
- Large chart ecosystem

**Next:** File 02 covers Helm architecture.
