# Setting Up Fluentd

## Fluentd Overview

Fluentd is a lightweight log aggregator:

```
Input (read logs)
      ↓
Filter (process, enrich)
      ↓
Output (send to Elasticsearch)
```

Runs as DaemonSet on every node.

---

## Prerequisites

- EKS cluster running
- Elasticsearch cluster accessible
- kubectl access
- Helm installed

---

## Install Fluentd via Helm

```bash
# Add Helm repo
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

# Install Fluentd
helm install fluent-bit fluent/fluent-bit \
  -n kube-system \
  --set config.outputs.es.enabled=true \
  --set config.outputs.es.host=elasticsearch.example.com \
  --set config.outputs.es.port=9200
```

---

## Fluentd Configuration

```yaml
# ConfigMap for Fluentd
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: kube-system
data:
  fluent-bit.conf: |
    [SERVICE]
        flush         5
        log_level     info
        daemon        off

    [INPUT]
        name              tail
        path              /var/log/containers/*.log
        parser            docker
        tag               kube.*
        refresh_interval  5

    [FILTER]
        name                kubernetes
        match               kube.*
        keep_log            off
        k8s-logging.parser  on
        k8s-logging.exclude on
        kube_url            https://kubernetes.default.svc:443

    [OUTPUT]
        name            es
        match           *
        host            elasticsearch
        port            9200
        http_user       elastic
        http_passwd     changeme
        index           kubernetes-%Y.%m.%d
        type            _doc
```

---

## Verify Installation

```bash
# Check Fluentd pods
kubectl get pods -n kube-system -l app=fluent-bit

# Check logs
kubectl logs -n kube-system -l app=fluent-bit

# Verify pods are running on all nodes
kubectl get nodes | wc -l       # Node count
kubectl get pods -n kube-system -l app=fluent-bit | wc -l  # Should match
```

---

## Real-World Configuration

```yaml
# Production Fluentd setup
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.0.0
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluent-bit/etc/
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: fluent-bit-config
```

---

## Common Issues

### Issue 1: Fluentd Can't Connect to Elasticsearch

```bash
# Check logs for errors
kubectl logs -n kube-system -l app=fluent-bit | grep -i error

# Verify Elasticsearch connectivity
kubectl exec -it -n kube-system pod/fluent-bit-xxxx -- \
  curl -u elastic:password http://elasticsearch:9200/_cluster/health
```

### Issue 2: High Memory Usage

```yaml
# Reduce log buffering
[SERVICE]
    storage.path /var/log/flb-storage
    storage.sync normal
    storage.checksum on
    storage.max_chunks_up 128
```

### Issue 3: Logs Not Appearing in Elasticsearch

```bash
# Check if index created
curl -u elastic:password http://elasticsearch:9200/_cat/indices

# Check for errors in Fluentd
kubectl logs -n kube-system -l app=fluent-bit | tail -50

# Verify parser config
kubectl describe configmap fluent-bit-config -n kube-system
```

---

## Tuning for Production

```yaml
# High-volume environment
[SERVICE]
    flush              5
    daemon             off
    log_level          info
    max_recvbuf_size   32k

[INPUT]
    name               tail
    path               /var/log/containers/*.log
    parser             docker
    tag                kube.*
    refresh_interval   5
    mem_buf_limit      50MB
    skip_long_lines    on

[OUTPUT]
    name               es
    match              *
    host               elasticsearch
    port               9200
    retry_limit        5
    Trace_Error        on
```

---

## Summary

Fluentd setup:

**Installation:**
- Helm or manifest
- Runs as DaemonSet
- One pod per node

**Configuration:**
- Input: Read from /var/log/containers
- Filter: Enrich with Kubernetes metadata
- Output: Send to Elasticsearch

**Next:** File 05 covers ELK stack integration.
