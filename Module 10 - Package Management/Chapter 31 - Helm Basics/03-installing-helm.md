# Installing and Configuring Helm

## System Requirements

- Kubernetes cluster
- kubectl configured
- Internet access (to download charts)
- 50MB disk space (for Helm)

---

## Install Helm

### Linux/Mac

```bash
# Download and install
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version

# Output:
# version.BuildInfo{Version:"v3.12.0", ...}
```

### Windows (PowerShell)

```powershell
# Using Chocolatey
choco install kubernetes-helm

# Using Scoop
scoop install helm

# Manual download
# 1. Download from https://github.com/helm/helm/releases
# 2. Extract
# 3. Add to PATH
```

### macOS

```bash
# Using Homebrew
brew install helm

# Verify
helm version
```

---

## Initialize Helm

```bash
# Create default directories
helm repo update

# Check version
helm version

# Get help
helm help
```

---

## Add Chart Repositories

Official repositories:

```bash
# Bitnami (most charts here)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Official Helm charts
helm repo add stable https://charts.helm.sh/stable

# Prometheus community
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

# Grafana
helm repo add grafana https://grafana.github.io/helm-charts

# Update all repos
helm repo update
```

---

## Search for Charts

```bash
# Search in all repos
helm search repo postgres

# Search specific repo
helm search repo bitnami/postgres

# List all available versions
helm search repo bitnami/postgres --versions

# View chart details
helm inspect chart bitnami/postgres
```

---

## Install First Chart

```bash
# Install PostgreSQL from Bitnami
helm install my-database bitnami/postgresql \
  --namespace databases \
  --create-namespace \
  --set postgresqlPassword=mypassword \
  --set persistence.size=50Gi

# Verify installation
helm list -n databases

# Check status
helm status my-database -n databases

# Get connection info
helm get values my-database -n databases
```

---

## Configuration Files

### kubeconfig

Helm uses your kubeconfig:

```bash
# Set context before installing
kubectl config use-context my-cluster

# Helm uses same context
helm install app chart

# Or specify explicitly
helm install app chart --kubeconfig=/path/to/config

# View current context
helm repo list
```

### values.yaml Customization

```bash
# View default values
helm show values bitnami/postgresql

# Save to file and edit
helm show values bitnami/postgresql > myvalues.yaml

# Install with custom values
helm install mydb bitnami/postgresql -f myvalues.yaml
```

---

## Common Installation Scenarios

### Scenario 1: Deploy with Custom Image

```bash
# Install Nginx with custom image
helm install my-nginx bitnami/nginx \
  --set image.repository=myregistry.azurecr.io/nginx \
  --set image.tag=custom-v1 \
  --set replicaCount=3
```

### Scenario 2: Deploy with Persistence

```bash
# Install PostgreSQL with persistent volume
helm install mydb bitnami/postgresql \
  --set persistence.enabled=true \
  --set persistence.size=100Gi \
  --set persistence.storageClassName=fast-ssd \
  --set postgresqlPassword=secure-password
```

### Scenario 3: Deploy with Ingress

```bash
# Install application with ingress
helm install myapp mychart \
  --set ingress.enabled=true \
  --set ingress.hostname=myapp.example.com \
  --set ingress.tls.enabled=true \
  --set ingress.tls.secretName=myapp-tls
```

---

## Helm Configuration

### Default Namespace

```bash
# Install to specific namespace
helm install myapp chart -n production

# Create namespace if missing
helm install myapp chart -n production --create-namespace
```

### Helm Home Directory

```bash
# Default: ~/.helm (or $XDG_CONFIG_HOME/helm on Linux)
# View
echo $HELM_HOME

# Override
HELM_CONFIG_HOME=/custom/path helm install myapp chart
```

---

## Verification

```bash
# List all releases
helm list -A

# Check specific release
helm status myapp

# View manifests that will be created
helm template myapp chart

# Dry-run (simulate install)
helm install myapp chart --dry-run --debug
```

---

## Troubleshooting Installation

### Issue 1: Chart Not Found

```bash
# Error: chart not found
helm search repo nonexistent/chart
# Error: no results found

# Fix: Update repos
helm repo update

# Find correct name
helm search repo chart-name
```

### Issue 2: Insufficient Permissions

```bash
# Error: permission denied
helm install myapp chart -n production

# Fix: Ensure RBAC permissions
kubectl create clusterrolebinding helm-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=default:helm
```

### Issue 3: PVC Not Binding

```bash
# Error: PersistentVolumeClaim pending
helm install mydb postgresql

# Check PVC status
kubectl get pvc

# Fix: Create storage class
kubectl apply -f storageclass.yaml
```

---

## Best Practices

1. **Use Namespaces**
   ```bash
   helm install myapp chart -n production
   ```

2. **Version Control Values**
   ```bash
   git commit values-prod.yaml
   helm install myapp chart -f values-prod.yaml
   ```

3. **Dry-Run Before Installing**
   ```bash
   helm install myapp chart --dry-run --debug
   ```

4. **Use Specific Chart Versions**
   ```bash
   helm install myapp chart --version 1.2.3
   ```

5. **Create Release Backups**
   ```bash
   helm get values myapp > backup-values.yaml
   ```

---

## Summary

Installing Helm:

**Installation:**
- Linux/Mac/Windows
- One-line install
- Verify with `helm version`

**Repository Management:**
- Add repos (Bitnami, Prometheus, etc.)
- Update regularly
- Search for charts

**Install Chart:**
- `helm install release chart`
- Customize with values
- Verify with `helm status`

**Configuration:**
- Namespace selection
- Custom values files
- kubeconfig context

**Next:** File 04 covers Helm commands in detail.
