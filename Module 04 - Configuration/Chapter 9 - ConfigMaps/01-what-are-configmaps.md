# What Are ConfigMaps

## Introduction

Every application needs configuration — database hostnames, API endpoints, feature flags, log levels, and environment-specific settings. Hardcoding this configuration inside the Docker image is wrong because the same image needs to run in dev, staging, and production with different settings.

**ConfigMaps** solve this. A ConfigMap is a Kubernetes object that stores non-sensitive configuration data as key-value pairs. You define the configuration once in a ConfigMap and inject it into Pods as environment variables, command-line arguments, or files — without changing the container image.

---

## The Problem ConfigMaps Solve

Without ConfigMaps, your container image must bake in configuration:

```dockerfile
FROM node:18
ENV DATABASE_HOST=production-db.mycompany.com   # hardcoded — wrong
ENV LOG_LEVEL=info
COPY app/ /app/
CMD ["node", "/app/server.js"]
```

This creates one image that only works in production. Development needs a different image. Staging needs another.

With ConfigMaps, the image is environment-agnostic:

```dockerfile
FROM node:18
# No configuration baked in
COPY app/ /app/
CMD ["node", "/app/server.js"]
```

Configuration comes from a ConfigMap at runtime:

```yaml
# dev ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  DATABASE_HOST: "dev-db.local"
  LOG_LEVEL: "debug"
  MAX_CONNECTIONS: "5"
---
# production ConfigMap (same name, different namespace)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DATABASE_HOST: "prod-db.us-east-1.rds.amazonaws.com"
  LOG_LEVEL: "warn"
  MAX_CONNECTIONS: "50"
```

The application reads `DATABASE_HOST` from environment — in dev it gets the dev database, in production it gets the production database. Same image. Same application code.

---

## What a ConfigMap Stores

ConfigMaps store data as key-value pairs. The values can be:

**Simple strings** — environment variable values, port numbers, log levels:
```yaml
data:
  LOG_LEVEL: "info"
  PORT: "8080"
  ENVIRONMENT: "production"
  FEATURE_FLAG_NEW_UI: "false"
```

**Multi-line content** — entire configuration files:
```yaml
data:
  nginx.conf: |
    server {
        listen 80;
        server_name _;
        location / {
            proxy_pass http://localhost:8080;
        }
    }
  
  app-settings.json: |
    {
      "database": {
        "poolSize": 20,
        "timeout": 30
      },
      "cache": {
        "ttl": 300
      }
    }
  
  startup-script.sh: |
    #!/bin/bash
    echo "Starting application..."
    export DB_URL="${DATABASE_HOST}:${DATABASE_PORT}/${DATABASE_NAME}"
    exec node server.js
```

---

## ConfigMap vs Secret

| | ConfigMap | Secret |
|---|-----------|--------|
| **For** | Non-sensitive configuration | Sensitive data (passwords, tokens, certs) |
| **Storage** | Plain text in etcd | Base64 encoded in etcd |
| **Encryption at rest** | Not encrypted by default | Encrypted if envelope encryption enabled |
| **Visible in logs** | Yes | Partially masked by Kubernetes |
| **Use for** | Log levels, hostnames, ports, feature flags | Passwords, API keys, TLS certificates |

Never store passwords, API keys, or tokens in a ConfigMap — use Secrets for sensitive data.

---

## ConfigMap Data Limits

ConfigMaps have a 1MB data limit. This is usually more than enough for configuration files. If you need more than 1MB of configuration data, reconsider whether configuration data belongs in Kubernetes objects at all — large blobs are better stored in S3 and fetched via init containers.

---

## ConfigMap Immutability

For Kubernetes 1.21+, you can mark a ConfigMap as immutable. Immutable ConfigMaps:
- Cannot be modified after creation
- Reduce load on the API Server (no need to watch for changes)
- Prevent accidental configuration changes in production

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
  namespace: production
data:
  LOG_LEVEL: "warn"
  PORT: "8080"
immutable: true    # cannot be edited — delete and recreate with a new name to change
```

When using immutable ConfigMaps, adopt a versioned naming convention: `app-config-v1`, `app-config-v2`, etc. Update the Deployment to reference the new version — this forces a Pod rollout.

---

## Viewing and Managing ConfigMaps

```bash
# List all ConfigMaps in a namespace
kubectl get configmaps -n production
# Or short form:
kubectl get cm -n production

# View a ConfigMap's data
kubectl describe configmap app-config -n production

# View as YAML (shows all fields)
kubectl get configmap app-config -n production -o yaml

# Get a specific key's value
kubectl get configmap app-config -n production \
  -o jsonpath='{.data.LOG_LEVEL}'
# Output: warn

# Create from literal values (quick test)
kubectl create configmap test-config \
  --from-literal=KEY1=value1 \
  --from-literal=KEY2=value2 \
  -n production

# Delete a ConfigMap
kubectl delete configmap app-config -n production
```

---

## Real-World ConfigMap Examples

### Web Application Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
  namespace: production
data:
  # Application settings
  APP_ENV: "production"
  APP_PORT: "8080"
  LOG_LEVEL: "warn"
  LOG_FORMAT: "json"

  # Service discovery (internal Kubernetes services)
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "myapp"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"

  # AWS settings
  AWS_REGION: "us-east-1"
  S3_BUCKET: "myapp-uploads-prod"
  SQS_QUEUE_URL: "https://sqs.us-east-1.amazonaws.com/123456789/myapp-jobs"

  # Feature flags
  FEATURE_NEW_CHECKOUT: "true"
  FEATURE_DARK_MODE: "false"
  MAX_UPLOAD_SIZE_MB: "50"
```

### Nginx Configuration File

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: production
data:
  nginx.conf: |
    user nginx;
    worker_processes auto;
    
    events {
        worker_connections 1024;
    }
    
    http {
        access_log /var/log/nginx/access.log;
        error_log  /var/log/nginx/error.log warn;
        
        upstream backend {
            server localhost:8080;
        }
        
        server {
            listen 80;
            server_name _;
            
            location /healthz {
                return 200 "ok\n";
            }
            
            location / {
                proxy_pass         http://backend;
                proxy_set_header   Host $host;
                proxy_set_header   X-Real-IP $remote_addr;
                proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_read_timeout 60s;
            }
        }
    }
```

---

## Summary

- ConfigMaps store non-sensitive configuration data as key-value pairs — simple strings and multi-line file content
- The same Docker image runs in all environments; ConfigMaps provide the environment-specific configuration at runtime
- ConfigMaps can hold individual values (for environment variables) and entire configuration files (for mounting as files into containers)
- Use Secrets for sensitive data (passwords, API keys) — never ConfigMaps
- ConfigMaps have a 1MB data limit — large configuration blobs belong in S3
- Mark ConfigMaps as `immutable: true` in production to prevent accidental changes and reduce API Server load

In the next file, `02-creating-configmaps.md`, you will learn all the ways to create ConfigMaps — from literal values, from files, from directories, and from YAML — with the trade-offs of each approach.
