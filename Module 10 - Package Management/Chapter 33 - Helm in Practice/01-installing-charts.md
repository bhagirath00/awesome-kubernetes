# Installing Charts from Repositories

## Install Overview

```bash
helm install [release-name] [chart]

# Install
# release-name: What you call this deployment
# chart: Where to get it (repo/name or local path)
```

---

## Basic Installation

```bash
# Install from repository
helm install my-app bitnami/nginx

# Install in specific namespace
helm install my-app bitnami/nginx -n production --create-namespace

# Install with custom name
helm install my-web-server bitnami/nginx

# See what was deployed
helm list -n production
helm status my-app -n production
```

---

## Customize Installation

### Single Value Override

```bash
# Set one value
helm install my-app bitnami/nginx \
  --set replicaCount=5

# Set nested value
helm install my-app bitnami/nginx \
  --set image.repository=custom-nginx

# Set boolean
helm install my-app bitnami/nginx \
  --set ingress.enabled=true
```

### Multiple Overrides

```bash
helm install my-app bitnami/nginx \
  --set replicaCount=5 \
  --set service.type=LoadBalancer \
  --set persistence.enabled=true \
  --set persistence.size=100Gi
```

### Values File

```bash
# Save to file
helm show values bitnami/nginx > my-values.yaml

# Edit file
vim my-values.yaml

# Install with custom values
helm install my-app bitnami/nginx -f my-values.yaml
```

### Multiple Values Files

```bash
# Install with cascading overrides
helm install my-app bitnami/nginx \
  -f values.yaml \
  -f values-prod.yaml \
  -f secrets-prod.yaml

# Merged in order: values.yaml, then values-prod.yaml, then secrets-prod.yaml
```

---

## Real-World Installation Examples

### Example 1: PostgreSQL with Persistence

```bash
# Create values file
cat > postgres-values.yaml <<EOF
auth:
  username: appuser
  password: secure-password
  database: appdb
  postgresPassword: root-password

primary:
  persistence:
    enabled: true
    size: 50Gi
    storageClassName: fast-ssd

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
EOF

# Install
helm install postgres bitnami/postgresql \
  -n databases --create-namespace \
  -f postgres-values.yaml

# Verify
helm status postgres -n databases
```

### Example 2: Redis for Caching

```bash
helm install redis bitnami/redis \
  -n cache --create-namespace \
  --set auth.enabled=true \
  --set auth.password=redis-secret \
  --set replica.replicaCount=3 \
  --set master.persistence.size=10Gi
```

### Example 3: Monitoring Stack

```bash
# Prometheus community all-in-one
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi \
  --set grafana.adminPassword=admin123
```

---

## Dry-Run (Test Before Install)

```bash
# See what would be created
helm install my-app bitnami/nginx --dry-run

# See detailed output
helm install my-app bitnami/nginx --dry-run --debug

# Save to file for review
helm install my-app bitnami/nginx --dry-run --debug > install-preview.yaml

# Validate YAML syntax
helm install my-app bitnami/nginx --dry-run | kubectl apply --dry-run=client -f -
```

---

## Wait for Resources

```bash
# Wait for deployment to be ready
helm install my-app bitnami/nginx --wait

# Wait with timeout
helm install my-app bitnami/nginx --wait --timeout 5m

# Get status while waiting
helm status my-app --show-values
```

---

## Helm Hooks

Execute scripts during install:

```yaml
# Pre-install hook (run before resources created)
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: pre-install
    helm.sh/hook-weight: "-5"
spec:
  template:
    spec:
      containers:
      - name: setup
        image: busybox
        command: ["sh", "-c", "echo 'Setting up...'"]

# Post-install hook (run after resources created)
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: post-install
    helm.sh/hook-weight: "1"
spec:
  template:
    spec:
      containers:
      - name: post-setup
        image: busybox
        command: ["sh", "-c", "echo 'Setup complete'"]
```

Available hooks:
- pre-install, post-install
- pre-upgrade, post-upgrade
- pre-delete, post-delete
- pre-rollback, post-rollback
- test

---

## Installation with Dependencies

```bash
# Chart with database dependency
helm install myapp ./myapp-chart \
  --set mysql.enabled=true \
  --set mysql.auth.rootPassword=root123 \
  --set mysql.auth.password=app123 \
  --set mysql.auth.database=myappdb

# Installs myapp AND mysql
```

---

## Security During Install

```bash
# Use secrets from environment variable
export DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id db-password | jq -r '.SecretString')

helm install myapp ./chart \
  --set database.password=$DB_PASSWORD

# Or from file (not in git)
helm install myapp ./chart -f secrets.yaml

# RBAC: Ensure user has permissions
kubectl auth can-i create deployments --namespace=production
```

---

## Common Issues

### Issue 1: Chart Not Found

```bash
# Error: chart not found
helm install myapp bitnami/nginx
# error: chart not found

# Fix: Add repo first
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install myapp bitnami/nginx
```

### Issue 2: PVC Not Binding

```bash
# Error: PersistentVolumeClaim stuck in Pending
# Check storage class exists
kubectl get sc

# If missing, create one
kubectl apply -f storageclass.yaml
```

### Issue 3: Image Pull Error

```bash
# Error: ErrImagePull
# Check image exists and is accessible
docker pull myregistry/image:tag

# If private registry, create secret
kubectl create secret docker-registry regcred \
  --docker-server=myregistry \
  --docker-username=user \
  --docker-password=pass

# Reference in values
imagePullSecrets:
  - name: regcred
```

---

## Best Practices

1. **Always Dry-Run First**
   ```bash
   helm install app chart --dry-run --debug
   ```

2. **Use Values Files**
   - Version control values
   - Keep secrets separate

3. **Name Releases Meaningfully**
   - prod-postgres (not db1)
   - dev-cache (not c1)

4. **Check Release Status**
   ```bash
   helm status app
   helm get values app
   ```

5. **Document Customization**
   - What values you changed
   - Why each change

---

## Summary

Installing charts:

**Basic:**
- helm install release chart
- -n namespace for namespace
- --dry-run to test

**Customize:**
- --set key=value (single)
- -f values.yaml (file)
- Multiple -f for cascading

**Advanced:**
- --wait (for readiness)
- Hooks (setup/cleanup)
- Dependencies (auto-installed)

**Next:** File 02 covers upgrades.
