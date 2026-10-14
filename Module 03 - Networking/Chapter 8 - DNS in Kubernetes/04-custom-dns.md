# Custom DNS Configuration

## Introduction

The default CoreDNS configuration works well for most clusters. But real-world enterprise environments often have requirements that go beyond the defaults: resolving private corporate domains, forwarding DNS for specific zones to on-premise resolvers, rewriting DNS names, or integrating with AWS Route 53 private hosted zones.

This file covers how to customize CoreDNS configuration for common enterprise scenarios.

---

## Editing the CoreDNS ConfigMap

All CoreDNS customization is done by editing the `coredns` ConfigMap in the `kube-system` namespace:

```bash
# View current configuration
kubectl get configmap coredns -n kube-system -o yaml

# Edit directly (changes take effect within ~30 seconds — CoreDNS watches for changes)
kubectl edit configmap coredns -n kube-system

# Or apply a new version declaratively:
kubectl apply -f coredns-configmap.yaml
```

After editing, CoreDNS reloads automatically (the `reload` plugin in the Corefile handles this). You do not need to restart CoreDNS Pods.

```bash
# Verify the reload happened
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20

# Look for:
# [INFO] Reloading
# [INFO] plugin/reload: Running configuration ...
# [INFO] Reloading complete
```

---

## Adding a Stub Zone — Private Corporate Domain

A stub zone forwards DNS queries for a specific domain to a custom DNS server, instead of going to the default upstream resolver.

**Use case:** Your cluster needs to resolve `mycompany.internal` hostnames that only an on-premise DNS server knows about.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }

    # Stub zone: forward mycompany.internal to on-premise DNS
    mycompany.internal:53 {
        errors
        cache 30
        forward . 10.100.0.2 10.100.0.3    # on-premise DNS server IPs
    }

    # Stub zone: forward corp.example.com to a specific DNS server
    corp.example.com:53 {
        errors
        cache 30
        forward . 192.168.1.10
    }
```

With this configuration:
- Queries for `*.mycompany.internal` → forwarded to `10.100.0.2` (on-premise DNS)
- Queries for `*.cluster.local` → handled by the `kubernetes` plugin
- All other queries → forwarded to the VPC DNS resolver (`/etc/resolv.conf`)

---

## Forwarding to AWS Route 53 Private Hosted Zones

If you have AWS Route 53 Private Hosted Zones attached to your VPC, CoreDNS can forward to the VPC DNS resolver (which resolves Route 53 private zones):

```yaml
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153

        # Forward ALL non-cluster queries to the VPC DNS resolver
        # On AWS, this is the VPC base IP + 2 (e.g., 10.0.0.2 for 10.0.0.0/16 VPC)
        forward . 10.0.0.2 {
           max_concurrent 1000
        }

        cache 30
        loop
        reload
        loadbalance
    }

    # Private hosted zone: specific domain forwards to VPC resolver
    myapp.aws.internal:53 {
        errors
        cache 30
        forward . 10.0.0.2      # VPC DNS resolves Route 53 private hosted zones
    }
```

The VPC DNS resolver at `<VPC CIDR base + 2>` (e.g., `10.0.0.2` for a `10.0.0.0/16` VPC) can resolve Route 53 Private Hosted Zones associated with the VPC. By forwarding to it, your Pods resolve private AWS DNS names without special configuration.

---

## DNS Rewrites — Mapping One Name to Another

The `rewrite` plugin changes DNS query names before they are resolved. Useful for:
- Migrating from one Service name to another without changing application code
- Creating aliases for frequently-used services

```yaml
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }

        # Rewrite: 'db' → 'postgres-service.production.svc.cluster.local'
        # Allows applications to use the short name 'db' from any namespace
        rewrite name db.cluster.local postgres-service.production.svc.cluster.local

        # Rewrite: 'cache' → 'redis-service.production.svc.cluster.local'
        rewrite name cache.cluster.local redis-service.production.svc.cluster.local

        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

With the rewrite above, any Pod querying `db.cluster.local` gets the IP of `postgres-service.production.svc.cluster.local` instead.

---

## Handling DNS for Hybrid Cloud — VPN Connected On-Premise

In a hybrid cloud setup where your EKS cluster connects to on-premise via AWS Direct Connect or VPN, Pods may need to reach on-premise services by DNS name:

```yaml
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }

    # On-premise zone — routed through Direct Connect
    onprem.company.com:53 {
        errors
        cache 60
        forward . 10.200.1.10 10.200.1.11 {    # on-premise DNS servers
            policy sequential                    # try first server, fallback to second
        }
    }

    # Internal corporate LDAP/AD zone
    corp.ad.company.com:53 {
        errors
        cache 30
        forward . 10.200.2.10 {
            tls_servername corp-ad-dns.company.com
        }
    }
```

---

## Custom Hosts — Static DNS Entries

For a small number of static entries, use the `hosts` plugin instead of a full stub zone:

```yaml
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }

        # Static host entries
        hosts {
            10.100.1.50  legacy-service.mycompany.com
            10.100.1.51  old-api.mycompany.com
            10.100.1.52  monitoring.mycompany.com
            fallthrough
        }

        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

`fallthrough` means: if a query does not match a hosts entry, continue to the next plugin.

---

## Testing Custom DNS Configuration

After updating the CoreDNS ConfigMap, test that the changes work:

```bash
# Run a test Pod with networking tools
kubectl run dns-debug --image=tutum/dnsutils --rm -it --restart=Never -- bash

# Inside the test Pod:

# Test cluster DNS still works
nslookup kubernetes.default.svc.cluster.local
# Should return the kubernetes Service ClusterIP

# Test stub zone resolution (on-premise domain)
nslookup internal-tool.mycompany.internal
# Should return an IP from the on-premise DNS server

# Test Route 53 private hosted zone
nslookup myapp.aws.internal
# Should return the private IP from Route 53

# Test rewrite rule
nslookup db.cluster.local
# Should return the postgres-service ClusterIP

# Check which DNS server is being queried
nslookup -debug google.com 2>&1 | grep Server
# Server: 172.20.0.10  ← CoreDNS

# Test with specific DNS server (bypass CoreDNS to test upstream)
nslookup mycompany.internal 10.100.0.2
# Should work if the on-premise DNS server is reachable
```

---

## CoreDNS Troubleshooting Reference

```bash
# Is CoreDNS running?
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Are both replicas ready?
kubectl get deployment coredns -n kube-system

# View recent CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# Enable debug logging temporarily (in the Corefile, add 'log' plugin):
# .:53 {
#     log            ← add this to log all queries
#     errors
#     ...
# }
# WARNING: Remove after debugging — log generates massive output in production

# Check CoreDNS metrics
kubectl port-forward -n kube-system svc/kube-dns 9153:9153 &
curl http://localhost:9153/metrics | grep coredns_dns_request

# Validate ConfigMap syntax before applying
kubectl create configmap test-coredns --from-file=Corefile=./Corefile --dry-run=client
```

---

## Summary

- CoreDNS is configured via the `coredns` ConfigMap in `kube-system` — changes take effect within ~30 seconds without Pod restarts
- **Stub zones** forward queries for specific domains to custom DNS servers — use for on-premise domains, Route 53 private hosted zones, or corporate DNS
- The VPC DNS resolver (`VPC CIDR +2`) resolves Route 53 Private Hosted Zones — forward to it for AWS-private domain resolution
- The **rewrite** plugin maps one DNS name to another — useful for migrations and creating short aliases usable across namespaces
- The **hosts** plugin adds static DNS entries for a small number of fixed external hostnames
- After editing the Corefile, verify with a test Pod using `nslookup` against cluster services, stub zones, and external domains
- Enable the `log` plugin temporarily for deep DNS debugging — disable in production immediately after use

This completes Module 3 — Networking. You now understand the complete Kubernetes networking stack: the flat network model, Pod-to-Pod communication via VPC CNI, iptables-based Service routing, all four Service types, DNS-based service discovery, and CoreDNS customization.

In Module 4, you will move to Configuration — learning how to manage application settings with ConfigMaps, secrets with Kubernetes Secrets and External Secrets Operator, and resource management with requests, limits, and quotas.
