# Helm Commands

## Most Important Commands

### helm repo

Manage chart repositories:

```bash
# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# List repositories
helm repo list

# Update all repos
helm repo update

# Update specific repo
helm repo update bitnami

# Remove repository
helm repo remove bitnami

# Search repo
helm repo search bitnami

# Show values from repo
helm repo search bitnami/postgresql --versions
```

---

## helm search

Find charts:

```bash
# Search all repos
helm search repo postgresql

# Search with version info
helm search repo postgresql --versions

# Search only matching pattern
helm search repo postgres

# Show detailed info
helm search repo bitnami/postgresql
```

---

## helm show

Inspect chart:

```bash
# Show chart metadata
helm show chart bitnami/postgresql

# Show default values
helm show values bitnami/postgresql

# Show README
helm show readme bitnami/postgresql

# Show all info
helm show all bitnami/postgresql > chart-info.txt
```

---

## helm install

Deploy chart:

```bash
# Basic install
helm install my-release bitnami/postgresql

# Install with name
helm install my-db bitnami/postgresql

# Install in namespace
helm install my-db bitnami/postgresql -n databases --create-namespace

# Install with custom values
helm install my-db bitnami/postgresql -f values.yaml

# Install with single value override
helm install my-db bitnami/postgresql \
  --set postgresqlPassword=mypassword

# Install multiple overrides
helm install my-db bitnami/postgresql \
  --set postgresqlPassword=mypassword \
  --set persistence.size=100Gi \
  --set replicaCount=3

# Install specific version
helm install my-db bitnami/postgresql --version 11.0.0

# Dry-run (test without creating)
helm install my-db bitnami/postgresql --dry-run --debug

# Idempotent install (create or update)
helm upgrade --install my-db bitnami/postgresql

# Wait for resources
helm install my-db bitnami/postgresql --wait
```

---

## helm list

View installed releases:

```bash
# List all releases
helm list

# List in specific namespace
helm list -n databases

# List in all namespaces
helm list -A

# Show all including deleted
helm list --all

# Show chart versions
helm list --all-namespaces --long
```

---

## helm status

Check release status:

```bash
# Check release
helm status my-db

# Check in specific namespace
helm status my-db -n databases

# Show detail
helm status my-db --show-values
```

---

## helm upgrade

Update release:

```bash
# Upgrade to new chart version
helm upgrade my-db bitnami/postgresql

# Upgrade with new values
helm upgrade my-db bitnami/postgresql -f new-values.yaml

# Upgrade single value
helm upgrade my-db bitnami/postgresql \
  --set persistence.size=200Gi

# Test first
helm upgrade my-db bitnami/postgresql --dry-run --debug

# Force replacement (delete pod, recreate)
helm upgrade my-db bitnami/postgresql --force

# Keep failed release
helm upgrade my-db bitnami/postgresql --preserve-values
```

---

## helm rollback

Revert to previous version:

```bash
# View release history
helm history my-db

# Rollback to previous release
helm rollback my-db

# Rollback to specific revision
helm rollback my-db 1

# Clean up after rollback
helm rollback my-db 1 --cleanup-on-fail
```

---

## helm uninstall

Delete release:

```bash
# Delete release
helm uninstall my-db

# Delete in specific namespace
helm uninstall my-db -n databases

# Keep release history
helm uninstall my-db --keep-history

# Cascade delete (delete resources)
helm uninstall my-db --cascade=foreground
```

---

## helm get

Retrieve release information:

```bash
# Get all info
helm get all my-db

# Get values (what you set)
helm get values my-db

# Get manifest (rendered YAML)
helm get manifest my-db

# Get hooks
helm get hooks my-db

# Get notes (post-install messages)
helm get notes my-db
```

---

## helm template

Render templates locally:

```bash
# Generate YAML without installing
helm template my-db bitnami/postgresql

# Generate with custom values
helm template my-db bitnami/postgresql -f values.yaml

# Generate to file
helm template my-db bitnami/postgresql > manifest.yaml

# Validate output
helm template my-db bitnami/postgresql | kubectl apply --dry-run=client -f -
```

---

## helm verify

Validate chart:

```bash
# Check chart for errors
helm lint my-chart

# Verify with strict mode
helm lint my-chart --strict

# Check dependencies
helm dependency check my-chart

# Update dependencies
helm dependency update my-chart
```

---

## Real-World Workflows

### Workflow 1: Deploy New App

```bash
# 1. Add repo
helm repo add bitnami https://charts.bitnami.com/bitnami

# 2. Search for chart
helm search repo postgres

# 3. Inspect values
helm show values bitnami/postgresql > values.yaml

# 4. Edit values
vim values.yaml

# 5. Install
helm install mydb bitnami/postgresql -f values.yaml

# 6. Verify
helm status mydb
```

### Workflow 2: Upgrade App

```bash
# 1. Check current release
helm status myapp

# 2. View history
helm history myapp

# 3. Upgrade
helm upgrade myapp myapp-chart --set image.tag=v2.0.0

# 4. Verify
helm status myapp

# 5. If issues, rollback
helm rollback myapp
```

### Workflow 3: Monitor Multiple Environments

```bash
# Dev
helm install dev-app myapp-chart -n dev --create-namespace

# Staging
helm install staging-app myapp-chart -n staging \
  -f values-staging.yaml --create-namespace

# Production
helm install prod-app myapp-chart -n production \
  -f values-prod.yaml -f secrets-prod.yaml --create-namespace

# View all
helm list -A
```

---

## Helm Plugins

Extend Helm with plugins:

```bash
# Install plugin
helm plugin install https://github.com/chartmuseum/helm-push.git

# List plugins
helm plugin list

# Remove plugin
helm plugin uninstall push

# Common plugins:
# - helm-diff: Shows diff before upgrade
# - helm-push: Push to ChartMuseum
# - helm-secrets: Manage secrets
```

---

## Summary

Helm commands:

**Repositories:**
- helm repo add/list/update/remove

**Search/Inspect:**
- helm search repo (find)
- helm show chart/values/readme (inspect)

**Lifecycle:**
- helm install (deploy)
- helm upgrade (update)
- helm rollback (revert)
- helm uninstall (delete)

**Information:**
- helm list (view releases)
- helm status (check status)
- helm get (retrieve info)
- helm history (version history)

**Development:**
- helm template (render locally)
- helm lint (validate)
- helm dependency (manage deps)

**Module 10 Chapter 31 Complete - 4 Files!**

You now understand Helm basics:
- File 01: What is Helm (package manager)
- File 02: Architecture (components, structure)
- File 03: Installation (setup, repos)
- File 04: Commands (all important commands)

**Next:** Chapter 32 covers Helm charts creation.
