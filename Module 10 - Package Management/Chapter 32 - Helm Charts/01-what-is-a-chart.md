# What is a Helm Chart

## Chart Definition

A Helm chart is a packaged Kubernetes application:

```
Chart = Package + Configuration + Templates
        (like .jar, .deb, or .rpm file for Kubernetes)
```

It contains everything needed to deploy an application.

---

## Chart Directory Structure

```
myapp-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
├── values-production.yaml
├── values-staging.yaml
├── charts/
│   └── mysql/
│       └── Chart.yaml
└── README.md
```

---

## Chart.yaml

Metadata file:

```yaml
apiVersion: v2                    # Helm API version
name: myapp                       # Chart name
description: My application       # Description
version: 1.0.0                    # Chart version (you control)
appVersion: "1.0.0"              # Application version
keywords:
  - web
  - api
maintainers:
  - name: DevOps Team
    email: devops@company.com
home: https://github.com/company/myapp
sources:
  - https://github.com/company/myapp
dependencies:                     # Other charts this needs
  - name: mysql
    version: 8.0.0
    repository: https://charts.bitnami.com/bitnami
```

---

## values.yaml

Default configuration:

```yaml
# Global settings
namespace: default

# Docker image
image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

# Replicas
replicaCount: 3

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
  className: "nginx"
  host: myapp.example.com
  path: /
  tls:
    enabled: false
    secretName: myapp-tls

# Environment variables
env:
  LOG_LEVEL: INFO
  DATABASE_URL: postgresql://localhost

# Database dependency
mysql:
  enabled: false
  auth:
    rootPassword: "password"
    database: "myapp"
```

---

## Template Example: deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
      - name: {{ .Chart.Name }}
        securityContext:
          {{- toYaml .Values.securityContext | nindent 12 }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
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
        volumeMounts:
        - name: config
          mountPath: /etc/config
      volumes:
      - name: config
        configMap:
          name: {{ include "myapp.fullname" . }}
```

---

## Template Example: service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  namespace: {{ .Values.namespace }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    protocol: TCP
    name: http
  selector:
    {{- include "myapp.selectorLabels" . | nindent 4 }}
```

---

## Conditional Templates

Include only if enabled:

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: {{ .Values.ingress.path }}
        pathType: Prefix
        backend:
          service:
            name: {{ include "myapp.fullname" . }}
            port:
              number: {{ .Values.service.port }}
{{- end }}
```

---

## Loop Templates

Iterate over values:

```yaml
# ConfigMap with multiple environment variables
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}
data:
  {{- range $key, $value := .Values.env }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}

# Creates:
# LOG_LEVEL: "INFO"
# DATABASE_URL: "postgresql://localhost"
```

---

## Built-in Variables

Available in every template:

```
.Release.Name          # Release name (e.g., "my-app")
.Release.Namespace     # Namespace
.Chart.Name            # Chart name (e.g., "myapp")
.Chart.Version         # Chart version
.Chart.AppVersion      # App version
.Values.xxx            # Values from values.yaml
.Values.image.tag      # Nested value
.Files                 # Files in chart
.Capabilities          # Kubernetes API capabilities
```

---

## Helper Template (_helpers.tpl)

Reusable template functions:

```yaml
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := .Chart.Name }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

Use with: `\{{ include "myapp.fullname" . \}}`

---

## Chart Testing

```bash
# Validate chart
helm lint myapp-chart

# Render templates locally
helm template myapp-chart

# Test install (dry-run)
helm install myapp myapp-chart --dry-run --debug

# Validate rendered YAML
helm template myapp myapp-chart | kubectl apply --dry-run=client -f -
```

---

## Create New Chart

```bash
# Generate chart scaffold
helm create myapp-chart

# This creates:
# myapp-chart/
# ├── Chart.yaml
# ├── values.yaml
# └── templates/
#     ├── deployment.yaml
#     ├── service.yaml
#     └── ...
```

---

## Chart Dependencies

If your chart needs MySQL:

```yaml
# Chart.yaml
dependencies:
- name: mysql
  version: 8.0.0
  repository: https://charts.bitnami.com/bitnami

# Install dependencies
helm dependency update myapp-chart

# Pass values to dependency
helm install myapp myapp-chart \
  --set mysql.enabled=true \
  --set mysql.auth.password=mypassword
```

---

## Real-World Example

```yaml
# values.yaml for production
namespace: production

image:
  repository: registry.company.com/myapp
  tag: "1.2.3"
  pullPolicy: Always

replicaCount: 5

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1
    memory: 1Gi

service:
  type: LoadBalancer
  port: 443
  targetPort: 8443

ingress:
  enabled: true
  host: myapp.company.com
  tls:
    enabled: true
    secretName: myapp-tls

env:
  LOG_LEVEL: ERROR
  DATABASE_URL: postgresql://prod-db:5432/myapp

mysql:
  enabled: true
  auth:
    rootPassword: very-secure-password
    database: myapp-prod
```

---

## Summary

Helm charts:

**Structure:**
- Chart.yaml (metadata)
- values.yaml (defaults)
- templates/ (manifests)
- charts/ (dependencies)

**Templates:**
- Jinja-like syntax
- Variables: `\{{ .Values.xxx \}}`
- Conditionals: `\{{ if \}}`
- Loops: `\{{ range \}}`

**Functions:**
- Helpers for reuse
- Built-in functions
- Custom definitions

**Next:** File 02 covers chart structure in detail.
