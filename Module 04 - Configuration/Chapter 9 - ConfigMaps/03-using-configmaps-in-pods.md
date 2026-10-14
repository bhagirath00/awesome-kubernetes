# Using ConfigMaps in Pods

## Introduction

Once you have a ConfigMap, you need to get its data into your Pods. Kubernetes provides two ways: inject data as **environment variables** (for simple key-value configuration) or mount data as **files inside the container** (for configuration files). This file covers the environment variable approach — file mounting is covered in the next file.

---

## Method 1: Inject a Single Key as an Environment Variable

Use `valueFrom.configMapKeyRef` to inject a specific key from a ConfigMap as one environment variable:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: web
        image: web-app:v1.0
        env:
        - name: LOG_LEVEL           # env var name in the container
          valueFrom:
            configMapKeyRef:
              name: app-config      # ConfigMap name
              key: LOG_LEVEL        # key within the ConfigMap
        
        - name: APP_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: PORT
        
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
        
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: REDIS_HOST
              optional: true        # Pod still starts if this key doesn't exist
```

The `optional: true` field means the Pod starts even if the ConfigMap or the specific key does not exist. Without it, a missing ConfigMap or key causes the Pod to fail with `CreateContainerConfigError`.

```bash
# Verify the env vars are set inside the Pod
kubectl exec -it web-pod-abc123 -n production -- env | grep -E "LOG_LEVEL|APP_PORT|DATABASE"

# Output:
# LOG_LEVEL=warn
# APP_PORT=8080
# DATABASE_HOST=postgres-service.production.svc.cluster.local
```

---

## Method 2: Inject All Keys as Environment Variables — `envFrom`

Instead of mapping keys one by one, inject every key in a ConfigMap as an environment variable using `envFrom`:

```yaml
spec:
  containers:
  - name: web
    image: web-app:v1.0
    envFrom:
    - configMapRef:
        name: app-config       # inject ALL keys from app-config
        optional: false        # Pod fails if ConfigMap doesn't exist (default behavior)
    - configMapRef:
        name: feature-flags    # also inject all keys from feature-flags ConfigMap
```

Every key in `app-config` becomes an environment variable with the same name. If `app-config` has:
```
LOG_LEVEL=warn
PORT=8080
DATABASE_HOST=postgres-service
```

The container gets all three as environment variables automatically.

### Adding a Prefix with `envFrom`

If multiple ConfigMaps have overlapping key names, use a prefix to avoid collisions:

```yaml
spec:
  containers:
  - name: web
    envFrom:
    - prefix: "APP_"           # all keys prefixed with APP_
      configMapRef:
        name: app-config
    - prefix: "DB_"            # all keys prefixed with DB_
      configMapRef:
        name: database-config
```

If `app-config` has `PORT=8080` and `database-config` also has `PORT=5432`, using prefixes avoids the collision:
- `APP_PORT=8080`
- `DB_PORT=5432`

---

## Method 3: Mixing ConfigMap and Secret Values

Real applications need both configuration (ConfigMap) and sensitive values (Secrets). Mix both in the `env` block:

```yaml
spec:
  containers:
  - name: api
    image: api-server:v2.0
    env:
    # Non-sensitive from ConfigMap
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_HOST
    - name: DATABASE_PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: DATABASE_PORT
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    
    # Sensitive from Secret
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: api-credentials
          key: api-key

    # Or use envFrom to bulk-load both
    envFrom:
    - configMapRef:
        name: app-config          # all non-sensitive config
    - secretRef:
        name: app-secrets         # all sensitive config
```

---

## Using ConfigMap Values in Command-Line Arguments

ConfigMap values can also be used in container command-line arguments via `$(VAR_NAME)` substitution:

```yaml
spec:
  containers:
  - name: app
    image: my-app:v1.0
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
    - name: PORT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: PORT
    
    command: ["/app/server"]
    args:
    - "--log-level=$(LOG_LEVEL)"    # substitutes env var value
    - "--port=$(PORT)"
    - "--env=production"
```

Kubernetes performs `$(VAR_NAME)` substitution in `args` and `command` fields using the env vars defined in the same container spec.

---

## Verifying ConfigMap Injection

```bash
# Check if ConfigMap exists and has the right data
kubectl get configmap app-config -n production -o yaml

# Check if env vars are set in the Pod
kubectl exec -it web-pod-abc123 -n production -- env

# Check a specific variable
kubectl exec web-pod-abc123 -n production -- printenv DATABASE_HOST
# postgres-service.production.svc.cluster.local

# If Pod failed to start due to missing ConfigMap:
kubectl describe pod web-pod-abc123 -n production | grep -A 5 "Events:"
# Warning  Failed  CreateContainerConfigError: couldn't find key LOG_LEVEL in ConfigMap production/app-config
```

---

## What Happens When a ConfigMap Is Updated

**Environment variable injection** — does NOT update automatically:

```bash
# Update the ConfigMap
kubectl patch configmap app-config -n production \
  --type=merge -p='{"data":{"LOG_LEVEL":"debug"}}'

# The running Pod STILL has LOG_LEVEL=warn — env vars are read at startup only
kubectl exec web-pod -n production -- printenv LOG_LEVEL
# warn  ← old value

# Trigger a rollout to restart Pods and pick up the new value
kubectl rollout restart deployment/web-app -n production

# New Pods read the updated ConfigMap
kubectl exec new-web-pod -n production -- printenv LOG_LEVEL
# debug  ← new value
```

This behavior — env vars being static after Pod start — is why many teams use **file-based ConfigMap mounting** for configuration that changes frequently. Files in volumes update automatically within the kubelet sync period (~60 seconds).

---

## ConfigMap Key Name Constraints

When using `envFrom` (injecting all keys as env vars), ConfigMap key names must be valid environment variable names. Invalid env var names are silently skipped.

Valid env var names:
- Letters, digits, underscores only
- Cannot start with a digit
- Uppercase is conventional but lowercase works

```yaml
# These keys work fine as env var names
data:
  LOG_LEVEL: "warn"         # ✅ valid
  PORT: "8080"              # ✅ valid
  DATABASE_HOST: "..."      # ✅ valid

# These keys are valid in ConfigMap but CANNOT be used with envFrom
data:
  log-level: "warn"         # ❌ hyphens not allowed in env var names
  9PORT: "8080"             # ❌ cannot start with digit
  app.config: "..."         # ❌ dots not allowed in env var names
```

For keys with hyphens or dots, use `configMapKeyRef` and specify a valid env var name:

```yaml
env:
- name: LOG_LEVEL             # valid env var name
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: log-level          # ConfigMap key can have hyphens
```

---

## Summary

- `env[].valueFrom.configMapKeyRef` injects a single ConfigMap key as one env var — precise but verbose for many keys
- `envFrom[].configMapRef` injects all keys from a ConfigMap as env vars — convenient but requires valid env var key names
- Use the `prefix` field with `envFrom` to namespace env vars from multiple ConfigMaps and prevent collisions
- Mix `configMapRef` and `secretRef` in `envFrom` to load all config and secrets in two lines
- Use `$(VAR_NAME)` syntax in `command` and `args` fields to substitute env var values into container startup commands
- Env var injection from ConfigMaps is **static** — changes to the ConfigMap require Pod restarts to take effect; use `kubectl rollout restart`

In the next file, `04-configmap-as-volume.md`, you will learn how to mount ConfigMaps as files inside containers — the approach that enables dynamic config updates without Pod restarts.
