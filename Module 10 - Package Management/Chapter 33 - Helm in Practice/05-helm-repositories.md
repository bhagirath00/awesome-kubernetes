# Helm Repositories

## What is a Helm Repository?

Repository = Collection of Helm charts

```
Helm Chart Repository
├── Chart 1: nginx
├── Chart 2: postgresql
├── Chart 3: redis
└── Chart N: other-apps

Like npm registry, PyPI, or Docker Hub
```

---

## Official Repositories

```bash
# Bitnami (most comprehensive)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Prometheus Community (monitoring)
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

# Grafana (dashboards)
helm repo add grafana https://grafana.github.io/helm-charts

# Jetstack (cert-manager)
helm repo add jetstack https://charts.jetstack.io

# Official Helm stable (legacy)
helm repo add stable https://charts.helm.sh/stable
```

---

## Manage Repositories

```bash
# Add repository
helm repo add myrepo https://charts.example.com

# List all repositories
helm repo list

# Output:
# NAME                            URL
# bitnami                         https://charts.bitnami.com/bitnami
# prometheus-community            https://prometheus-community.github.io/helm-charts
# myrepo                          https://charts.example.com

# Update repository (refresh chart list)
helm repo update

# Update specific repository
helm repo update bitnami

# Remove repository
helm repo remove bitnami
```

---

## Search Repositories

```bash
# Search all repos for "postgres"
helm search repo postgres

# Output:
# NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
# bitnami/postgresql                      12.1.0          14.5            A powerful, open source...
# bitnami/postgresql-ha                   9.0.0           14.5            PostgreSQL with HA
# myrepo/custom-postgres                  1.0.0           13.0            Custom postgres...

# Search specific repo
helm search repo bitnami/postgres

# Show all versions
helm search repo bitnami/postgres --versions

# Search with version constraints
helm search repo bitnami/postgres --version ">=11 <12"
```

---

## Create Private Repository

### Option 1: ChartMuseum (Recommended)

```bash
# Install ChartMuseum
helm install chartmuseum bitnami/chartmuseum \
  -n chartmuseum --create-namespace \
  --set persistence.enabled=true \
  --set persistence.size=50Gi

# Get URL
kubectl get svc -n chartmuseum

# Add to Helm
helm repo add mycompany http://chartmuseum:8080

# Push chart
curl --data-binary @my-app-chart-1.0.0.tgz \
  http://chartmuseum:8080/api/charts

# Install from private repo
helm install myapp mycompany/my-app-chart
```

### Option 2: GitHub Pages

```bash
# Create gh-pages branch
git checkout --orphan gh-pages
git rm -rf .

# Create index
helm repo index . --url https://github.com/company/helm-charts/tree/gh-pages

# Push charts
git add .
git commit -m "Add helm charts"
git push origin gh-pages

# Add to Helm
helm repo add mycompany https://company.github.io/helm-charts

# Search
helm search repo mycompany
```

### Option 3: S3

```bash
# Upload chart to S3
aws s3 cp my-app-chart-1.0.0.tgz s3://my-charts/

# Create index in S3
aws s3 cp index.yaml s3://my-charts/

# Add S3 repo to Helm
helm repo add mycompany s3://my-charts

# Search
helm search repo mycompany
```

---

## Repository Authentication

### Basic Auth

```bash
# Add repo with credentials
helm repo add myrepo https://charts.example.com \
  --username=user \
  --password=pass

# Or in ~/.helm/repositories.yaml
apiVersion: ""
repositories:
- name: myrepo
  url: https://charts.example.com
  username: user
  password: pass
```

### Certificate Auth

```bash
# Add repo with certificate
helm repo add myrepo https://charts.example.com \
  --ca-file=/path/to/ca.crt \
  --cert-file=/path/to/client.crt \
  --key-file=/path/to/client.key
```

---

## Repository Index

Repository index.yaml file lists all charts:

```yaml
apiVersion: v1
entries:
  nginx:
  - apiVersion: v2
    appVersion: 1.20.0
    created: "2024-01-15T10:00:00Z"
    description: A Helm chart for Nginx
    digest: abc123
    home: https://nginx.org
    icon: https://nginx.org/icon.png
    keywords:
    - web
    - proxy
    name: nginx
    sources:
    - https://github.com/nginx/nginx
    urls:
    - https://charts.example.com/nginx-13.2.0.tgz
    version: 13.2.0
  postgresql:
  - apiVersion: v2
    ...
generated: "2024-01-15T10:00:00Z"
```

---

## Repository Usage Examples

### Example 1: Deploy Monitoring Stack

```bash
# Add repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install monitoring
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

helm install grafana grafana/grafana \
  -n monitoring
```

### Example 2: Multi-Chart Deployment

```bash
# Database + Cache + App
helm install postgres bitnami/postgresql \
  -n production --create-namespace

helm install redis bitnami/redis \
  -n production

helm install myapp ./myapp-chart \
  -n production \
  -f values-prod.yaml
```

### Example 3: Internal Company Charts

```bash
# Add company repo
helm repo add company https://charts.company.com

# Deploy internal microservices
helm install auth-service company/auth-service \
  -n microservices --create-namespace

helm install api-gateway company/api-gateway \
  -n microservices

helm install payment-service company/payment-service \
  -n microservices
```

---

## Helm Hub (Chart Discovery)

```bash
# Search Helm Hub directly
helm search hub nginx

# Or via web
# https://hub.helm.sh

# Find charts by category
# - Monitoring
# - Database
# - Infrastructure
# - Security
```

---

## Repository Best Practices

1. **Organize by Type**
   - Bitnami: General-purpose charts
   - Prometheus-community: Monitoring
   - Company: Internal services

2. **Version Your Charts**
   - semantic versioning (MAJOR.MINOR.PATCH)
   - tag releases in git

3. **Keep Index Updated**
   - Regular helm repo update
   - Check for security updates

4. **Document Repository**
   - README with available charts
   - Links to chart documentation

5. **Secure Private Repos**
   - Use authentication
   - HTTPS only
   - Regular access reviews

---

## Repository Troubleshooting

### Issue 1: Chart Not Found

```bash
# Error: chart not found
helm search repo nonexistent
# Error: no results found

# Fix: Add repository first
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo bitnami/nginx
```

### Issue 2: Outdated Index

```bash
# Old chart versions showing
helm repo update

# Force refresh
helm repo update --force
```

### Issue 3: Authentication Failed

```bash
# Error: 401 Unauthorized
# Check credentials
helm repo remove private
helm repo add private https://charts.company.com \
  --username=user \
  --password=pass
```

---

## Summary

Helm repositories:

**Official Repos:**
- Bitnami (most charts)
- Prometheus-community (monitoring)
- Grafana (dashboards)
- Others by use case

**Management:**
- helm repo add (register)
- helm repo update (refresh)
- helm repo list (view)
- helm search repo (find)

**Private Repos:**
- ChartMuseum (full-featured)
- GitHub Pages (simple)
- S3 (cloud storage)

**Security:**
- Basic auth
- Certificate auth
- HTTPS only

**Module 10 Complete! (14 files total)**

Summary of Module 10 - Package Management (Helm):
- Chapter 31 - Helm Basics (4 files): What is Helm, architecture, installation, commands
- Chapter 32 - Helm Charts (5 files): Chart structure, values, templates, functions
- Chapter 33 - Helm in Practice (5 files): Installing, upgrading, rollback, custom charts, repos

**Modules 9 and 10 Complete!**

Total: 27 files across 6 chapters in 2 modules

Module 9 - Observability (13 files):
- Chapter 28 - Logging (5 files)
- Chapter 29 - Monitoring (5 files)
- Chapter 30 - Tracing (3 files)

Module 10 - Package Management (14 files):
- Chapter 31 - Helm Basics (4 files)
- Chapter 32 - Helm Charts (5 files)
- Chapter 33 - Helm in Practice (5 files)

**Ready for your review!**
