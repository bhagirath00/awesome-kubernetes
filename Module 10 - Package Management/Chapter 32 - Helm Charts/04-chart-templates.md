# Chart Templates

## Template Syntax

Helm uses Go template language:

```yaml
# Variable substitution
name: {{ .Release.Name }}

# Conditionals
{{- if .Values.ingress.enabled }}
...
{{- end }}

# Loops
{{- range .Values.env }}
- {{ . }}
{{- end }}

# Functions
message: {{ .Values.appName | upper }}

# Comments
{{- /* This is a template comment */ }}
```

---

## Common Variables

```
.Release.Name           # Release name
.Release.Namespace      # Namespace
.Release.Service        # "Helm" (static)
.Release.IsUpgrade      # true if upgrade
.Release.IsInstall      # true if install

.Chart.Name             # Chart name
.Chart.Version          # Chart version
.Chart.AppVersion       # App version
.Chart.Description      # Description
.Chart.Type             # "application" or "library"
.Chart.ApiVersion       # "v2"

.Values                 # All values from values.yaml
.Values.image.tag       # Nested value
.Values.replicas        # Simple value

.Files                  # Chart files
.Capabilities           # K8s capabilities
.Template               # Current template metadata
.Subcharts             # Dependency charts
```

---

## Variable Access Patterns

```yaml
# Simple variable
replicas: {{ .Values.replicas }}

# Nested variable
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}

# With default value (if missing)
tag: {{ .Values.image.tag | default "latest" }}

# Quoted string (for YAML safety)
name: {{ .Values.appName | quote }}

# Convert to different format
replicas: {{ .Values.replicaCount | toString }}

# Multiply
cpuLimit: {{ .Values.cpuRequest | mul 5 }}m
```

---

## Whitespace Control

Trim whitespace with `-`:

```yaml
# WRONG — extra blank lines
metadata:
  name: {{ .Release.Name }}

  labels:
    app: web

# RIGHT — clean output
metadata:
  name: {{ .Release.Name }}
  {{- if .Values.labels }}
  labels:
    {{- toYaml .Values.labels | nindent 4 }}
  {{- end }}
```

### Whitespace Operators

```
{{  }}    # Normal spaces preserved
{{- }}    # Remove whitespace before
{{ -}}    # Remove whitespace after
{{- -}}   # Remove both
```

---

## Conditionals

```yaml
# Simple if
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# If-else
{{- if .Values.mysql.enabled }}
env:
- name: DATABASE_TYPE
  value: mysql
{{- else }}
env:
- name: DATABASE_TYPE
  value: postgresql
{{- end }}

# Multiple conditions (and)
{{- if and .Values.ingress.enabled .Values.ingress.tls }}
...
{{- end }}

# Multiple conditions (or)
{{- if or .Values.redis.enabled .Values.memcached.enabled }}
...
{{- end }}

# Not operator
{{- if not .Values.debug }}
env:
- name: PRODUCTION
  value: "true"
{{- end }}

# Compare values
{{- if eq .Values.environment "production" }}
...
{{- end }}
```

---

## Loops

```yaml
# Loop over list
{{- range .Values.namespaces }}
- {{ . }}
{{- end }}

# Loop with index
{{- range $index, $value := .Values.items }}
- name: item-{{ $index }}
  value: {{ $value }}
{{- end }}

# Loop over map keys
{{- range $key, $value := .Values.env }}
- name: {{ $key }}
  value: {{ $value | quote }}
{{- end }}

# Loop with limit
{{- range first 5 .Values.items }}
- {{ . }}
{{- end }}
```

---

## Functions and Pipes

```yaml
# Functions format: function value arguments
# Pipes format: value | function

# String functions
name: {{ .Values.appName | upper }}
lower: {{ .Values.message | lower }}
trimmed: {{ .Values.text | trimAll "\"" }}
replaced: {{ .Values.message | replace "hello" "hi" }}

# Quote for YAML strings
password: {{ .Values.secret | quote }}

# Default value
tag: {{ .Values.imageTag | default "latest" }}

# Required value (error if missing)
token: {{ .Values.apiToken | required "API token required" }}

# List functions
count: {{ len .Values.items }}
first: {{ first .Values.items }}
last: {{ last .Values.items }}

# Math functions
doubled: {{ .Values.port | mul 2 }}
half: {{ .Values.memory | div 2 }}

# Type conversions
string: {{ .Values.count | toString }}
integer: {{ .Values.port | int }}
```

---

## Includes and Helpers

```yaml
# Call helper template
labels:
  {{- include "mychart.labels" . | nindent 2 }}

# Include other file
spec:
  {{- include "mychart.podSpec" . | nindent 2 }}

# Include with context
{{- include "mychart.database" (dict "Values" .Values "Release" .Release) }}
```

---

## Template Examples

### Deployment Template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
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

### ConfigMap Template

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}-config
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: |
    {{ $value | nindent 4 }}
  {{- end }}
```

### Conditional Template

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  rules:
  {{- range .Values.ingress.hosts }}
  - host: {{ . }}
    http:
      paths:
      - path: /
        backend:
          service:
            name: {{ include "mychart.fullname" . }}
            port:
              number: {{ .Values.service.port }}
  {{- end }}
{{- end }}
```

---

## Debugging Templates

```bash
# Render templates locally (without installing)
helm template myapp mychart

# Render with custom values
helm template myapp mychart -f values-prod.yaml

# Render specific template
helm template myapp mychart --show-only templates/deployment.yaml

# See rendered manifests before install
helm install myapp mychart --dry-run --debug

# Validate YAML output
helm template myapp mychart | kubectl apply --dry-run=client -f -
```

---

## Best Practices

1. **Use Helpers**
   - Define in _helpers.tpl
   - Reuse consistently
   - Avoid repetition

2. **Clean Whitespace**
   - Use `\{{-` and `-\}}`
   - Proper nindent for indentation

3. **Quote Strings**
   - Prevent YAML parsing issues
   - Use | quote or quote function

4. **Document Complex Logic**
   - Comments explain intent
   - Keep templates readable

5. **Validate Output**
   - Always test with `--dry-run`
   - Check rendered YAML

---

## Summary

Chart templates:

**Syntax:**
- `\{{ \}}` for expressions
- `\{{- -\}}` for whitespace control
- Go template language

**Variables:**
- .Release.*, .Chart.*, .Values.*
- Nested access with dots
- Default values with |

**Control:**
- Conditionals: if/else
- Loops: range
- Functions: upper, lower, quote, etc.

**Next:** File 05 covers template functions.
