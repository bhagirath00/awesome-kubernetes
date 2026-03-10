# LoadBalancer Service

## Introduction

The LoadBalancer Service type is the standard way to expose an application to external traffic on AWS EKS. When you create a LoadBalancer Service, Kubernetes provisions a cloud load balancer automatically — on AWS, this is either a Classic Load Balancer (legacy) or an AWS Network Load Balancer (NLB) via the AWS Load Balancer Controller.

---

## How LoadBalancer Services Work on EKS

When you create a Service with `type: LoadBalancer`:

```
Step 1: Kubernetes creates the Service object

Step 2: The cloud controller manager (or AWS Load Balancer Controller)
        detects the new LoadBalancer Service

Step 3: AWS Load Balancer Controller makes AWS API calls to:
        - Create an NLB in your VPC
        - Configure target groups pointing to worker node IPs:NodePorts
        - Configure listeners on port 80/443

Step 4: The NLB's DNS name is written back to the Service's
        status.loadBalancer.ingress[0].hostname field

Step 5: External traffic → NLB → Worker Node:NodePort →
        kube-proxy → Pod
```

```bash
kubectl get service web-service -n production

# After NLB is provisioned (takes ~1-2 minutes):
# NAME          TYPE           CLUSTER-IP      EXTERNAL-IP                             PORT(S)
# web-service   LoadBalancer   172.20.45.123   abc123.elb.us-east-1.amazonaws.com      80:30080/TCP
#                                              ↑ This is the NLB DNS name
```

---

## Creating a LoadBalancer Service — AWS NLB

The AWS Load Balancer Controller uses annotations to configure the NLB. Here is a production-ready LoadBalancer Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: production
  annotations:
    # Use the new AWS Load Balancer Controller (not the legacy in-tree controller)
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"

    # Create an internet-facing NLB (use "internal" for private NLBs)
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"

    # Place the NLB in specific subnets (public subnets for internet-facing)
    service.beta.kubernetes.io/aws-load-balancer-subnets: >
      subnet-abc123,subnet-def456,subnet-ghi789

    # Health check for the NLB target group
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/healthz"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "8080"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"

    # Enable cross-zone load balancing (distribute traffic evenly across AZs)
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

    # Tag the NLB for cost allocation
    service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: >
      Environment=production,Team=platform,ManagedBy=kubernetes

    # Enable access logs to S3
    service.beta.kubernetes.io/aws-load-balancer-access-log-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-name: "my-nlb-logs-bucket"
    service.beta.kubernetes.io/aws-load-balancer-access-log-s3-bucket-prefix: "production/web-service"

spec:
  type: LoadBalancer
  selector:
    app: web-server
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: https
    port: 443
    targetPort: 8443
    protocol: TCP
  externalTrafficPolicy: Local    # preserves client IP, avoids cross-node hops
```

```bash
kubectl apply -f loadbalancer-service.yaml

# Monitor NLB provisioning (takes 1-3 minutes)
kubectl get service web-service -n production -w

# Once provisioned:
# NAME          TYPE           CLUSTER-IP    EXTERNAL-IP                                    PORT(S)
# web-service   LoadBalancer   172.20.45.1   k8s-prod-web-abc123.elb.us-east-1.amazonaws.com  80:30080/TCP

# Test the NLB
curl http://k8s-prod-web-abc123.elb.us-east-1.amazonaws.com/healthz
```

---

## Target Type: ip vs instance

The `aws-load-balancer-nlb-target-type` annotation controls how the NLB sends traffic to your cluster:

### `instance` mode (legacy)
The NLB targets worker node IPs + NodePort. Traffic: NLB → Node:NodePort → kube-proxy → Pod.

### `ip` mode (recommended with VPC CNI)
The NLB targets Pod IPs directly. Traffic: NLB → Pod IP:Port directly. No kube-proxy hop.

```yaml
# Recommended for EKS with VPC CNI
service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
```

With `ip` mode, the NLB talks directly to Pod IPs (which are real VPC IPs via VPC CNI). This eliminates the NodePort + kube-proxy layer, reducing latency and preserving the client's source IP naturally.

---

## Internal NLB — For Private Services

For services that should only be reachable within the VPC (not from the internet):

```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
    service.beta.kubernetes.io/aws-load-balancer-subnets: >
      subnet-private-abc123,subnet-private-def456
```

Internal NLBs get private IP addresses from your private subnets. They are reachable from:
- Other services in the same VPC
- On-premise networks connected via VPN or Direct Connect
- Other VPCs with VPC peering

---

## TLS Termination at the NLB

For HTTPS traffic, the NLB can terminate TLS using an ACM (AWS Certificate Manager) certificate:

```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: >
      arn:aws:acm:us-east-1:123456789:certificate/abc-123-def-456

    # Which ports to apply TLS to
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"

    # What protocol to use between NLB and Pod (after TLS termination)
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"

spec:
  ports:
  - name: https
    port: 443
    targetPort: 8080      # Pod receives plain HTTP (TLS already terminated)
    protocol: TCP
  - name: http
    port: 80
    targetPort: 8080
```

With this setup, the NLB terminates TLS and forwards plain HTTP to Pods. Pods do not need to handle TLS certificates.

---

## Checking NLB Status in AWS

```bash
# Get the NLB ARN from the Service annotation (added automatically)
kubectl get service web-service -n production -o yaml | \
  grep "arn:aws:elasticloadbalancing"

# List all NLBs in your account
aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --query 'LoadBalancers[?Type==`network`].[LoadBalancerName,State.Code,DNSName]' \
  --output table

# Check NLB target group health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:123456789:targetgroup/k8s-prod/abc123

# Output:
# TargetHealthDescriptions:
# - Target:
#     Id: 10.0.1.45        # Pod IP (in ip mode)
#     Port: 8080
#   TargetHealth:
#     State: healthy
# - Target:
#     Id: 10.0.1.46
#     Port: 8080
#   TargetHealth:
#     State: healthy
```

---

## Cost Considerations

Each LoadBalancer Service creates one AWS NLB. NLBs cost approximately **$0.008 per LCU-hour** plus a fixed **$0.0225 per hour** (~$16.20/month minimum per NLB).

For clusters with many Services that need external access, provisioning one NLB per Service gets expensive fast. The recommended approach for HTTP services:
- Use **one NLB per cluster** (or per environment)
- Use an **Ingress** (which uses an ALB) to route HTTP traffic to multiple Services based on host and path
- Reserve LoadBalancer Services for TCP/UDP services that cannot use HTTP routing

---

## Summary

- LoadBalancer Service type automatically provisions an AWS NLB when used with the AWS Load Balancer Controller on EKS
- Use `aws-load-balancer-nlb-target-type: "ip"` to route traffic directly to Pod IPs — eliminates the NodePort/kube-proxy layer and preserves client IP
- Use `aws-load-balancer-scheme: "internet-facing"` for public services and `"internal"` for private services accessible only within the VPC
- NLBs can terminate TLS using ACM certificates — annotate with the certificate ARN
- `externalTrafficPolicy: Local` with `ip` target mode preserves the client source IP
- Each LoadBalancer Service creates one NLB — for HTTP services with many routes, use Ingress + ALB instead to save cost

In the next file, `05-externalname.md`, you will learn about ExternalName Services — which create DNS aliases to external services, allowing Pods to reference external databases and APIs by a stable Kubernetes Service name.
