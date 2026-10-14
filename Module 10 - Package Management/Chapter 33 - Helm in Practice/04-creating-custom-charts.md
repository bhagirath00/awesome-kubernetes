# Creating Custom Charts

## Create Chart Scaffold

```bash
# Generate new chart
helm create my-app-chart

# This creates:
# my-app-chart/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# ├── templates/
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── hpa.yaml
# │   ├── ingress.yaml
# │   ├── configmap.yaml
# │   ├── _helpers.tpl
# │   └── NOTES.txt
# └── README.md
```

---

## Build Custom Chart Step-by-Step

### Step 1: Update Chart Metadata

```yaml
# Chart.yaml
apiVersion: v2
name: my-web-app
description: Production web application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - web
  - app
maintainers:
  - name: Your Name
    email: you@company.com
```

### Step 2: Define Default Values

```yaml
# values.yaml
namespace: default

image:
  repository: my-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent

replicaCount: 3

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  host: myapp.example.com

env:
  LOG_LEVEL: INFO
  DEBUG: "false"
```

### Step 3: Create Deployment Template

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app-chart.fullname" . }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "my-app-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

### Step 4: Add Conditional Ingress

```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app-chart.fullname" . }}
spec:
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ include "my-app-chart.fullname" . }}
            port:
              number: {{ .Values.service.port }}
{{- end }}
```

---

## Test Custom Chart

```bash
# Validate chart syntax
helm lint my-app-chart

# Render templates locally
helm template my-release my-app-chart

# Render with custom values
helm template my-release my-app-chart \
  -f my-app-chart/values-prod.yaml

# Test install (dry-run)
helm install my-release my-app-chart --dry-run --debug

# Install for real
helm install my-release my-app-chart

# Verify
helm status my-release
```

---

## Real-World Example: E-Commerce API

```bash
# Create chart structure
helm create ecommerce-api

# File: ecommerce-api/values.yaml
image:
  repository: my-registry/ecommerce-api
  tag: "1.0.0"

replicaCount: 3

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

database:
  enabled: true
  type: postgresql
  host: postgres.databases.svc
  port: 5432
  name: ecommerce

cache:
  enabled: true
  host: redis.cache.svc
  port: 6379

ingress:
  enabled: true
  host: api.company.com
  tls:
    enabled: true

# File: ecommerce-api/templates/deployment.yaml
# (with database and cache environment variables)

# Install
helm install prod-api ./ecommerce-api \
  -n production \
  -f values-prod.yaml
```

---

## Chart Dependencies

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: 11.0.0
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled

  - name: redis
    version: 17.0.0
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled

# values.yaml
postgresql:
  enabled: true
  auth:
    username: appuser
    password: password
    database: appdb

redis:
  enabled: true
  auth:
    enabled: false
```

Update dependencies:

```bash
helm dependency update my-app-chart
# Downloads postgres and redis charts
```

---

## Publish Chart

### Option 1: Local File

```bash
# Package chart
helm package my-app-chart
# Creates: my-app-chart-1.0.0.tgz

# Install from local file
helm install my-release ./my-app-chart-1.0.0.tgz
```

### Option 2: Git Repository

```bash
# Push chart to git
git add my-app-chart/
git commit -m "Add my-app-chart"
git push

# Install from repo
helm install my-release https://github.com/company/charts/tree/main/my-app-chart
```

### Option 3: ChartMuseum (Private Repo)

```bash
# Push to ChartMuseum
curl --data-binary @my-app-chart-1.0.0.tgz \
  http://chartmuseum.company.com/api/charts

# Install from ChartMuseum
helm repo add company http://chartmuseum.company.com
helm install my-release company/my-app-chart
```

---

## Common Chart Patterns

### Pattern 1: Multiple Environment Support

```yaml
# values-dev.yaml
replicas: 1
environment: development
database: false

# values-prod.yaml
replicas: 5
environment: production
database: true
resources:
  requests:
    memory: 1Gi
```

### Pattern 2: Optional Components

```yaml
# values.yaml
cache:
  enabled: false

monitoring:
  enabled: true

# template uses conditionals
{{- if .Values.cache.enabled }}
CACHE_URL: redis://...
{{- end }}
```

---

## Best Practices for Custom Charts

1. **Follow Naming Conventions**
   - Descriptive chart names
   - Consistent template names
   - Clear value names

2. **Provide Good Defaults**
   - Sensible production-like defaults
   - Document alternatives
   - Include examples

3. **Use Conditionals Wisely**
   - Optional features
   - Environment-specific resources
   - Dependency flags

4. **Document Everything**
   - README with usage examples
   - Chart.yaml with metadata
   - Comments in values.yaml

5. **Test Thoroughly**
   - helm lint
   - helm template
   - helm install --dry-run
   - Real install and verify

---

## Summary

Creating custom charts:

**Setup:**
- helm create chart-name
- Update Chart.yaml

**Configuration:**
- Define values.yaml
- Create templates/

**Features:**
- Conditionals for optional parts
- Environment-specific overrides
- Dependencies management

**Testing:**
- helm lint (validate)
- helm template (render)
- helm install --dry-run (test)

**Distribution:**
- Package (local file)
- Git repository
- ChartMuseum

**Next:** File 05 covers Helm repositories.
