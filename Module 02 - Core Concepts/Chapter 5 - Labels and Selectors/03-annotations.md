# Annotations

## Introduction

Annotations look similar to labels — they are also key-value pairs attached to Kubernetes objects. But they serve a completely different purpose. While labels are for **selection and grouping**, annotations are for **storing metadata** that tools, controllers, and humans can read.

Understanding the difference between labels and annotations, and knowing when to use each, is important for keeping your Kubernetes configuration clean and purposeful.

---

## Labels vs Annotations — The Core Difference

| | Labels | Annotations |
|---|--------|-------------|
| **Purpose** | Grouping, selection, organizing | Metadata, configuration, documentation |
| **Queryable?** | ✅ Yes — `kubectl get pods -l app=web` | ❌ No — cannot filter by annotation |
| **Used by selectors?** | ✅ Yes — Services, Deployments use them | ❌ No |
| **Value size** | Max 63 characters | Up to 256 KB per annotation key |
| **Value characters** | Restricted — alphanumeric, `-`, `_`, `.` | Any characters — JSON, URLs, long strings |
| **Who reads them** | Kubernetes itself, controllers | External tools, humans, admission webhooks |

The key rule: **if you need to select resources by it, use a label. If you just need to store information, use an annotation.**

---

## Annotation Syntax

```yaml
metadata:
  annotations:
    key: "value"
```

Annotation keys follow the same prefix rules as label keys. Annotation values can be any string — including JSON, YAML, long URLs, and multi-line text.

```yaml
metadata:
  annotations:
    # Simple strings
    deploy-timestamp: "2024-01-15T10:30:00Z"
    deployed-by: "jenkins-pipeline"
    git-commit: "a1b2c3d4e5f6789"
    contact: "platform-team@mycompany.com"

    # URLs
    runbook: "https://wiki.mycompany.com/runbooks/web-app"
    dashboard: "https://grafana.mycompany.com/d/web-app"
    source-repo: "https://github.com/mycompany/web-app"

    # JSON stored as a string value
    build-metadata: |
      {"buildNumber": "142", "triggeredBy": "github-webhook", "branch": "main"}

    # Long description
    description: |
      Web application frontend serving customer-facing traffic.
      Connects to PostgreSQL RDS and Redis ElastiCache.
      Scales from 3 to 50 replicas based on CPU.
```

---

## How Tools Use Annotations

Annotations are the primary way Kubernetes tools configure behavior without modifying the core API. Here are the most important examples:

### AWS Load Balancer Controller

The AWS Load Balancer Controller uses annotations on Services and Ingress resources to configure how AWS Load Balancers are created:

```yaml
# Service with annotations that configure an AWS NLB (Network Load Balancer)
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
  annotations:
    # Tell the AWS LBC to create an NLB (not a Classic LB)
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"

    # Use an internet-facing (public) NLB
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"

    # Attach to specific subnets
    service.beta.kubernetes.io/aws-load-balancer-subnets: "subnet-abc123,subnet-def456"

    # Enable access logging to S3
    service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "my-lb-logs"

    # Add resource tags to the NLB
    service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: >
      Environment=production,Team=platform,ManagedBy=kubernetes
spec:
  selector:
    app: web-server
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 8080
```

Every annotation here configures a specific aspect of how the NLB is created in your AWS account. Without annotations, you would need separate AWS API calls to configure these features.

### Ingress Annotations for ALB (Application Load Balancer)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: production
  annotations:
    # Use the AWS ALB Ingress Controller
    kubernetes.io/ingress.class: alb

    # Internet-facing ALB
    alb.ingress.kubernetes.io/scheme: internet-facing

    # Route to Pod IPs directly (not NodePort)
    alb.ingress.kubernetes.io/target-type: ip

    # Use HTTPS with ACM certificate
    alb.ingress.kubernetes.io/certificate-arn: >
      arn:aws:acm:us-east-1:123456789:certificate/abc-123-def-456

    # Force HTTP to HTTPS redirect
    alb.ingress.kubernetes.io/ssl-redirect: "443"

    # Health check configuration
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"

    # WAF association
    alb.ingress.kubernetes.io/wafv2-acl-arn: >
      arn:aws:wafv2:us-east-1:123456789:regional/webacl/my-waf/abc123

    # ALB resource tags
    alb.ingress.kubernetes.io/tags: Environment=production,Team=platform
spec:
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Cluster Autoscaler

The Cluster Autoscaler uses annotations to decide whether a node can be safely scaled down:

```yaml
# Prevent a specific node from being removed by Cluster Autoscaler
kubectl annotate node ip-10-0-1-100.ec2.internal \
  cluster-autoscaler.kubernetes.io/scale-down-disabled=true
```

### Argo CD

Argo CD uses annotations to control sync behavior and store application state:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"          # deploy in wave 2 (after wave 1 resources)
    argocd.argoproj.io/hook: PreSync            # run this Job before the sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded  # delete after success
```

### Prometheus Scraping

Prometheus uses annotations to discover which Pods to scrape for metrics:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"        # tell Prometheus to scrape this Pod
    prometheus.io/port: "9090"          # which port to scrape
    prometheus.io/path: "/metrics"      # which path
```

---

## Using Annotations for Documentation and Operations

Annotations are the right place to store operational information directly on the resource — no separate documentation needed:

```yaml
metadata:
  annotations:
    # Who owns this resource
    owner: "backend-team"
    contact: "backend@mycompany.com"
    slack-channel: "#backend-oncall"

    # What this resource does
    description: |
      API gateway service handling all external API requests.
      Rate limiting: 1000 requests/second per IP.
      Timeout: 30 seconds.

    # Operational runbooks
    runbook-general: "https://wiki.mycompany.com/runbooks/api-gateway"
    runbook-high-latency: "https://wiki.mycompany.com/runbooks/api-latency"
    runbook-5xx-errors: "https://wiki.mycompany.com/runbooks/api-errors"

    # Deployment information
    git-repo: "https://github.com/mycompany/api-gateway"
    git-commit: "a1b2c3d4e5f6789abcdef"
    git-branch: "main"
    build-pipeline: "https://jenkins.mycompany.com/job/api-gateway/142"
    deploy-timestamp: "2024-01-15T10:30:00Z"
    deployed-by: "jenkins"

    # Monitoring links
    grafana-dashboard: "https://grafana.mycompany.com/d/api-gateway"
    cloudwatch-logs: >
      https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#logsV2:log-groups/log-group//api-gateway
```

When an on-call engineer wakes up to an alert at 3 AM, they can run `kubectl describe pod <pod-name>` and immediately find the runbook, Grafana dashboard, and contact information — all without searching through separate documentation systems.

---

## Reading and Setting Annotations

```bash
# View annotations for a Pod
kubectl describe pod web-pod -n production

# Annotations section shows:
# Annotations:  deployed-by: jenkins
#               git-commit: a1b2c3d4
#               runbook: https://wiki.mycompany.com/runbooks/web-app

# Get a specific annotation value
kubectl get pod web-pod -n production \
  -o jsonpath='{.metadata.annotations.deployed-by}'

# Output: jenkins

# Add an annotation to a running resource
kubectl annotate pod web-pod -n production \
  incident="INC-2024-001" \
  note="Investigating high CPU — added 2024-01-15"

# Update an annotation (--overwrite required)
kubectl annotate pod web-pod -n production \
  note="Resolved — memory leak fixed" \
  --overwrite

# Remove an annotation (minus sign after key)
kubectl annotate pod web-pod -n production note-
```

---

## Last Applied Configuration — A Hidden Annotation

When you use `kubectl apply`, Kubernetes stores the last-applied configuration as an annotation on the resource:

```bash
kubectl get pod web-pod -n production \
  -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}'
```

This annotation stores a JSON snapshot of the YAML you last applied. `kubectl apply` uses this to calculate the three-way merge diff when you apply again — determining what was added, changed, or removed since the last apply.

This is why `kubectl apply` is preferred over `kubectl create` — apply tracks changes while create does not.

---

## Summary

- Annotations are key-value pairs for storing metadata — unlike labels, they are not used for selection or grouping
- Annotation values can be large (up to 256KB) and contain any characters — JSON, URLs, multi-line text
- Tools use annotations to configure behavior: AWS Load Balancer Controller uses annotations to configure NLBs and ALBs, Argo CD uses them for sync waves and hooks, Prometheus uses them for scrape discovery
- Use annotations to embed operational documentation directly on resources: owner, contact, runbook URLs, Grafana dashboard links, CI/CD build info
- `kubectl annotate` adds or updates annotations on running resources — use `--overwrite` to update, `-` suffix to remove
- `kubectl apply` stores the last applied configuration as an annotation — this enables three-way merge on subsequent applies

In the next file, `04-labels-best-practices.md`, you will learn the label and annotation strategies used by real production teams — consistent naming schemes, what to always label, and how to structure labels for cost tracking and monitoring.
