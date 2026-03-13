# ConfigMap as a Volume

## Introduction

Mounting a ConfigMap as a volume creates files inside the container. Each key in the ConfigMap becomes a filename, and the value becomes the file content. This is the right approach for configuration files that applications read from disk — Nginx config, application settings in JSON or YAML, scripts, and certificate bundles.

The key advantage over environment variable injection: **file mounts update automatically** when the ConfigMap changes, without restarting the Pod.

---

## Basic ConfigMap Volume Mount

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        volumeMounts:
        - name: nginx-config       # references the volume name below
          mountPath: /etc/nginx    # directory inside the container
          readOnly: true           # good practice — config should not be writable
      
      volumes:
      - name: nginx-config         # volume name
        configMap:
          name: nginx-config-cm    # ConfigMap name
```

The ConfigMap `nginx-config-cm` must have `nginx.conf` as a key:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config-cm
  namespace: production
data:
  nginx.conf: |
    user nginx;
    worker_processes auto;
    events { worker_connections 1024; }
    http {
        server {
            listen 80;
            location /healthz { return 200 "ok\n"; }
            location / { proxy_pass http://localhost:8080; }
        }
    }
```

After mounting, the container has:
```
/etc/nginx/nginx.conf     ← content of the 'nginx.conf' key
```

---

## Mounting Multiple Files from One ConfigMap

If the ConfigMap has multiple keys, all of them become files in the mounted directory:

```yaml
data:
  nginx.conf: |
    ... main nginx config ...
  
  default.conf: |
    ... server block config ...
  
  mime.types: |
    ... mime type mappings ...
```

After mounting at `/etc/nginx/`, the container sees:
```
/etc/nginx/nginx.conf
/etc/nginx/default.conf
/etc/nginx/mime.types
```

---

## Mounting a Specific Key to a Specific File Path — `items`

By default, ALL keys in the ConfigMap are mounted as files. Use the `items` field to mount only specific keys, and control the filename:

```yaml
volumes:
- name: app-config
  configMap:
    name: app-config-cm
    items:
    - key: app-settings.json     # ConfigMap key
      path: settings.json        # filename in the mounted directory
    - key: startup.sh
      path: startup.sh
      mode: 0755                 # file permissions (executable script)
    # The LOG_LEVEL and PORT keys in the ConfigMap are NOT mounted as files
```

After mounting at `/etc/app/`:
```
/etc/app/settings.json    ← content of 'app-settings.json' key
/etc/app/startup.sh       ← executable, content of 'startup.sh' key
```

---

## Mounting a ConfigMap Key as a Single File — `subPath`

By default, mounting a ConfigMap to a directory replaces the entire directory contents. If you need to mount a single ConfigMap key as a file inside an existing directory (without replacing other files), use `subPath`:

```yaml
spec:
  containers:
  - name: web
    image: web-app:v1.0
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf    # mount as a single file
      subPath: nginx.conf                 # use this key from the ConfigMap volume
      readOnly: true
  
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config-cm
```

With `subPath`:
- `/etc/nginx/nginx.conf` is replaced with the ConfigMap key's content
- All other files in `/etc/nginx/` are untouched (e.g., the container's built-in mime.types, fastcgi_params)

> ⚠️ **Warning:** When using `subPath`, the file does **NOT** update automatically when the ConfigMap changes. `subPath` mounts are static — they behave like env var injection in this regard. Only directory mounts (without `subPath`) auto-update.

---

## ConfigMap Default File Permissions

Mounted ConfigMap files have a default permission of `0644` (readable by owner and group, not executable). Change this with the `defaultMode` field:

```yaml
volumes:
- name: scripts-config
  configMap:
    name: startup-scripts
    defaultMode: 0755    # make all files executable
```

Or per-file with `mode` in the `items` list:

```yaml
volumes:
- name: app-config
  configMap:
    name: app-config-cm
    defaultMode: 0644    # default: read-only
    items:
    - key: startup.sh
      path: startup.sh
      mode: 0755         # this specific file is executable
    - key: settings.json
      path: settings.json
      # inherits defaultMode: 0644
```

---

## Automatic Updates — How File Mounts Refresh

This is the most important operational feature of volume-mounted ConfigMaps. When you update a ConfigMap, the mounted files inside running Pods update automatically — typically within 60–90 seconds (the kubelet's configmap sync period).

```bash
# Update the ConfigMap
kubectl apply -f updated-nginx-config.yaml

# Watch the file update inside the running Pod (no restart needed)
kubectl exec -it nginx-pod -n production -- watch -n 5 cat /etc/nginx/nginx.conf

# The file content changes within ~60 seconds of the ConfigMap update
```

**How it works:**
1. kubelet polls the API Server for ConfigMap updates every 60 seconds (default)
2. When it detects a change, it updates the files in the mount
3. Kubernetes uses atomic symlink swapping — the update is instant and consistent (no partial file reads)

```bash
# Verify the symlink structure inside the Pod (explains the atomic update)
kubectl exec -it nginx-pod -n production -- ls -la /etc/nginx/

# drwxr-xr-x  ..data -> ..2024_01_15_10_30_00.abcdef123456
# lrwxrwxrwx  nginx.conf -> ..data/nginx.conf
# (atomic: '..data' symlink is swapped atomically when ConfigMap changes)
```

### Important: Application Must Reload After File Update

Updating the files does not automatically reload the application. Your application must:
- Watch for file changes and reload (Nginx does this with `nginx -s reload`)
- Use a sidecar that watches files and sends SIGHUP to the main process
- Be stateless and accept config from file on each request

```bash
# Trigger Nginx to reload its config after ConfigMap update
kubectl exec -it nginx-pod -n production -- nginx -s reload
# Nginx gracefully reloads — no dropped connections
```

---

## Complete Example — Application with Config File

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
  namespace: production
data:
  settings.json: |
    {
      "server": {
        "port": 8080,
        "timeout": 30
      },
      "database": {
        "host": "postgres-service",
        "port": 5432,
        "name": "myapp",
        "poolSize": 20
      },
      "logging": {
        "level": "warn",
        "format": "json"
      }
    }
---
# Deployment uses it
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
spec:
  template:
    spec:
      containers:
      - name: api
        image: api-server:v2.0
        env:
        - name: CONFIG_FILE
          value: "/etc/app/settings.json"    # application reads from this path
        volumeMounts:
        - name: app-settings
          mountPath: /etc/app
          readOnly: true
      
      volumes:
      - name: app-settings
        configMap:
          name: app-settings
```

---

## Summary

- ConfigMap volume mounts create files inside containers — each ConfigMap key becomes a filename, each value becomes the file content
- Use `items` to mount only specific keys and control the filename; use `mode` to control file permissions (use `0755` for executable scripts)
- Use `subPath` to mount a single file without replacing the entire directory — but note that `subPath` mounts do NOT auto-update when the ConfigMap changes
- **Directory mounts auto-update** within ~60 seconds when the ConfigMap is updated — Kubernetes uses atomic symlink swapping for consistency
- Auto-update changes the file content but does NOT reload the application — the app must be designed to watch for file changes or accept a signal to reload

This completes Chapter 9 — ConfigMaps. You now understand how to create, manage, and inject ConfigMaps into Pods as env vars and as files.

In Chapter 10, `01-what-are-secrets.md`, you will learn about Kubernetes Secrets — similar to ConfigMaps but designed for sensitive data like passwords, API keys, and TLS certificates.
