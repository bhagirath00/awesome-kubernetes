# Helm Chart Structure

## Directory Tree

```
web-app-chart/
├── Chart.yaml                # Chart metadata
├── values.yaml              # Default values
├── values-dev.yaml          # Development overrides
├── values-prod.yaml         # Production overrides
├── README.md                # Documentation
├── LICENSE                  # License
├── templates/
│   ├── deployment.yaml      # Deployment manifest
│   ├── service.yaml         # Service manifest
│   ├── ingress.yaml         # Ingress manifest
│   ├── configmap.yaml       # ConfigMap
│   ├── secret.yaml          # Secret
│   ├── serviceaccount.yaml  # ServiceAccount
│   ├── hpa.yaml             # HorizontalPodAutoscaler
│   ├── pdb.yaml             # PodDisruptionBudget
│   ├── _helpers.tpl         # Template helpers
│   ├── NOTES.txt            # Post-install notes
│   └── tests/
│       └── test-connection.yaml  # Tests
├── charts/                  # Dependency charts
│   └── mysql/
│       ├── Chart.yaml
│       └── templates/
└── .helmignore              # Files to ignore
```

---

## Chart.yaml Details

```yaml
apiVersion: v2
name: web-app
description: Production web application
version: 1.2.3                 # Chart version (SemVer)
appVersion: "1.2.3"           # Application version
type: application             # Chart type (application or library)
keywords:
  - web
  - api
  - production
maintainers:
  - name: Alice
    email: alice@example.com
  - name: Bob
    email: bob@example.com
home: https://github.com/company/web-app
sources:
  - https://github.com/company/web-app
  - https://hub.docker.com/r/company/web-app
icon: https://example.com/icon.png
dependencies:
  - name: mysql
    version: "^8.0.0"
    repository: https://charts.bitnami.com/bitnami
    condition: mysql.enabled
  - name: redis
    version: "^17.0.0"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

---

## values.yaml Organization

Structure values logically:

```yaml
# Global settings
namespace: default
nameOverride: ""
fullnameOverride: ""

# Image configuration
image:
  repository: web-app
  pullPolicy: IfNotPresent
  tag: ""  # Overrides appVersion

# Replicas and scaling
replicaCount: 3

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

# Resource limits
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Service configuration
service:
  type: ClusterIP
  port: 80
  targetPort: 8080
  annotations: {}

# Ingress configuration
ingress:
  enabled: false
  className: "nginx"
  annotations: {}
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com

# Pod configuration
podAnnotations: {}
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true

# Node selection
nodeSelector: {}
tolerations: []
affinity: {}

# Environment variables
env: {}

# ConfigMap
configMap:
  data: {}

# Secret
secret:
  data: {}

# ServiceAccount
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Dependencies
mysql:
  enabled: false
  auth:
    rootPassword: "password"
    username: "app"
    password: "app-password"
    database: "appdb"

redis:
  enabled: false
  auth:
    enabled: false
```

---

## Template Organization

Keep templates organized:

```
templates/
├── deployment.yaml          # Main app deployment
├── service.yaml             # Kubernetes Service
├── ingress.yaml             # Ingress for routing
├── configmap.yaml           # Configuration data
├── secret.yaml              # Secrets
├── serviceaccount.yaml      # RBAC identity
├── hpa.yaml                 # Auto-scaling
├── pdb.yaml                 # Disruption budget
├── NOTES.txt                # Post-install messages
├── _helpers.tpl             # Helper functions
└── tests/
    ├── test-connection.yaml # Integration tests
    └── test-config.yaml     # Configuration tests
```

---

## Required vs. Optional Templates

```yaml
# REQUIRED — always deployed
templates/
├── deployment.yaml
├── service.yaml
└── _helpers.tpl

# CONDITIONAL — deployed if enabled
templates/
├── ingress.yaml             # {{- if .Values.ingress.enabled }}
├── hpa.yaml                 # {{- if .Values.autoscaling.enabled }}
├── secret.yaml              # {{- if .Values.secret.create }}
└── mysql/                   # {{- if .Values.mysql.enabled }}
```

---

## Template Naming Conventions

```
deployment.yaml    → Main deployment
service.yaml       → Service exposure
ingress.yaml       → External routing
configmap.yaml     → Configuration
secret.yaml        → Sensitive data
serviceaccount.yaml → RBAC
hpa.yaml           → Horizontal Pod Autoscaler
pdb.yaml           → Pod Disruption Budget
_helpers.tpl       → Helper functions (underscore = internal)
NOTES.txt          → Post-install messages
```

---

## NOTES.txt

Post-installation instructions:

```yaml
1. Get the application URL by running these commands:
{{- if .Values.ingress.enabled }}
  http{{ if .Values.ingress.tls.enabled }}s{{ end }}://{{ .Values.ingress.host }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "web-app.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.service.type }}
  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
  You can watch the status by running 'kubectl get svc -w {{ include "web-app.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "web-app.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- end }}

2. View the logs:
  kubectl logs -n {{ .Release.Namespace }} deployment/{{ include "web-app.fullname" . }}

3. Get shell access:
  kubectl exec -it -n {{ .Release.Namespace }} deployment/{{ include "web-app.fullname" . }} -- /bin/sh
```

---

## Chart.yaml API Versions

```yaml
# Helm 2 charts (deprecated)
apiVersion: v1

# Helm 3 charts (current standard)
apiVersion: v2

# Difference:
# v1: dependencies in Chart.yaml, apiVersion in each manifest
# v2: dependencies as separate files, cleaner format
```

---

## .helmignore

Files to ignore when packaging:

```
# .helmignore
.git/
.gitignore
.DS_Store
*.swp
*.swo
*~
.idea/
*.iml
*.ipr
*.iws
.vscode/
*.code-workspace
.env
.env.local
tests/
docs/
*.md
```

---

## Chart Dependencies (Chart.yaml)

```yaml
dependencies:
  # Simple dependency
  - name: mysql
    version: 8.0.0
    repository: https://charts.bitnami.com/bitnami
  
  # Optional dependency
  - name: redis
    version: 17.0.0
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled      # Only if redis.enabled=true
  
  # Conditional by tags
  - name: postgresql
    version: 11.0.0
    repository: https://charts.bitnami.com/bitnami
    tags:
      - database
      - postgres
    condition: postgresql.enabled

# Update dependencies:
# helm dependency update web-app-chart
```

---

## Packaging Charts

```bash
# Create chart tarball
helm package web-app-chart

# Creates: web-app-chart-1.2.3.tgz

# Sign chart
helm package web-app-chart --sign --key mycert

# Verify signature
helm verify web-app-chart-1.2.3.tgz
```

---

## Chart Distribution

Options to share charts:

```
1. Local file: web-app-chart-1.2.3.tgz
2. Git repository
3. ChartMuseum (private repository)
4. Artifactory
5. Docker registry
6. Helm Hub (public)
```

---

## Summary

Chart structure:

**Core Files:**
- Chart.yaml (metadata)
- values.yaml (defaults)
- templates/ (manifests)

**Template Organization:**
- deployment, service, ingress
- configmap, secret, rbac
- hpa, pdb, tests

**Advanced:**
- Dependencies
- Conditional templates
- Helper functions

**Next:** File 03 covers values.yaml in detail.
