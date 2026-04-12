# TLS in Kubernetes Overview

## The Story Behind TLS

Imagine your app is a shop in a busy city.

People walk in and out all day. Some people are customers, some are workers, and some are strangers trying to listen in. If you speak in plain text, anyone nearby can hear your words. That is how unsecured network traffic feels.

TLS is the lock on the door and the private conversation inside the shop. It helps make sure the right person is talking to the right service, and that nobody can read the message in the middle.

In Kubernetes, TLS matters because your app traffic is not just moving across one machine. It moves across pods, nodes, load balancers, and sometimes the internet.

---

## What TLS Does

TLS gives you three main things:

| Goal | What It Means | Why You Care |
|------|---------------|--------------|
| Encryption | Data is hidden from others | People cannot read traffic in transit |
| Identity | You know who you are talking to | Clients can trust the server |
| Integrity | Data is not changed on the way | Messages arrive exactly as sent |

Think of it like sending a sealed letter instead of yelling your order across the room.

---

## Where TLS Shows Up in Kubernetes

TLS is common in many places inside a cluster:

- Ingress controllers serving HTTPS
- Internal APIs speaking to each other
- Webhooks that need secure calls
- Metrics and dashboard tools
- cert-manager creating and renewing certificates

If you use AWS, TLS is also common in front of ALBs, NLBs, and public services.

---

## HTTP vs HTTPS

HTTP is plain text.

HTTPS is HTTP wrapped inside TLS.

### Example

```text
HTTP:
GET /login
username=alice&password=123456

HTTPS:
Encrypted data
Only the client and server can read it
```

If someone captures HTTP traffic, they may see everything.

If someone captures HTTPS traffic, they only see encrypted data.

---

## The TLS Handshake Story

TLS does not just start working in one step. It goes through a handshake.

Here is the simple story:

1. The client says hello
2. The server sends its certificate
3. The client checks the certificate
4. Both sides agree on encryption keys
5. Secure communication starts

### Why the Handshake Matters

The handshake is where trust begins.

If the certificate is wrong, expired, or signed by someone you do not trust, the client should stop.

That is what protects you from fake servers.

---

## Certificates and Keys

A certificate is like a digital ID card.

It says who the server is and who signed the ID.

### Main Parts

| Part | Meaning |
|------|---------|
| Private key | Secret key that must stay safe |
| Public key | Shared key inside the certificate |
| Certificate | Signed document with identity info |
| CA | Certificate Authority that signs certs |

### Real-Life Analogy

If the certificate is your passport, the private key is the stamp machine only you should have.

If someone gets the private key, they can pretend to be your server.

---

## TLS in AWS EKS

In an AWS EKS cluster, TLS is often used at the edge and inside the cluster.

Common places include:

- ALB Ingress Controller with HTTPS
- Private APIs with internal certificates
- Web apps using ACM certificates
- Services behind an NGINX ingress controller

### Example Scenario

You run a shopping app on EKS.

- Customers visit `https://shop.example.com`
- The load balancer handles TLS at the edge
- The ingress sends traffic to the service
- Internal APIs may also use TLS between pods

That setup protects both the user and the platform.

---

## Common TLS Terms

| Term | Meaning |
|------|---------|
| Certificate Authority (CA) | Trusted signer of certificates |
| CSR | Certificate Signing Request |
| SAN | Subject Alternative Name, extra hostnames in a cert |
| PEM | Common text format for certs and keys |
| CRT | Certificate file |
| KEY | Private key file |

You do not need to memorize every term at once. Start with the idea that a certificate proves identity.

---

## Why TLS Matters for DevOps

DevOps is not just about shipping fast.

It is also about shipping safely.

TLS helps because:

- It protects customer data
- It supports compliance
- It reduces risk during service communication
- It builds trust in your platform

If a service mesh, ingress, or external load balancer is part of your stack, TLS is never far away.

---

## A Small Example of Secure HTTPS Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
	name: web-ingress
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
								name: web
								port:
									number: 80
```

This tells Kubernetes to use a TLS secret for the hostname.

The secret holds the certificate and private key.

---

## What Can Go Wrong

TLS problems often show up in simple ways:

- Browser says certificate is not trusted
- Service refuses connection
- Certificate expired
- Hostname does not match certificate
- Private key does not match certificate

These are the kinds of problems you want to catch early in staging.

---

## Summary

TLS protects traffic by encrypting it, proving identity, and keeping data intact. In Kubernetes, it shows up in ingress, internal services, webhooks, and any place where trust matters.

If you understand the handshake and the role of certificates, you are already ahead of many people starting out with Kubernetes security. Next, Chapter 42 moves into the certificates themselves and how they work inside the cluster.
# TLS in Kubernetes

## A Simple Story to Start

Imagine you run a small online store on Kubernetes.

At first, the app works fine. People visit the site, place orders, and the backend handles requests.

Then one day someone asks a very normal question:

> "Is the traffic between the browser and the app safe?"

That is where TLS enters the story.

TLS protects data while it moves across the network. Without it, anyone who can watch traffic may read passwords, session tokens, and API data.

In DevOps work, TLS is not a fancy extra. It is part of the basic safety bar.

---

## What TLS Does

TLS gives you three things:

| Goal | What it means |
|------|---------------|
| Encryption | Data is hidden while moving across the network |
| Identity | The client can check that the server is real |
| Integrity | Data cannot be changed without detection |

Think of TLS like a sealed truck carrying valuable boxes.

- Encryption is the locked truck
- Identity is the driver badge check
- Integrity is the seal that shows if someone opened the box

---

## Why TLS Matters in Kubernetes

Kubernetes makes it easy to run many services.

That is good, but it also means there are many network paths:

- Browser to ingress
- Ingress to service
- Service to service
- Pod to database

If those paths are open and unprotected, you create risk.

### Common Real-World Risks

- A user logs in over plain HTTP
- An internal API call is intercepted
- A token is exposed in transit
- A service gets a fake certificate

TLS reduces those risks.

---

## TLS in the Request Flow

Here is the normal path in an AWS EKS app:

```text
User browser
	 ↓ TLS
AWS Load Balancer or Ingress
	 ↓ TLS or internal routing
Kubernetes Service
	 ↓
Pod
```

Sometimes TLS ends at the ingress.

Sometimes TLS continues all the way to the pod.

That choice depends on your security needs.

---

## Basic Terms You Must Know

| Term | Meaning |
|------|---------|
| Certificate | A file that proves identity |
| Private key | Secret key used to prove ownership |
| Public key | Key shared with others |
| CA | Certificate Authority, the trusted signer |
| CSR | Certificate Signing Request |
| Secret | Kubernetes object that stores the certificate and key |

### Simple Example

The certificate is like an ID card.

The private key is like the password that proves the ID card belongs to you.

Never share the private key.

---

## TLS in Plain English

When a browser connects to a server:

1. The server sends its certificate
2. The browser checks if the certificate is trusted
3. They agree on a shared session key
4. All traffic after that is encrypted

The session key is fast to use, so TLS does not slow everything down too much.

---

## Kubernetes TLS Use Cases

### 1. HTTPS for public apps

Your app should not accept login requests over plain HTTP.

### 2. Internal API protection

Service-to-service calls inside the cluster should be encrypted when the data matters.

### 3. Admin dashboards

Things like internal dashboards and metrics endpoints should never be exposed without TLS.

### 4. Compliance

Many companies need TLS for audit and security rules.

---

## Example: Public App in AWS EKS

Suppose you deploy a shopping app behind an AWS Application Load Balancer.

Without TLS:

```text
Browser → HTTP → Ingress → App
```

With TLS:

```text
Browser → HTTPS → Ingress → App
```

That one change protects login and checkout traffic.

### Simple Ingress Example

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

This says the Ingress should use a TLS secret called `shop-tls`.

---

## TLS Modes You Will See

| Mode | Meaning | Typical Use |
|------|---------|-------------|
| Termination | TLS ends at the ingress | Public HTTPS apps |
| Re-encryption | TLS continues to backend | Sensitive internal apps |
| Passthrough | Ingress passes TLS through | Apps that manage their own TLS |

### Story Example

If your app is a simple blog, TLS termination at the ingress is often enough.

If your app handles payments, you may want re-encryption so traffic stays encrypted deeper inside the cluster.

---

## What Happens If You Skip TLS

Without TLS:

- Passwords can be sniffed
- Session cookies can be stolen
- API calls can be read
- Users may lose trust

That is why TLS is one of the first things you should add when an app goes live.

---

## Practical DevOps Advice

Start simple:

1. Put TLS on the public edge first
2. Store the certificate in a Kubernetes Secret
3. Test that HTTPS works
4. Move to internal TLS later if needed

Do not try to solve everything at once.

If you are learning, think of TLS as the first security fence around the app.

---

## Summary

TLS keeps Kubernetes traffic safe by encrypting data, proving identity, and protecting integrity. In AWS EKS, it is usually the first layer of protection for public apps and an important layer for internal services too.

Start with the public edge, learn the certificate flow, and then expand to deeper internal traffic if your app needs it.

Next, the story continues with the certificates that make TLS work inside Kubernetes.
