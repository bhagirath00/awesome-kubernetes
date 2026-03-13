# Creating ConfigMaps

## Introduction

There are four ways to create ConfigMaps in Kubernetes: from literal key-value pairs, from files, from directories, and from YAML manifests. Each approach suits different situations. This file covers all four methods with real examples and explains when to use each.

---

## Method 1: From Literal Values

The fastest way to create a ConfigMap for testing or simple cases:

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=PORT=8080 \
  --from-literal=ENVIRONMENT=production \
  -n production

# Verify
kubectl get configmap app-config -n production -o yaml
```

```yaml
# Output:
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  ENVIRONMENT: production
  LOG_LEVEL: info
  PORT: "8080"
```

**When to use:** Quick tests, one-off configurations. Not recommended for production — configuration should be in version-controlled YAML.

---

## Method 2: From a Single File

Create a ConfigMap from an existing configuration file. The filename becomes the key, the file content becomes the value:

```bash
# Given a file: nginx.conf
cat nginx.conf
# server {
#     listen 80;
#     ...
# }

kubectl create configmap nginx-config \
  --from-file=nginx.conf \
  -n production

# The ConfigMap gets one key: 'nginx.conf'
# Value: the entire file content
```

You can also specify a custom key name:

```bash
# Key name is 'config' instead of 'nginx.conf'
kubectl create configmap nginx-config \
  --from-file=config=nginx.conf \
  -n production
```

---

## Method 3: From a Directory

Create a ConfigMap from all files in a directory. Each filename becomes a key:

```bash
ls config/
# nginx.conf
# app-settings.json
# startup.sh

kubectl create configmap app-configs \
  --from-file=config/ \
  -n production

# Resulting ConfigMap keys: nginx.conf, app-settings.json, startup.sh
kubectl get configmap app-configs -n production -o yaml
```

**When to use:** When your application expects a directory of configuration files (e.g., a `conf.d/` directory that Nginx reads).

---

## Method 4: YAML Manifest (Recommended for Production)

Define ConfigMaps in YAML and store in version control. This is the only approach that gives you:
- Git history of every configuration change
- Code review for config changes
- Ability to apply with CI/CD pipelines

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: web-app
    environment: production
data:
  # Simple key-value pairs
  LOG_LEVEL: "warn"
  PORT: "8080"
  ENVIRONMENT: "production"
  DATABASE_HOST: "postgres-service.production.svc.cluster.local"
  REDIS_HOST: "redis-service"
  AWS_REGION: "us-east-1"

  # Multi-line configuration file
  app-settings.json: |
    {
      "server": {
        "port": 8080,
        "timeout": 30,
        "maxConnections": 100
      },
      "logging": {
        "level": "warn",
        "format": "json",
        "destination": "stdout"
      },
      "cache": {
        "host": "redis-service",
        "port": 6379,
        "ttl": 300
      }
    }

  nginx.conf: |
    server {
        listen 80;
        server_name _;
        
        location /healthz {
            return 200 "ok\n";
        }
        
        location / {
            proxy_pass http://localhost:8080;
            proxy_read_timeout 60s;
        }
    }
```

```bash
kubectl apply -f configmap.yaml

# Update — just edit the YAML and apply again
kubectl apply -f configmap.yaml
# Output: configmap/app-config configured
```

---

## Updating a ConfigMap

```bash
# Method 1: Re-apply the YAML (recommended)
# Edit configmap.yaml, then:
kubectl apply -f configmap.yaml

# Method 2: Edit directly (not recommended for production)
kubectl edit configmap app-config -n production
# Opens in your default editor — save to apply

# Method 3: Patch a specific key
kubectl patch configmap app-config -n production \
  --type='merge' \
  -p='{"data":{"LOG_LEVEL":"debug"}}'

# Verify the update
kubectl get configmap app-config -n production -o jsonpath='{.data.LOG_LEVEL}'
```

> ⚠️ **Warning:** Updating a ConfigMap does NOT automatically restart Pods that use it as environment variables. Pods only read env vars at startup. You must restart the Pods (or rollout the Deployment) to pick up ConfigMap changes used as env vars. ConfigMaps mounted as files update automatically within the kubelet's sync period (~60 seconds).

---

## ConfigMap from Environment File (`--from-env-file`)

If you have an `.env` file format (common in development), you can create a ConfigMap directly from it:

```bash
cat production.env
# LOG_LEVEL=warn
# PORT=8080
# DATABASE_HOST=postgres-service
# AWS_REGION=us-east-1

kubectl create configmap app-config \
  --from-env-file=production.env \
  -n production
```

Each line `KEY=VALUE` in the file becomes a separate key in the ConfigMap. This differs from `--from-file` which stores the entire file content under a single key.

---

## Creating ConfigMaps With Kustomize

Kustomize (built into `kubectl`) can generate ConfigMaps from files and automatically append a hash suffix to the name — forcing a Pod rollout whenever the ConfigMap changes:

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=warn
  - PORT=8080
  files:
  - nginx.conf
  - app-settings.json
  options:
    disableNameSuffixHash: false   # default true — appends hash
```

```bash
kubectl apply -k .

# Creates: configmap/app-config-abc123de (with hash suffix)
# When nginx.conf changes → new hash → new ConfigMap name
# Deployment references the new name → automatic rollout
```

The hash suffix is the key benefit — when you change the config, the ConfigMap gets a new name, and any Deployment referencing it by that name automatically rolls out. No more "pods using stale config" problems.

---

## Organizing ConfigMaps for Multiple Environments

A clean pattern for managing config across environments:

```
kubernetes/
├── base/
│   └── configmap.yaml         ← shared config (same in all environments)
└── overlays/
    ├── production/
    │   └── configmap-patch.yaml  ← production-specific overrides
    ├── staging/
    │   └── configmap-patch.yaml  ← staging-specific overrides
    └── dev/
        └── configmap.yaml        ← dev-specific (can be completely different)
```

```yaml
# base/configmap.yaml — shared defaults
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  PORT: "8080"
  LOG_FORMAT: "json"
  AWS_REGION: "us-east-1"
```

```yaml
# overlays/production/configmap-patch.yaml — production overrides
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "warn"
  DATABASE_HOST: "prod-db.us-east-1.rds.amazonaws.com"
  MAX_CONNECTIONS: "100"
```

```bash
# Apply production configuration
kubectl apply -k overlays/production/ -n production

# Apply dev configuration
kubectl apply -k overlays/dev/ -n dev
```

---

## Summary

- Four ways to create ConfigMaps: from literals (quick tests), from files (single file), from directories (multiple files), from YAML (version-controlled production approach)
- YAML manifests are the recommended production approach — they live in Git, get code reviewed, and are applied by CI/CD pipelines
- `--from-env-file` imports `.env` format files — each `KEY=VALUE` line becomes a separate ConfigMap key
- Kustomize's `configMapGenerator` appends a hash to ConfigMap names — changing the config triggers automatic Pod rollouts
- Updating a ConfigMap does not restart Pods using it as env vars — only file-mounted ConfigMaps update automatically; env var-based ConfigMaps require a Pod restart

In the next file, `03-using-configmaps-in-pods.md`, you will learn how to inject ConfigMap data into Pods as environment variables, with the full walkthrough of `env`, `envFrom`, and the Downward API.
