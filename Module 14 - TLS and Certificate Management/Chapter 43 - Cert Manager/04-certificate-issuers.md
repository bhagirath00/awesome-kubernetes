# Certificate Issuers

## What an Issuer Does

An Issuer is the thing cert-manager talks to when it wants a certificate.

Think of it like the person at the front desk who can request the passport or approve the ID card.

The issuer is not the certificate itself. It is the source or authority behind the certificate.

---

## Issuer vs ClusterIssuer

| Type | Scope | When to Use |
|------|-------|-------------|
| Issuer | One namespace | Only one app team or one namespace needs it |
| ClusterIssuer | Whole cluster | Many namespaces need the same issuer |

### Simple Rule

- Use Issuer for local, namespace-only trust
- Use ClusterIssuer when many teams or apps need the same certificate source

---

## Example Issuer

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
	name: selfsigned-issuer
	namespace: production
spec:
	selfSigned: {}
```

This creates a self-signed issuer in one namespace.

That can be useful for testing or internal use.

---

## Example ClusterIssuer

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
			name: letsencrypt-account-key
		solvers:
			- http01:
					ingress:
						class: nginx
```

This ClusterIssuer can be used across namespaces.

---

## Common Issuer Types

| Type | Meaning | Best Use |
|------|---------|----------|
| SelfSigned | cert-manager signs its own cert | Testing only |
| CA | A private CA signs the cert | Internal platforms |
| ACME | Public CA like Let’s Encrypt | Public websites |

### Picking the Right One

If the app is public, ACME is a common choice.

If the app is internal, a private CA or self-signed issuer may be better.

---

## How Issuers Work With Certificates

When you create a Certificate resource, it points to an issuer.

The flow is:

1. You request a cert
2. The issuer processes the request
3. cert-manager stores the cert in a Secret
4. Your app or ingress uses that Secret

That keeps the process clean and automated.

---

## Private CA Example

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
	name: internal-ca
	namespace: production
spec:
	ca:
		secretName: internal-ca-keypair
```

This is useful when you want internal trust without using a public certificate authority.

---

## AWS EKS Use Cases

In AWS, issuers are useful for:

- Public websites with Let’s Encrypt
- Internal APIs with private CA
- Test namespaces with self-signed certificates

That gives you one certificate system for many environments.

---

## Common Mistakes

### Mistake 1: Using ClusterIssuer for Everything

That may be too broad if only one namespace needs the issuer.

### Mistake 2: Wrong Secret Name

If the issuer references a secret that does not exist, issuance will fail.

### Mistake 3: Namespace Mismatch

An Issuer only works in its namespace.

If your Certificate is in another namespace, it will not find it.

---

## Summary

Issuers are the source cert-manager uses to create certificates. Use Issuer for one namespace and ClusterIssuer for the whole cluster.

Once you understand issuer scope, the final step is automation and renewal. Next, Chapter 43 moves to automatic certificate renewal so you do not get surprised by expiry.
# Certificate Issuers

## What an Issuer Does

An issuer is the signer behind the certificate request.

When cert-manager asks for a certificate, the issuer is the part that says yes and produces the signed result.

Think of it like this:

- The Certificate says what you want
- The Issuer says who signs it
- The Secret stores the final result

---

## Issuer vs ClusterIssuer

| Type | Scope | When to Use |
|------|-------|-------------|
| Issuer | One namespace | One app or team |
| ClusterIssuer | Whole cluster | Shared platform use |

### Simple Rule

- Use Issuer when the certificate should stay inside one namespace
- Use ClusterIssuer when many namespaces need the same signing source

---

## Namespace Issuer Example

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
	name: internal-ca
	namespace: production
spec:
	selfSigned: {}
```

This example creates a simple self-signed issuer.

That is useful for testing or internal use.

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

This issuer can be used across many namespaces.

That is a common setup in large Kubernetes clusters.

---

## Internal CA Story

Not every certificate must come from Let’s Encrypt.

Inside a company, you may use an internal CA for:

- Private dashboards
- Internal APIs
- Admin services
- mTLS between services

That gives you control when public trust is not needed.

---

## Choosing the Right Issuer

| Use Case | Better Choice |
|----------|---------------|
| Public website | ClusterIssuer with Let’s Encrypt |
| One team’s internal app | Issuer |
| Shared platform certs | ClusterIssuer |
| Internal-only trust | Internal CA |

The choice depends on scope and ownership.

---

## Common Issues

### Problem 1: Wrong Issuer Reference

If the Certificate points to a name that does not exist, issuance fails.

### Problem 2: Namespace Mismatch

An Issuer only works in its namespace.

### Problem 3: Solver Not Ready

If the ACME solver cannot answer the challenge, Let’s Encrypt will not sign the certificate.

### Problem 4: Permission Problems

cert-manager needs enough RBAC to create and update its resources.

---

## How This Fits in DevOps

Issuers help you keep certificate logic in code.

That means you can store the issuer YAML in Git, apply it with CI/CD, and make certificate behavior part of your normal deployment process.

That is much better than someone manually fixing TLS at 2 AM.

---

## Summary

Issuers define who signs certificates in cert-manager. Use Issuer for one namespace and ClusterIssuer for the whole cluster.

Pick Let’s Encrypt for public apps and internal CAs for private systems. Next, the story ends with automatic renewal, where cert-manager keeps certificates alive before users ever see a problem.
