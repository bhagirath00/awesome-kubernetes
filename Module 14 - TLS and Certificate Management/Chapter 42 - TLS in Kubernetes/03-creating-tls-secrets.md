# Creating TLS Secrets

## Turning a Certificate Into Something Kubernetes Can Use

You now know what TLS is and why Kubernetes cares about certificates.

The next step is practical: how do you put the certificate into Kubernetes so your app or ingress can use it?

The answer is a TLS Secret.

Think of it as the safe box where Kubernetes keeps the certificate and private key together.

---

## What a TLS Secret Contains

A TLS secret usually has two parts:

- `tls.crt` for the certificate
- `tls.key` for the private key

### Why This Pair Matters

The certificate tells the outside world who you are.

The private key proves that you own that certificate.

If someone gets both, they can pretend to be your server, so the private key must stay protected.

---

## Creating a TLS Secret by Hand

```bash
kubectl create secret tls shop-tls \
	--cert=tls.crt \
	--key=tls.key \
	-n production
```

This is the fastest way to create a TLS secret.

### What Happens Next

1. Kubernetes stores the secret in the namespace
2. Ingress or workload references the secret by name
3. Traffic can use that certificate for HTTPS

---

## Using the Secret in an Ingress

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
	rules:
		- host: shop.example.com
			http:
				paths:
					- path: /
						pathType: Prefix
						backend:
							service:
								name: shop-web
								port:
									number: 80
```

Now the ingress can serve HTTPS for `shop.example.com`.

---

## Secret Scope

Secrets are namespace-scoped.

That means a secret in `production` does not automatically exist in `dev`.

This is good because it keeps environments separate.

### Practical Tip

Use different secrets for:

- dev
- staging
- production

That keeps mistakes smaller.

---

## Base64 and Secret Data

Kubernetes stores secret data in base64 form.

That is not real encryption. It is just encoding.

### Why This Matters

Do not confuse encoding with security.

You still need RBAC, encryption at rest, and proper private key protection.

---

## Common Ways to Create TLS Secrets

### 1. Create From Files

```bash
kubectl create secret tls app-tls --cert=server.crt --key=server.key
```

### 2. Apply YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
	name: app-tls
	namespace: production
type: kubernetes.io/tls
data:
	tls.crt: <base64-encoded-cert>
	tls.key: <base64-encoded-key>
```

### 3. Create With cert-manager Later

That is the cleaner production method when you want automatic issuance and renewal.

---

## Checking the Secret

```bash
kubectl get secret shop-tls -n production
kubectl describe secret shop-tls -n production
```

You should see:

- Secret type: `kubernetes.io/tls`
- Data keys: `tls.crt`, `tls.key`

---

## Common Mistakes

### Mistake 1: Wrong File Name

If you swap the cert and key files, TLS will fail.

### Mistake 2: Wrong Namespace

If the ingress is in `production` but the secret is in `dev`, it will not work.

### Mistake 3: Missing Private Key

Without the private key, the secret is incomplete.

### Mistake 4: Expired Certificate

If the cert is expired, the browser or client will refuse it.

---

## AWS Use Case

You may use TLS secrets for internal services on EKS while using ACM at the public edge.

For example:

- ACM certificate on the ALB
- TLS secret for the internal app ingress
- Separate secret for staging

This gives you flexibility.

---

## Summary

TLS secrets are how you store certificates and private keys for Kubernetes workloads. You can create them by hand, reference them in an ingress, and keep them separate by namespace.

Once you know how secrets work, the next big step is automating certificate management. That is where cert-manager comes in, and Chapter 43 starts that story.
# Creating TLS Secrets

## Why This Step Matters

You already have the certificate and private key.

Now you need to put them into Kubernetes so your app can use them.

That is what a TLS Secret does.

This may sound like a tiny step, but if you get it wrong, HTTPS breaks.

---

## The Basic Secret Type

For TLS, Kubernetes uses the special secret type `kubernetes.io/tls`.

### Example

```yaml
apiVersion: v1
kind: Secret
metadata:
	name: shop-tls
	namespace: production
type: kubernetes.io/tls
data:
	tls.crt: <base64-certificate>
	tls.key: <base64-private-key>
```

The names are important:

- `tls.crt` for the certificate
- `tls.key` for the private key

If you use the wrong key names, Kubernetes tools and Ingress controllers may not read the secret correctly.

---

## Creating a Secret with Kubectl

If you have files already, you can create the secret with one command.

```bash
kubectl create secret tls shop-tls \
	--cert=tls.crt \
	--key=tls.key \
	-n production
```

This is a common DevOps workflow.

### What Happens Behind the Scenes

- Kubernetes reads the files
- It base64-encodes them
- It stores them in a Secret object
- The Ingress can reference that Secret later

---

## Using the Secret in an Ingress

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
	rules:
		- host: shop.example.com
			http:
				paths:
					- path: /
						pathType: Prefix
						backend:
							service:
								name: shop-web
								port:
									number: 80
```

The Ingress controller reads the TLS secret and serves HTTPS.

---

## Common Mistakes

### Mistake 1: Wrong Namespace

If the secret is in `dev` and the Ingress is in `production`, they do not connect.

### Mistake 2: Wrong File Names

If you create the secret with files that are not a valid cert and key pair, the browser will fail the handshake.

### Mistake 3: Wrong Host Name

If the host in the Ingress does not match the certificate SANs, users will see warnings.

### Mistake 4: Private Key Missing

The secret must contain both parts.

---

## Verifying the Secret

After you create it, check it.

```bash
kubectl get secret shop-tls -n production
kubectl describe secret shop-tls -n production
```

You can also inspect the Ingress.

```bash
kubectl describe ingress shop-ingress -n production
```

Look for:

- The correct secret name
- The correct host name
- The right namespace

---

## A Small Story

Suppose you are on call.

The site is down, and the only clue is a browser certificate warning.

You check the Secret and notice the certificate expired last week.

You replace the Secret, restart the ingress controller if needed, and the site starts working again.

That is why TLS Secret handling is a real operational skill, not just a YAML exercise.

---

## AWS EKS Practice

In EKS, this often happens in one of two ways:

- You terminate TLS at the AWS load balancer
- You terminate TLS inside the cluster using an ingress controller

If you terminate in the cluster, the TLS Secret becomes part of your app deployment flow.

That is why CI/CD pipelines often create or update TLS secrets as part of the release process.

---

## Summary

Creating a TLS Secret is the practical step that lets Kubernetes use your certificate and private key. The secret type, file names, namespace, and host name all have to match.

When those pieces line up, your Ingress can serve HTTPS cleanly. Next, the story moves from manual TLS work to cert-manager, which automates the painful parts for you.
