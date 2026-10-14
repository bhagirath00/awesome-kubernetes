# ELK Stack Integration

## Full ELK Deployment

Complete centralized logging with Elasticsearch, Logstash, and Kibana.

---

## Prerequisites

- EKS cluster
- 3+ nodes (ELK is resource-intensive)
- At least 4 CPUs, 8GB RAM per node

---

## Install Elasticsearch

```bash
# Add Helm repo
helm repo add elastic https://Helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  -n elk \
  --create-namespace \
  --set replicas=3 \
  --set resources.requests.memory=2Gi \
  --set resources.limits.memory=2Gi
```

---

## Install Kibana

```bash
# Install Kibana
helm install kibana elastic/kibana \
  -n elk \
  --set elasticsearchHosts=http://elasticsearch-master:9200 \
  --set service.type=LoadBalancer
```

Kibana runs on port 5601:

```bash
# Get external IP
kubectl get svc -n elk kibana

# Access in browser
# http://external-ip:5601
```

---

## Fluentd → Elasticsearch

Configure Fluentd to send logs to Elasticsearch:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: elk
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>

    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
      kubernetes_url "#{ENV['FLUENT_FILTER_KUBERNETES_URL'] || 'https://' + ENV.fetch('KUBERNETES_SERVICE_HOST') + ':' + ENV.fetch('KUBERNETES_SERVICE_PORT') + '/api'}"
      verify_ssl "#{ENV['KUBERNETES_VERIFY_SSL'] || true}"
      ca_file "#{ENV['KUBERNETES_CA_FILE']}"
    </filter>

    <match **>
      @type elasticsearch
      @id output_elasticsearch
      @log_level info
      include_tag_key true
      host elasticsearch-master
      port 9200
      path _bulk
      compress gzip
      logstash_format true
      logstash_prefix "kubernetes"
      logstash_prefix_separator "-"
      logstash_dateformat %Y.%m.%d
      include_timestamp false
      type_name "_doc"
    </match>
```

---

## Deploy Fluentd as DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: elk
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
          - name: FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch-master"
          - name: FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
        resources:
          limits:
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 256Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

---

## Real-World Example: Production Setup

```bash
# Namespace for logging infrastructure
kubectl create namespace elk

# Deploy Elasticsearch (multi-node for HA)
helm install elasticsearch elastic/elasticsearch \
  -n elk \
  --set replicas=3 \
  --set nodeGroup=data \
  --set resources.requests.cpu=1 \
  --set resources.requests.memory=2Gi

# Deploy Kibana
helm install kibana elastic/kibana \
  -n elk \
  --set service.type=LoadBalancer

# Deploy Fluentd DaemonSet
kubectl apply -f fluentd-daemonset.yaml

# Verify all running
kubectl get pods -n elk
```

---

## Using Kibana

### 1. Create Index Pattern

- Navigate to Kibana (http://your-ip:5601)
- Click "Stack Management" → "Index Patterns"
- Click "Create index pattern"
- Pattern: `kubernetes-*`
- Time field: `@timestamp`
- Click "Create"

### 2. Explore Logs

- Click "Discover"
- Select index `kubernetes-*`
- See all logs from cluster

### 3. Search Examples

```
kubernetes.pod_name:web-app

level:ERROR

kubernetes.namespace:production AND @timestamp:["now-1h" TO "now"]

message:*timeout*
```

### 4. Create Dashboard

- Click "Dashboard"
- Create new dashboard
- Add visualizations (pie chart, bar chart)
- Show logs by namespace, pod, error level

---

## Retention and Cleanup

```bash
# View Elasticsearch indices
curl -u elastic:password http://elasticsearch:9200/_cat/indices

# Delete old indices (older than 30 days)
curl -X DELETE \
  -u elastic:password \
  http://elasticsearch:9200/kubernetes-2023-12-*

# Automate with Index Lifecycle Management (ILM)
```

---

## Monitoring ELK Itself

```bash
# Check Elasticsearch cluster health
curl -u elastic:password http://elasticsearch:9200/_cluster/health

# Check disk usage
curl -u elastic:password http://elasticsearch:9200/_cat/nodes

# Monitor index sizes
curl -u elastic:password http://elasticsearch:9200/_cat/indices?v
```

---

## Common Mistakes

### Mistake 1: Insufficient Resources

```yaml
# WRONG — not enough memory
resources:
  requests:
    memory: 512Mi
  limits:
    memory: 512Mi

# Elasticsearch crashes with OOMKilled
```

**Fix:** Allocate proper resources.

```yaml
# RIGHT
resources:
  requests:
    memory: 2Gi
  limits:
    memory: 2Gi
```

### Mistake 2: Single Elasticsearch Node

```yaml
# WRONG — single node cluster
replicas: 1

# Data loss risk
# No redundancy
# Can't do maintenance
```

**Fix:** Use 3+ nodes for HA.

---

## Troubleshooting

```bash
# Elasticsearch won't start
kubectl describe pod elasticsearch-0 -n elk

# Check disk space
kubectl exec elasticsearch-0 -n elk -- df -h

# Verify connectivity
kubectl exec fluentd-xxxxx -n elk -- \
  curl -u elastic:password http://elasticsearch:9200/_cluster/health

# Check Kibana logs
kubectl logs -n elk kibana-xxxxx
```

---

## Cost Optimization

```yaml
# For development (small cluster)
elasticsearch:
  replicas: 1
  storage: 10Gi

# For production (high volume)
elasticsearch:
  replicas: 3
  storage: 100Gi
  node_roles: ["master", "data"]

# For high-traffic (separate roles)
elasticsearch-master:
  replicas: 3
  roles: ["master"]
elasticsearch-data:
  replicas: 5
  roles: ["data"]
```

---

## Summary

ELK stack integration:

**Components:**
- Elasticsearch: Storage and search
- Kibana: UI for searching
- Fluentd: Log aggregation

**Setup:**
1. Deploy Elasticsearch (3+ nodes)
2. Deploy Kibana
3. Deploy Fluentd DaemonSet
4. Create Kibana index pattern
5. Start searching logs

**Management:**
- Index lifecycle policies
- Retention cleanup
- Cluster health monitoring

**Module 9 Chapter 28 Complete - 5 Files!**

You now understand:
- File 01: Logging architecture overview
- File 02: Node-level logging on each node
- File 03: Cluster-level centralized logging
- File 04: Fluentd setup and configuration
- File 05: Full ELK integration

**Next:** Chapter 29 covers monitoring and metrics.
