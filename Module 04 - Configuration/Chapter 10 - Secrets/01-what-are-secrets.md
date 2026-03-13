# What Are Secrets

## Introduction

Every application handles sensitive data — database passwords, API keys, TLS certificates, OAuth tokens. Storing this data in ConfigMaps, Docker images, or environment files that get committed to Git is a serious security risk. **Secrets** are Kubernetes objects designed specifically for sensitive data.

Secrets provide a separate storage path from ConfigMaps, with access controls, and optionally encryption at rest. When combined with proper RBAC, Secrets ensure that only the Pods and users that need sensitive data can access it.

---

## How Secrets Differ from ConfigMaps

```
ConfigMap:
  data:
    LOG_LEVEL: "warn"
  Stored in etcd as plain text

Secret:
  data:
    password: d2VhazNycGFzc3dvcmQ=   ← base64 encoded
  Stored in etcd (optionally encrypted with envelope encryption)
```

The main differences:

| | ConfigMap | Secret |
|---|-----------|--------|
| **Data stored as** | Plain text | Base64 encoded |
| **Encryption at rest** | No | Yes, if envelope encryption configured |
| **RBAC enforcement** | Yes (same as all objects) | Yes — with tighter scope possible |
| **Shown in describe** | Full values visible | Values not shown by default |
| **For** | Non-sensitive config | Passwords, keys, certificates |

> ⚠️ **Important:** Base64 is **not** encryption. It is encoding. Anyone with access to the Secret object can decode the value in one command. The real protection comes from RBAC (restricting who can `get` Secrets) and optionally from envelope encryption (encrypting the etcd data at rest).

---

## Secret Types

Kubernetes has several built-in Secret types for specific use cases:

```yaml
type: Opaque                        # default — arbitrary user-defined data
type: kubernetes.io/service-account-token   # service account token
type: kubernetes.io/dockerconfigjson        # Docker registry credentials
type: kubernetes.io/tls                     # TLS certificate and key
type: kubernetes.io/basic-auth             # username and password
type: kubernetes.io/ssh-auth               # SSH private key
type: bootstrap.kubernetes.io/token        # bootstrap token
```

| Type | Use Case |
|------|---------|
| `Opaque` | Database passwords, API keys, custom secrets |
| `kubernetes.io/tls` | TLS certificates for Ingress, services |
| `kubernetes.io/dockerconfigjson` | Pull images from private registries |
| `kubernetes.io/basic-auth` | HTTP basic authentication credentials |
| `kubernetes.io/service-account-token` | Auto-created for ServiceAccounts |

---

## The Opaque Secret Type

For most use cases — database passwords, API keys, tokens — use `Opaque`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
data:
  username: YWRtaW4=          # base64 encoded "admin"
  password: d2VhazNycGFzc3dvcmQ=  # base64 encoded "weak3rpassword"
```

Base64 encoding:
```bash
# Encode a value to base64
echo -n "mypassword" | base64
# bXlwYXNzd29yZA==

# The -n flag prevents a newline from being encoded
# Without -n: echo "mypassword" | base64 encodes "mypassword\n" — WRONG

# Decode to verify
echo "bXlwYXNzd29yZA==" | base64 --decode
# mypassword
```

You can also use `stringData` — values are plain text and Kubernetes encodes them automatically:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
stringData:                     # plain text — Kubernetes base64 encodes it
  username: admin
  password: "mypassword123!"    # readable in the YAML — use carefully
```

> ⚠️ **Warning:** Even though `stringData` is convenient, plain-text secrets in YAML files should never be committed to Git. Use a secrets management tool (AWS Secrets Manager + External Secrets Operator, SOPS, Sealed Secrets) for production.

---

## TLS Secret

TLS Secrets store a certificate and private key for use with Ingress:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: LS0t...  # base64 encoded certificate (PEM format)
  tls.key: LS0t...  # base64 encoded private key (PEM format)
```

Create from existing certificate files:
```bash
kubectl create secret tls tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem \
  -n production
```

Reference in an Ingress:
```yaml
spec:
  tls:
  - hosts:
    - api.mycompany.com
    secretName: tls-secret    # references the TLS Secret
```

---

## Docker Registry Secret

Pull images from AWS ECR or private Docker registries:

```bash
# Create a registry secret for Docker Hub
kubectl create secret docker-registry ecr-secret \
  --docker-server=123456789.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  -n production
```

For EKS, the recommended approach is not registry Secrets but **IRSA (IAM Roles for Service Accounts)** — the worker node's IAM role or the Pod's service account gets ECR pull permissions, eliminating the need for registry secrets entirely.

---

## What Kubernetes Does Not Do For You

Kubernetes Secrets provide controlled access and optional encryption at rest. They do NOT:
- Encrypt the Secret at rest by default (requires enabling envelope encryption in etcd)
- Prevent administrators from reading Secrets (cluster admins can always read Secrets)
- Rotate credentials automatically (you manage rotation)
- Integrate with AWS Secrets Manager natively (requires External Secrets Operator)

For a production-grade Secrets solution on AWS, the recommended pattern is:
1. Store the actual secret value in **AWS Secrets Manager**
2. Use **External Secrets Operator** to sync it into a Kubernetes Secret
3. Reference the Kubernetes Secret in your Pods

This pattern is covered in `05-external-secrets-operator.md`.

---

## Secret Access in etcd — Securing with Envelope Encryption

By default, Secrets in etcd are stored as base64-encoded plain text. Envelope encryption encrypts the Secret data before writing to etcd using a KMS key:

```yaml
# encryption-config.yaml (provided to kube-apiserver)
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - kms:
      name: aws-kms
      endpoint: unix:///var/run/kmsplugin/socket.sock
      apiVersion: v2
      timeout: 3s
  - identity: {}   # fallback — stores unencrypted (should be last)
```

On EKS, you can enable envelope encryption via the console or Terraform:

```bash
# Enable on existing EKS cluster
aws eks associate-encryption-config \
  --cluster-name my-cluster \
  --encryption-config '[{"resources":["secrets"],"provider":{"keyArn":"arn:aws:kms:us-east-1:123456789:key/abc-123-def"}}]' \
  --region us-east-1
```

With envelope encryption enabled, Secrets are encrypted in etcd using the KMS key. Even if etcd storage is compromised, the data is unreadable without the KMS key.

---

## Summary

- Secrets store sensitive data (passwords, API keys, TLS certificates) separately from ConfigMaps — with base64 encoding and optional encryption at rest
- Base64 is encoding, not encryption — real security comes from RBAC (restricting who can `get` Secrets) and envelope encryption (encrypting etcd)
- Four most common Secret types: `Opaque` (custom data), `kubernetes.io/tls` (certificates), `kubernetes.io/dockerconfigjson` (registry auth), `kubernetes.io/basic-auth`
- Use `stringData` for readable YAML definitions — but never commit plain-text secrets to Git; use External Secrets Operator, SOPS, or Sealed Secrets for production
- Enable EKS envelope encryption with a KMS key to protect Secrets stored in etcd

In the next file, `02-creating-secrets.md`, you will learn all the ways to create Secrets — from literal values, from files, with kubectl, and using YAML manifests — with guidance on which approach is safe for production.
