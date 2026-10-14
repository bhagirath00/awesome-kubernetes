# CoreDNS Overview

## Introduction

Every time a Pod resolves a Service name like `web-service.production.svc.cluster.local`, something inside the cluster answers that DNS query. That something is **CoreDNS** — the cluster DNS server that runs as a Deployment in the `kube-system` namespace.

Without CoreDNS, Pods cannot resolve Service names. Network traffic between services would require hard-coded IP addresses, making the entire cluster DNS-dependent application layer unusable. CoreDNS is a critical component — when it fails, nothing in the cluster can communicate by name.

---

## What CoreDNS Is

CoreDNS is a flexible, extensible DNS server written in Go. It replaced `kube-dns` as the default cluster DNS starting in Kubernetes 1.11. It runs as a standard Kubernetes Deployment with a Service in front of it.

```bash
# CoreDNS runs as a Deployment in kube-system
kubectl get deployment coredns -n kube-system

# NAME      READY   UP-TO-DATE   AVAILABLE   AGE
# coredns   2/2     2            2           30d

# CoreDNS Pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# NAME                       READY   STATUS    RESTARTS   AGE
# coredns-6d4b75cb6d-abc12   1/1     Running   0          30d
# coredns-6d4b75cb6d-def34   1/1     Running   0          30d

# CoreDNS Service — every Pod uses this IP for DNS
kubectl get service kube-dns -n kube-system

# NAME       TYPE        CLUSTER-IP    PORT(S)         AGE
# kube-dns   ClusterIP   172.20.0.10   53/UDP,53/TCP   30d
```

CoreDNS always has the same ClusterIP (`172.20.0.10` in this example). Every Pod on the cluster is configured to use this IP as its DNS nameserver.

---

## How Every Pod Gets Its DNS Configuration

When a Pod starts, the kubelet writes a `/etc/resolv.conf` file inside the Pod's network namespace:

```bash
# Inside any Pod — view the DNS configuration
kubectl exec -it web-pod-abc123 -n production -- cat /etc/resolv.conf

# Output:
# nameserver 172.20.0.10
# search production.svc.cluster.local svc.cluster.local cluster.local ec2.internal
# options ndots:5
```

Three key settings:

**`nameserver 172.20.0.10`**
All DNS queries go to CoreDNS at this IP. kubelet sets this to the ClusterIP of the `kube-dns` Service.

**`search production.svc.cluster.local svc.cluster.local cluster.local ec2.internal`**
These are search domains. When a Pod queries `web-service`, the resolver tries each search domain in order:
- `web-service.production.svc.cluster.local` → found → done
- (would try next ones if not found)

**`options ndots:5`**
If the query has fewer than 5 dots, treat it as a relative name and append search domains. If it has 5 or more dots, try it as an absolute name first. This is why `web-service` (0 dots) goes through search domains but `api.external-company.com` (3 dots) still tries search domains first (3 < 5).

---

## CoreDNS Configuration — The Corefile

CoreDNS is configured via a **Corefile**, stored as a ConfigMap:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors                    # log errors
        health {                  # health check endpoint
           lameduck 5s
        }
        ready                     # readiness endpoint
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure           # enable pod DNS records
           fallthrough in-addr.arpa ip6.arpa
           ttl 30                  # DNS TTL for cluster records (30 seconds)
        }
        prometheus :9153           # metrics endpoint for Prometheus
        forward . /etc/resolv.conf { # forward non-cluster queries to upstream DNS
           max_concurrent 1000
        }
        cache 30                   # cache responses for 30 seconds
        loop                       # detect forwarding loops
        reload                     # auto-reload config on change
        loadbalance                # round-robin DNS responses
    }
```

### Key Plugins in the Corefile

**`kubernetes cluster.local`** — handles all cluster DNS queries for `*.cluster.local` names. It reads from the Kubernetes API to resolve Service and Pod names.

**`forward . /etc/resolv.conf`** — forwards all non-cluster DNS queries to the upstream nameserver (the VPC DNS resolver at the node level, which is the VPC network's `+2` address).

**`cache 30`** — caches DNS responses for 30 seconds. Reduces load on CoreDNS and improves response times for repeated queries.

**`prometheus :9153`** — exposes CoreDNS metrics in Prometheus format. These metrics tell you query rate, error rate, and latency.

---

## How CoreDNS Answers a Query

Let us trace what happens when a Pod queries `web-service.production.svc.cluster.local`:

```
Pod sends DNS query to 172.20.0.10:53 (CoreDNS)
         │
         ▼
CoreDNS receives: "what is web-service.production.svc.cluster.local?"
         │
         ▼
kubernetes plugin checks: is this a *.cluster.local query?
         │ Yes
         ▼
Queries the Kubernetes API:
  GET /api/v1/namespaces/production/services/web-service
         │
         ▼
API Server returns Service object with ClusterIP: 172.20.45.123
         │
         ▼
CoreDNS returns A record: web-service.production.svc.cluster.local → 172.20.45.123
TTL: 30 seconds (cached for 30 seconds by CoreDNS)
         │
         ▼
Pod resolver receives: 172.20.45.123
Pod connects to 172.20.45.123:80
```

For an external query like `api.stripe.com`:

```
Pod sends DNS query to 172.20.0.10:53 (CoreDNS)
         │
CoreDNS: not a *.cluster.local query
         │
         ▼
forward plugin: forward to /etc/resolv.conf upstream
         │
         ▼
VPC DNS resolver (169.254.169.253 on AWS or VPC+2 address)
         │
         ▼
Resolves api.stripe.com → real IP
         │
         ▼
Response returned to Pod
```

---

## CoreDNS Metrics — Monitoring DNS Health

CoreDNS exposes Prometheus metrics at `:9153/metrics`:

```bash
# Check CoreDNS metrics endpoint
kubectl exec -n kube-system coredns-6d4b75cb6d-abc12 -- \
  wget -O- http://localhost:9153/metrics | grep -E "coredns_dns_requests|coredns_dns_responses"

# Key metrics to monitor:
# coredns_dns_requests_total        — total DNS queries
# coredns_dns_responses_total       — responses by rcode
# coredns_dns_request_duration_seconds — query latency histogram
# coredns_cache_hits_total          — cache hit rate
# coredns_forward_requests_total    — forwarded (external) queries
```

In Grafana, watch for:
- **High `NXDOMAIN` response rate** — applications querying non-existent Service names (misconfigured)
- **High query latency** — CoreDNS under load or upstream resolver is slow
- **Low cache hit rate** — many unique queries, no caching benefit

---

## Scaling CoreDNS

EKS runs 2 CoreDNS replicas by default. For large clusters (hundreds of nodes, thousands of Pods), you may need to scale up:

```bash
# Scale CoreDNS to 4 replicas
kubectl scale deployment coredns -n kube-system --replicas=4

# Or configure Horizontal Pod Autoscaler for CoreDNS
# (a cluster-proportional-autoscaler is common for this)
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/cluster-proportional-autoscaler/master/examples/coredns/Deployment.yaml
```

CoreDNS is CPU-bound for high query volumes. Each replica can handle approximately 15,000–20,000 queries per second. For very large clusters, the NodeLocal DNSCache addon runs CoreDNS locally on each node, eliminating the network hop to central CoreDNS:

```bash
# Enable NodeLocal DNSCache on EKS
# This runs a DNS cache directly on each node, reducing CoreDNS load by 90%+
kubectl apply -f node-local-dns.yaml
```

---

## Summary

- CoreDNS is the cluster DNS server — it runs as a 2-replica Deployment in `kube-system` and is the nameserver for every Pod in the cluster
- Every Pod's `/etc/resolv.conf` points to the `kube-dns` Service ClusterIP, with search domains for the current namespace, `svc.cluster.local`, and `cluster.local`
- The `kubernetes` CoreDNS plugin handles all `*.cluster.local` queries by reading from the Kubernetes API; the `forward` plugin sends all other queries to the upstream VPC DNS resolver
- CoreDNS caches responses for 30 seconds by default — DNS changes in Kubernetes propagate within 30 seconds
- Monitor CoreDNS with Prometheus — watch for high NXDOMAIN rates (bad Service names) and high latency
- Scale CoreDNS horizontally for large clusters; use NodeLocal DNSCache for extreme DNS query volumes

In the next file, `02-service-discovery.md`, you will learn how Kubernetes service discovery works end to end — how applications find each other by name across namespaces and what DNS patterns to use in your application configuration.
