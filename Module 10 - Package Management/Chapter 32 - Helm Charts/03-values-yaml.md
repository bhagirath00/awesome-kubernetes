# values.yaml Deep Dive

## Overview

values.yaml defines default configuration that users can override:

```bash
# Uses defaults
helm install myapp chart

# Override value
helm install myapp chart --set replicas=5

# Override from file
helm install myapp chart -f custom-values.yaml

# Multiple overrides
helm install myapp chart \
  -f values-staging.yaml \
  -f secrets.yaml \
  --set image.tag=v2.0.0
```

---

## Merge Strategy

Values are merged in order:

```
1. Chart defaults (values.yaml)
   ↓ Merged with
2. Repository values (if using parent chart)
   ↓ Merged with
3. User file (-f custom-values.yaml)
   ↓ Merged with
4. CLI overrides (--set key=value)
   ↓ Result: Final values
```

CLI overrides have highest priority.

---

## Value Types

### Strings

```yaml
appName: my-application
version: "1.2.3"              # Quote for numeric strings
message: "hello world"
database_url: "postgresql://localhost"
```

### Numbers

```yaml
replicas: 3
port: 8080
timeout: 30
memory_limit: 512
cpu_request: 100
percentage: 75.5
```

### Booleans

```yaml
enabled: true
debug: false
createDatabase: yes
deleteOnTermination: no
```

### Arrays/Lists

```yaml
# List syntax
namespaces:
  - default
  - kube-system
  - monitoring

# Or
names: [alice, bob, charlie]
```

### Maps/Objects

```yaml
# Nested object
database:
  host: localhost
  port: 5432
  username: app
  password: secret
  options:
    timeout: 30
    retries: 3

# Access in template
{{ .Values.database.host }}
```

---

## Environment-Specific Values

Create override files:

```bash
# values-dev.yaml
replicas: 1
resources:
  requests:
    memory: 128Mi
ingress:
  enabled: false

# values-prod.yaml
replicas: 5
resources:
  requests:
    memory: 1Gi
ingress:
  enabled: true
```

Use them:

```bash
# Development
helm install myapp chart -f values-dev.yaml

# Production
helm install myapp chart -f values-prod.yaml
```

---

## Conditional Values

Use conditionals in templates:

```yaml
# values.yaml
mysql:
  enabled: true
  auth:
    rootPassword: password
    database: myapp

redis:
  enabled: false
  replicas: 3
```

Template:

```yaml
# deployment.yaml template
{{- if .Values.mysql.enabled }}
env:
- name: DATABASE_URL
  value: mysql://{{ .Values.mysql.auth.username }}@mysql:3306
{{- end }}

{{- if .Values.redis.enabled }}
- name: REDIS_URL
  value: redis://redis:6379
{{- end }}
```

---

## Default Values Pattern

```yaml
# Don't leave empty, always provide defaults
image:
  repository: myapp          # Not empty
  tag: "1.0.0"              # Not empty
  pullPolicy: IfNotPresent  # Not empty

resources:
  requests:
    cpu: 100m               # Defaults matter
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# All have defaults, never break on missing values
```

---

## Secrets in values.yaml

Never commit secrets:

```yaml
# WRONG — commits password to git
database:
  password: "super-secret-password"

# RIGHT — leave empty
database:
  password: ""

# User provides secrets separately
helm install myapp chart \
  --set database.password=$DB_PASSWORD
```

Or use separate secrets file:

```bash
# secrets.yaml (not in git)
database:
  password: "super-secret"

# Install
helm install myapp chart \
  -f values.yaml \
  -f secrets.yaml
```

---

## Complex Example

```yaml
# Application config
app:
  name: my-app
  version: 1.0.0
  environment: production
  
  # Server settings
  server:
    host: 0.0.0.0
    port: 8080
    enableDebug: false
    
    # TLS
    tls:
      enabled: true
      certPath: /etc/tls/cert.pem
      keyPath: /etc/tls/key.pem

# Deployment config
deployment:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

# Resource allocation
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi

# Database
database:
  type: postgresql
  host: postgres.default.svc
  port: 5432
  name: myapp
  username: app
  password: ""  # Provide separately
  poolSize: 10
  connectionTimeout: 30

# Cache
redis:
  enabled: true
  host: redis.default.svc
  port: 6379
  ttl: 3600

# External API
externalAPI:
  enabled: true
  endpoint: https://api.example.com
  timeout: 10
  retries: 3
  headers:
    Authorization: ""  # Provide separately
    User-Agent: myapp/1.0

# Observability
observability:
  logging:
    level: INFO
    format: json
  metrics:
    enabled: true
    port: 9090
  tracing:
    enabled: false
    endpoint: ""
```

---

## Validation

values.schema.json (Helm 3.7+):

```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "replicas"],
  "properties": {
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "pattern": "^[a-z0-9]"
        },
        "tag": {
          "type": "string"
        }
      }
    },
    "replicas": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100
    }
  }
}
```

Validation runs during install:

```bash
helm install myapp chart -f invalid-values.yaml
# Error: image.repository must match pattern
```

---

## Tips for Good values.yaml

1. **Provide Sensible Defaults**
   - Every value should have default
   - Don't leave critical values empty

2. **Document Everything**
   - Comments explaining purpose
   - Examples of values
   - When to override

3. **Group Related Values**
   - database, cache, external-api
   - Not scattered randomly

4. **Use Consistent Naming**
   - enabled (not enabled flag)
   - replicas (not replica count)
   - Consistent across chart

5. **Provide Environment-Specific Files**
   - values-dev.yaml
   - values-staging.yaml
   - values-prod.yaml

---

## Summary

values.yaml:

**Purpose:**
- Default configuration
- User-overridable values
- Environment-specific configs

**Types:**
- Strings, numbers, booleans
- Arrays and objects
- Conditionals

**Patterns:**
- Never hardcode secrets
- Provide sensible defaults
- Group related values
- Document thoroughly

**Next:** File 04 covers templates in detail.
