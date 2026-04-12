# Kubernetes Certificates

## Why Kubernetes Needs Certificates

Kubernetes is like a busy control room.

Different parts of the system talk to each other all the time:

- The API server talks to kubelets
- Controllers talk to the API server
- Users talk to the API server
- Admission webhooks talk to the cluster

If those conversations are not trusted, the whole cluster becomes unsafe.

That is why Kubernetes uses certificates.

---

## The Main Certificate Story in Kubernetes

There are two big certificate jobs in Kubernetes:

1. Prove who the user or service is
2. Encrypt communication between components

### Simple View

| Component | Uses Certificates For |
|-----------|------------------------|
| API server | Secure control plane communication |
| Kubelet | Secure node communication |
| Users | Secure access with client certs or tokens |
| Webhooks | Trusted admission calls |

---

## Types of Certificates

### 1. Server Certificate

A server certificate proves the identity of the server.

Example: The Kubernetes API server presents a certificate to kubectl.

### 2. Client Certificate

A client certificate proves the identity of the client.

Example: kubectl or a controller may use a client cert.

### 3. CA Certificate

The CA certificate is the trusted root that signs other certificates.

If the CA is trusted, the signed certificates it creates can be trusted too.

---

## How Kubernetes Uses Them

Imagine you run kubectl against a cluster.

Here is the story:

1. kubectl connects to the API server
2. The API server shows its certificate
3. kubectl checks that it trusts the CA
4. kubectl may send its own client certificate
5. The API server checks the client identity
6. Secure communication begins

That is how Kubernetes keeps the control plane protected.

---

## Where Certificates Live

Certificates may live in different places:

- Control plane files on the master node
- kubeconfig files on the user machine
- Kubernetes Secrets for app TLS certificates
- CSR resources for new certificate requests

In AWS EKS, some control plane certificate work is managed for you, but your apps still need their own certificates.

---

## kubeconfig and Access

Your kubeconfig file often contains:

- Cluster info
- User info
- Context info
- Certificates or tokens

### Example View

```yaml
apiVersion: v1
clusters:
	- cluster:
			server: https://api.example.com
			certificate-authority-data: <base64>
		name: my-cluster
users:
	- name: admin
		user:
			client-certificate-data: <base64>
			client-key-data: <base64>
```

This file tells kubectl how to talk to the cluster securely.

---

## Common Certificate Problems

### Expired Certificate

If a certificate expires, trust breaks.

### Wrong Hostname

If the certificate is issued for `api.example.com` but you connect to `cluster.example.com`, the check fails.

### Wrong CA

If the client trusts a different CA, it will reject the certificate.

### Private Key Mismatch

If the certificate and private key do not belong together, TLS will fail.

---

## Using Certificates in Kubernetes Secrets

Most apps store TLS certs in a Secret.

### Example

```bash
kubectl create secret tls shop-tls \
	--cert=tls.crt \
	--key=tls.key \
	-n production
```

That secret can then be used by an ingress or application.

### Why This Is Useful

It keeps the certificate and private key inside the cluster in a standard format.

---

## AWS and Certificates

In AWS, you may also use:

- ACM for public TLS certificates
- Private certificates for internal services
- Load balancers that terminate TLS

For example, an ALB in front of EKS can use an ACM certificate so users connect over HTTPS.

That saves you from managing the certificate on the public edge yourself.

---

## Why This Matters for Ops

When certificates fail, production feels it fast.

You may see:

- Apps unreachable
- Login failures
- Webhook errors
- Health check problems

That is why certificate renewal and rotation should be part of your normal operations, not an emergency surprise.

---

## Summary

Kubernetes uses certificates to secure the control plane, user access, and app communication. The main idea is simple: trust the right identity and reject the wrong one.

If you understand server certs, client certs, and CA trust, the rest becomes easier. Next, Chapter 42 moves into creating TLS secrets so your apps can actually use certificates inside the cluster.
# Kubernetes Certificates

## A Real-World Problem

You have a Kubernetes app and TLS is working on the surface.

Then one morning the browser shows a warning.

The certificate expired.

Now users are blocked, and the team has to fix it quickly.

This is why you need to understand how certificates work in Kubernetes, not just how to install them once.

---

## What a Certificate Does

A certificate is a trust document.

It says:

- who the server is
- what domain names it belongs to
- who signed it

### Important Parts

| Part | Meaning |
|------|---------|
| Subject | Who the certificate belongs to |
| SAN | Extra names the cert covers |
| Issuer | Who signed it |
| Validity | Start and end date |
| Key | Secret used to prove identity |

### Simple Story

Think of a certificate like a badge at a data center.

The badge says who you are.

The private key is the thing only you can use to prove the badge is yours.

---

## Kubernetes and TLS Objects

In Kubernetes, certificates often appear in these places:

- Ingress TLS secrets
- API server certificates
- kubelet certificates
- webhook certificates
- service certificates

This chapter focuses mostly on app TLS, but it helps to know the cluster also uses certificates in many internal places.

---

## Certificate Chain

Certificates usually come in a chain:

1. Root CA
2. Intermediate CA
3. Server certificate

### Why This Matters

Your browser or client trusts the root CA.

The root CA signs the intermediate CA.

The intermediate CA signs the server certificate.

If any link in the chain is broken, trust fails.

---

## Server Certificate Example

```text
Subject: shop.example.com
Issuer: Let's Encrypt R3
Valid From: 2026-04-01
Valid Until: 2026-06-30
SANs: shop.example.com, www.shop.example.com
```

This certificate says that the server is valid for both names.

That is why SANs are so important.

---

## Kubernetes Secret as Certificate Storage

Kubernetes usually stores TLS certs in a Secret.

### Example Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
	name: shop-tls
	namespace: production
type: kubernetes.io/tls
data:
	tls.crt: <base64-encoded-certificate>
	tls.key: <base64-encoded-private-key>
```

The secret contains:

- `tls.crt` for the certificate
- `tls.key` for the private key

### Why a Secret?

Because Kubernetes needs a standard way to store and mount sensitive data.

But remember:

- Secrets are not magic vaults
- You still need RBAC
- You still need proper access control

---

## Certificate Renewal Pain

Manual certificates cause trouble.

Imagine this story:

- One engineer requests the certificate
- Another engineer installs it
- Nobody tracks the expiry date
- Two months later it expires
- Production breaks

That is a common ops story.

### Why Manual Management Fails

- People forget expiry dates
- Files are copied into the wrong namespace
- Keys are stored in the wrong place
- Renewal is not automated

This is exactly the kind of problem cert-manager helps solve later in the module.

---

## Example: TLS Secret for an Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: shop-ingress
	namespace: production
spec:
	tls:
		- hosts:
				- shop.example.com
			secretName: shop-tls
```

The Ingress reads the certificate from the Secret.

If the secret is wrong or missing, HTTPS fails.

---

## Where Certificates Go Wrong

### Problem 1: Wrong Domain Name

If the certificate says `api.example.com` but users visit `shop.example.com`, the browser warns the user.

### Problem 2: Expired Certificate

If the expiry date passes, browsers reject it.

### Problem 3: Missing Private Key

Without the key, the server cannot prove it owns the certificate.

### Problem 4: Wrong Namespace

If the certificate Secret is in the wrong namespace, the Ingress cannot use it.

---

## Practical Workflow

Here is the usual flow in a Kubernetes project:

1. Buy or request a certificate
2. Convert it into a Kubernetes Secret
3. Reference the Secret in an Ingress
4. Test HTTPS
5. Track expiry
6. Renew before users notice a problem

That is fine for small systems, but it becomes too much work when you have many apps.

---

## AWS Example

On AWS, you may use:

- ACM for managed certificates at the load balancer
- Kubernetes Secrets for app-level TLS
- cert-manager for automatic issuance and renewal

Which option you choose depends on where you want TLS to end.

If the load balancer handles TLS, the app may only see internal HTTP.

If the app handles TLS, the cert is usually stored in a Secret.

---

## Summary

Kubernetes certificates are the trust pieces that make TLS work. They live inside Secrets, must match the right domain names, and must be renewed before expiry.

If you understand the certificate chain, SANs, and Secrets, you are ready to handle TLS in real cluster work.

Next, the story moves from certificates to actually creating TLS Secrets for your app.
