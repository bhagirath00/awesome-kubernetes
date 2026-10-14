# What Is cert-manager?

## The Problem cert-manager Solves

Imagine you are the person running a busy Kubernetes platform.

Certificates are everywhere:

- Ingress TLS
- Internal services
- Webhooks
- Test environments
- Production apps

If you manage all of them by hand, life gets ugly very fast.

You end up chasing:

- Expiration dates
- Manual renewals
- Secret updates
- DNS verification steps

cert-manager exists to take that work off your plate.

---

## What cert-manager Does

cert-manager is a Kubernetes controller that automates certificate creation and renewal.

It can:

- Request certificates
- Renew certificates before they expire
- Update Kubernetes Secrets
- Work with public and private certificate issuers

### Simple Story

Instead of a human checking every certificate by hand, cert-manager becomes the worker that watches the clocks, asks for renewals, and replaces the secret when needed.

---

## Why DevOps Teams Like It

cert-manager fits DevOps because it reduces manual work and lowers the chance of outage.

### Without cert-manager

1. Certificate expires
2. Website breaks
3. Team gets paged
4. Someone finds the cert file
5. Secret is updated manually
6. Traffic comes back

### With cert-manager

1. Certificate is near expiry
2. cert-manager renews it automatically
3. Secret is updated
4. Traffic keeps working

That is the kind of boring automation you want.

---

## Main Building Blocks

| Resource | Role |
|----------|------|
| Issuer | A certificate source inside one namespace |
| ClusterIssuer | A certificate source for the whole cluster |
| Certificate | A request for a certificate |
| Secret | Where the issued certificate is stored |

The key idea is simple: you ask for a cert, and cert-manager does the rest.

---

## How the Flow Works

Here is the basic story:

1. You create a Certificate resource
2. cert-manager sees it
3. cert-manager talks to an Issuer or ClusterIssuer
4. The issuer creates or signs the certificate
5. cert-manager stores it in a Secret
6. Your ingress or app uses the Secret

That flow removes most manual steps.

---

## Example Use Case in AWS EKS

You run `api.example.com` on EKS.

You want HTTPS.

You could create certs by hand, but then you need to renew them before expiry.

With cert-manager:

- You define the certificate once
- It can use Let’s Encrypt or a private CA
- It renews itself later

That is easier to operate.

---

## cert-manager in the Bigger Picture

cert-manager works well with:

- Ingress controllers
- AWS Load Balancers
- Internal services
- GitOps workflows

It is not the certificate authority itself. It is the automation layer that talks to issuers and manages the result.

---

## Common Beginner Mistake

Many people think cert-manager creates trust by itself.

It does not.

It automates the certificate lifecycle, but the trust still comes from the issuer you choose.

That means you still need to know whether you are using:

- A public CA
- A private CA
- A self-signed issuer

---

## Summary

cert-manager automates certificate issuance and renewal in Kubernetes. It reduces manual work, lowers expiry risk, and fits well into DevOps workflows on AWS EKS.

If TLS is the lock on the door, cert-manager is the person who keeps making sure the key does not expire. Next, Chapter 43 moves to installing cert-manager so you can use it in a real cluster.
# What Is cert-manager?

## The Problem It Solves

Picture this.

Your team has five Kubernetes apps.

Each one needs HTTPS.

Each certificate expires on a different date.

Each secret must be updated by hand.

Soon the team is spending time on certificate chores instead of shipping features.

cert-manager solves that problem.

---

## Simple Definition

cert-manager is a Kubernetes tool that automates certificate requests, issuance, and renewal.

It watches Kubernetes resources and makes sure the right certificate appears at the right time.

### What It Handles

- Requests for new certificates
- Creating secrets from issued certificates
- Renewing certificates before they expire
- Working with certificate issuers like Let’s Encrypt or internal CAs

---

## Why DevOps Teams Use It

Manual TLS work is slow and error-prone.

cert-manager helps you build a repeatable process.

### DevOps Benefits

- Less manual work
- Fewer expired certificates
- Easier GitOps workflows
- Better cluster consistency

### Story Example

A platform team deploys a new app every week.

Without cert-manager, each release needs manual TLS setup.

With cert-manager, the app gets the certificate automatically.

That saves time and reduces mistakes.

---

## Core Objects

cert-manager uses a few key resources:

| Resource | Role |
|----------|------|
| Issuer | Issues certificates inside one namespace |
| ClusterIssuer | Issues certificates across namespaces |
| Certificate | Describes the certificate you want |
| Secret | Stores the final cert and key |

### Easy Way to Think About It

- `Certificate` says what you want
- `Issuer` says who signs it
- `Secret` stores the result

---

## Basic Workflow

Here is the typical flow:

1. You create a Certificate resource
2. cert-manager sees it
3. It asks an Issuer to sign it
4. The Issuer returns a certificate
5. cert-manager stores it in a Secret
6. Your Ingress uses that Secret

That is the whole story in a simple form.

---

## Certificate Example

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
	name: shop-example-com
	namespace: production
spec:
	secretName: shop-tls
	dnsNames:
		- shop.example.com
		- www.shop.example.com
	issuerRef:
		name: letsencrypt-prod
		kind: ClusterIssuer
```

This says:

- I want a certificate for `shop.example.com`
- Put it in the secret `shop-tls`
- Use the cluster issuer `letsencrypt-prod`

---

## cert-manager and AWS EKS

In AWS EKS, cert-manager is often used when:

- You want automatic HTTPS for many apps
- You do not want to manage cert files by hand
- You want staging and production issuers
- You want renewals to happen without alerts every few months

It works well with ingress controllers and GitOps setups.

---

## What Happens When Something Goes Wrong

### Problem 1: Certificate Never Appears

Maybe the issuer is wrong.

Maybe DNS is not ready.

Maybe cert-manager does not have enough permissions.

### Problem 2: Secret Not Updated

Maybe the certificate exists, but the app still uses an old secret.

### Problem 3: Renewal Fails

Maybe the domain challenge cannot complete.

You need logs and events to trace it.

---

## Why This Is Better Than Manual Work

Without cert-manager:

- People copy files around
- Expiry dates get missed
- Secrets drift from what is in Git

With cert-manager:

- The process is declarative
- The cluster keeps itself updated
- Renewal can happen before users notice a problem

---

## Summary

cert-manager automates TLS certificate work inside Kubernetes. It creates, renews, and stores certificates for you, which is a big win for busy DevOps teams.

Think of it as the helper that keeps HTTPS from becoming a manual pain point. Next, the story continues with how to install cert-manager in a real cluster.
