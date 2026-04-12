# Automatic Certificate Renewal

## Why Renewal Matters

Certificates do not last forever.

That is a problem because expired certificates can break your website, your APIs, or your internal tools at the worst possible time.

Manual renewal works when you have one or two certs.

It becomes a trap when you have dozens.

cert-manager solves that by renewing certificates before they expire.

---

## The Renewal Story

Here is the simple flow:

1. cert-manager watches the certificate
2. It notices the expiration date is getting close
3. It requests a new certificate
4. It updates the Kubernetes Secret
5. Your app keeps using the new certificate

That means no outage if the automation is set up correctly.

---

## Renewal Timing

cert-manager usually renews before the certificate expires.

That gives you a safety window.

### Why That Helps

If renewal fails once, you still have time to fix it before traffic breaks.

That is much better than waiting until the last minute.

---

## What Gets Updated

When cert-manager renews a certificate, it usually updates:

- The Secret containing the cert and key
- The certificate resource status
- The ingress or workload that reads that Secret

Your app may not even need a restart if it can reload certificates from the secret.

---

## Example Certificate With Renewal

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
	name: shop-example-com
	namespace: production
spec:
	secretName: shop-example-com-tls
	issuerRef:
		name: letsencrypt-prod
		kind: ClusterIssuer
	dnsNames:
		- shop.example.com
	renewBefore: 240h
```

`renewBefore` tells cert-manager to renew the certificate before it actually expires.

---

## Why Automatic Renewal Is Good DevOps

This is a classic DevOps win:

- Less manual work
- Fewer surprise outages
- Better compliance
- Easier scale

When certificate handling is automated, teams can focus on app reliability instead of calendar reminders.

---

## When Renewal Can Fail

Even automation can fail.

Common reasons include:

- DNS challenge failure
- Wrong issuer config
- Invalid ingress routing
- Rate limits from the CA
- Secret permission issues

### How to Think About It

If renewal fails, check the challenge path, the issuer, and the secret target.

That usually finds the problem fast.

---

## Checking Certificate Status

```bash
kubectl get certificate -n production
kubectl describe certificate shop-example-com -n production
```

Look for:

- Ready state
- Renewal time
- Secret name
- Events showing success or failure

---

## AWS EKS Production Pattern

In AWS, a common setup is:

- cert-manager for cert automation
- Route 53 for DNS challenge
- ALB or NGINX ingress for traffic
- Let’s Encrypt or private CA for trust

That gives you a neat production path with less manual babysitting.

---

## Best Practices

1. Use staging first before production CA
2. Watch certificate events in your monitoring stack
3. Keep DNS and ingress config clean
4. Use separate namespaces for dev and prod certs
5. Test renewal in a staging namespace before rollout

---

## Summary

Automatic renewal is one of the best reasons to use cert-manager. It keeps certificates fresh, reduces outages, and makes TLS management much easier in Kubernetes.

If you have read this far, you now have the whole story: TLS basics, Kubernetes certificates, TLS secrets, cert-manager, issuers, Let’s Encrypt, and renewal. That is enough to build secure HTTPS workflows in AWS EKS with much less manual work.
# Automatic Certificate Renewal

## The Real Pain Point

The first certificate is easy.

The hard part is what happens 60 or 90 days later.

If renewal is manual, somebody must remember the expiry date, create a new request, update the secret, and verify the app.

That is exactly the sort of work computers should do for you.

---

## How Renewal Works

cert-manager watches certificate expiry dates.

When a certificate gets close to expiration, cert-manager starts the renewal process before the old certificate stops working.

### Simple Story

The certificate is still valid today.

But cert-manager sees that it will expire soon.

So it creates a new request early.

The new certificate gets stored in the same secret.

The app keeps working without users noticing.

---

## Why Early Renewal Matters

If you renew too late, the app may already be broken.

If you renew early, you give yourself room to fix problems.

That is why automation is valuable.

---

## Renewal Flow

```text
Certificate nearing expiry
	↓
cert-manager notices it
	↓
New request is created
	↓
Issuer signs new certificate
	↓
Secret is updated
	↓
Ingress serves the new cert
```

---

## Checking Certificate Status

```bash
kubectl get certificate -n production
kubectl describe certificate shop-example-com -n production
```

Look for things like:

- Ready status
- Expiry date
- Renewal time
- Secret name

That tells you whether the certificate is healthy.

---

## Example Renewal Configuration

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
  issuerRef:
	 name: letsencrypt-prod
	 kind: ClusterIssuer
```

cert-manager handles the renewal timing automatically.

You usually do not need to create a new object just for renewal.

---

## What Can Break Renewal

### Problem 1: Domain No Longer Points Correctly

If DNS changes, the renewal challenge can fail.

### Problem 2: Ingress Controller Not Ready

The HTTP-01 challenge needs the ingress path to work.

### Problem 3: DNS Provider Permissions Missing

For DNS-01, cert-manager needs access to update DNS records.

### Problem 4: Bad Secret Access

If the secret cannot be updated, the new certificate will not take effect.

---

## Renewal in AWS EKS

In EKS, renewal usually fits one of these patterns:

- The ingress controller serves the renewed secret automatically
- A GitOps pipeline watches cert-manager resources
- The platform team monitors certificate expiry as a backup

That gives you both automation and visibility.

---

## Best Practices

### 1. Monitor expiry anyway

Automation is great, but you should still alert on expiry.

### 2. Test renewal in staging

Do not wait until production to find out your challenge path is broken.

### 3. Keep the issuer healthy

If the issuer or DNS provider fails, renewal can stop.

### 4. Use clear certificate names

Good names help you know what is expiring.

---

## A Simple Operational Story

Your app launches on Monday.

cert-manager issues the certificate.

Two months later, the certificate is near expiry.

cert-manager renews it quietly.

Nobody gets paged.

The site keeps working.

That is the real win.

---

## Summary

Automatic renewal is what makes cert-manager useful in day-to-day operations. It keeps certificates fresh before they expire, which prevents outages and removes a lot of manual work.

For DevOps teams, this means less firefighting and more confidence in production HTTPS.

That completes Module 14 and the certificate story from TLS basics to automated renewal.
