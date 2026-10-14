# Let’s Encrypt Integration

## Why Let’s Encrypt Matters

Let’s Encrypt gives you free public TLS certificates.

That makes it very useful for apps that need HTTPS on the public internet.

Instead of buying and manually renewing certificates, you can let cert-manager talk to Let’s Encrypt and handle the work for you.

---

## The Basic Idea

Let’s Encrypt wants proof that you control the domain.

cert-manager helps prove that control through one of two main methods:

- HTTP-01 challenge
- DNS-01 challenge

### Simple Story

Let’s Encrypt is like a guard asking, “Do you really own this website?”

cert-manager answers by showing a proof token on your domain or in DNS.

---

## HTTP-01 Challenge

The HTTP-01 challenge proves domain control by serving a special file over HTTP.

### When to Use It

- You already have public HTTP access
- Your ingress can route the challenge path
- You want a simpler setup

### Example Flow

1. cert-manager asks Let’s Encrypt for a cert
2. Let’s Encrypt gives a challenge
3. cert-manager serves a token at a special URL
4. Let’s Encrypt checks the token
5. If correct, certificate is issued

---

## DNS-01 Challenge

The DNS-01 challenge proves control by adding a DNS record.

### When to Use It

- You need wildcard certificates
- Your app is not public yet
- You want strong proof through DNS
- You use Route 53 in AWS

### Why AWS Teams Like It

If you use Route 53, cert-manager can create DNS records during the challenge.

That is a good fit for EKS workloads.

---

## Example ClusterIssuer for Let’s Encrypt

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
	name: letsencrypt-prod
spec:
	acme:
		server: https://acme-v02.api.letsencrypt.org/directory
		email: admin@example.com
		privateKeySecretRef:
			name: letsencrypt-prod-account-key
		solvers:
			- http01:
					ingress:
						class: nginx
```

This tells cert-manager:

- Use Let’s Encrypt production service
- Register with the email address
- Store the account key in a secret
- Use HTTP-01 through NGINX ingress

---

## Certificate Request Example

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
```

Once this is applied, cert-manager starts the issuance flow.

---

## Common AWS Setup

In AWS EKS, the common pattern is:

- Route 53 for DNS
- cert-manager for certificate automation
- Ingress controller for HTTPS traffic
- Let’s Encrypt for public certs

That creates a smooth public web setup.

---

## Troubleshooting Let’s Encrypt

### Problem 1: Challenge Fails

Check whether the challenge path is reachable or the DNS record exists.

### Problem 2: Wrong Ingress Class

If cert-manager points to the wrong ingress class, the challenge will not reach the solver.

### Problem 3: Rate Limits

Let’s Encrypt has rate limits.

Do not keep re-issuing broken certificates over and over in production.

---

## Production vs Staging

Use the Let’s Encrypt staging server when you are testing.

That avoids burning through production rate limits while you work out mistakes.

---

## Summary

Let’s Encrypt gives you free public certificates, and cert-manager automates the hard part of getting them. HTTP-01 is simple when your app is public, while DNS-01 is better for wildcard and Route 53-based setups.

Use staging first, then move to production when the challenge flow is working. Next, Chapter 43 moves to certificate issuers, which are the source of trust behind the certificates.
# Let’s Encrypt Integration

## Why People Like Let’s Encrypt

Let’s Encrypt gives free public certificates.

That matters because many teams want HTTPS without buying certificates for every small app.

For DevOps teams, it means less manual work and no license cost for basic public TLS.

---

## The Big Idea

cert-manager can talk to Let’s Encrypt and ask for certificates automatically.

You define the domain, and cert-manager handles the rest.

### Simple Story

You deploy a new app called `shop.example.com`.

You create a Certificate resource.

cert-manager proves that you own the domain.

Let’s Encrypt signs the certificate.

Your app gets HTTPS.

---

## Challenge Types

Let’s Encrypt usually uses ACME challenges.

| Challenge | Meaning | When It Is Used |
|-----------|---------|-----------------|
| HTTP-01 | Prove domain ownership with an HTTP path | Common for ingress-based apps |
| DNS-01 | Prove domain ownership with a DNS record | Good for wildcard certs |

### Which One Fits AWS EKS?

If you already have an ingress controller and a public host, HTTP-01 is often the simplest.

If you need wildcard certificates like `*.example.com`, DNS-01 is usually the better choice.

---

## ClusterIssuer Example

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
	name: letsencrypt-prod
spec:
	acme:
		server: https://acme-v02.api.letsencrypt.org/directory
		email: admin@example.com
		privateKeySecretRef:
			name: letsencrypt-prod-account-key
		solvers:
			- http01:
					ingress:
						class: nginx
```

This says:

- Use Let’s Encrypt production servers
- Register with the email address
- Store the account key in a Secret
- Use HTTP-01 through the NGINX ingress class

---

## Certificate Request Flow

Here is what happens in the real world:

1. You create a Certificate resource
2. cert-manager asks Let’s Encrypt for a cert
3. Let’s Encrypt asks for proof
4. cert-manager answers the challenge
5. Let’s Encrypt signs the certificate
6. cert-manager stores it in a Secret

That is the full path.

---

## Example Certificate

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

This tells cert-manager exactly what to request.

---

## Why Challenges Fail

### Problem 1: DNS Does Not Point to the Cluster

Let’s Encrypt cannot prove ownership if the domain does not reach the ingress.

### Problem 2: Ingress Class Mismatch

If the issuer says `nginx` but your cluster uses another ingress controller, the challenge may fail.

### Problem 3: Firewall or Security Group Blocking

In AWS, your security groups must allow the challenge traffic path.

### Problem 4: DNS-01 Credentials Missing

If you use DNS challenges, cert-manager needs access to your DNS provider.

---

## AWS Example

In AWS Route 53, you may use DNS-01 for wildcard certificates.

That is useful when you have many subdomains:

- api.example.com
- app.example.com
- admin.example.com

Instead of creating one certificate for each, you can manage them more cleanly.

---

## Best Practice

Use the Let’s Encrypt staging server first.

Why?

- It reduces the risk of rate limits
- It lets you test the flow safely
- It helps you catch mistakes before production

Then switch to production once the flow is stable.

---

## Summary

Let’s Encrypt integration lets cert-manager automate public certificate issuance. It is one of the biggest reasons teams adopt cert-manager in the first place.

Pick the right challenge type, test with staging, and make sure your DNS and ingress setup are correct. Next, the story moves to issuers, where you define who is allowed to sign certificates in your cluster.
