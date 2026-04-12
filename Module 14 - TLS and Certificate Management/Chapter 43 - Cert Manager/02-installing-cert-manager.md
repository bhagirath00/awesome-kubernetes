# Installing cert-manager

## Getting cert-manager Into the Cluster

Once you know what cert-manager does, the next step is to install it.

That usually means adding its CRDs and controller pods to your cluster so it can start watching certificate resources.

Think of it like hiring the worker and giving them the clipboard they need to do the job.

---

## What You Need First

Before installation, check:

- You have cluster admin access
- Your Kubernetes version is supported
- You know which ingress or issuer you plan to use later
- You have DNS and internet access if you will use public issuers

---

## Standard Installation Flow

The most common install path is Helm.

### Example

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
	--namespace cert-manager \
	--create-namespace \
	--set crds.enabled=true
```

This does two important things:

- Installs cert-manager into its own namespace
- Creates the required CRDs

---

## Why the CRDs Matter

cert-manager introduces custom resources like:

- Certificate
- Issuer
- ClusterIssuer

Without the CRDs, Kubernetes does not know what those objects are.

That is why CRDs are part of the install.

---

## Checking the Installation

After install, verify the pods are running.

```bash
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager
```

You should see controller pods and the cert-manager CRDs.

---

## What the Main Pods Do

| Pod | Job |
|-----|-----|
| cert-manager | Watches certificate resources |
| cainjector | Injects CA data where needed |
| webhook | Validates cert-manager resources |

Each one has a clear job.

---

## Common Installation Choices

### Namespace Choice

Most teams install cert-manager in a separate namespace called `cert-manager`.

That keeps the platform components easy to find.

### Version Choice

Always use a version that matches your cluster and your support plan.

### CRD Installation

If you skip CRDs, the install will not work correctly.

---

## AWS EKS Notes

On EKS, cert-manager is often used with:

- ALB Ingress Controller
- Route 53 DNS challenges
- ACM for public endpoints

If you plan to use Let’s Encrypt DNS validation, you also need AWS credentials or IAM access that lets cert-manager manage DNS records.

---

## Troubleshooting Installation

### Problem 1: Pods Stay Pending

Check cluster resources and namespace status.

### Problem 2: CRDs Missing

Run:

```bash
kubectl get crd | grep cert-manager
```

If nothing shows up, the install did not create the CRDs.

### Problem 3: Webhook Errors

If the webhook is not ready, resource validation may fail.

Check the webhook pod logs and make sure DNS works inside the cluster.

---

## Summary

Installing cert-manager is mostly about getting the controller, webhook, and CRDs into the cluster the right way. Helm is the usual path, and the `cert-manager` namespace keeps things tidy.

Once it is installed, you can start asking it to issue certificates for real workloads. Next, Chapter 43 moves to Let’s Encrypt integration so you can issue public certificates automatically.
# Installing cert-manager

## Start With the Real Need

You do not install cert-manager just because it exists.

You install it because your cluster needs automatic TLS management.

Maybe you have many apps.

Maybe certificates expire too often.

Maybe your team wants one standard way to manage HTTPS.

That is the point where cert-manager becomes useful.

---

## Installation Options

There are a few ways to install cert-manager:

| Method | Best For |
|--------|----------|
| Helm | Teams already using Helm |
| kubectl + manifests | Simple manual installs |
| GitOps | Clusters managed by Argo CD or Flux |

In DevOps, Helm and GitOps are the most common choices.

---

## Typical Install Flow

The install usually looks like this:

1. Add the cert-manager Helm repo or manifests
2. Install the CRDs
3. Deploy the cert-manager controller
4. Confirm the pods are running
5. Create an Issuer or ClusterIssuer
6. Request a certificate

That gives the cluster the objects it needs to start working.

---

## Helm Example

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
	--namespace cert-manager \
	--create-namespace \
	--set crds.enabled=true
```

This is a common install style for production teams.

### Why CRDs Matter

cert-manager uses custom resources like Certificate and Issuer.

Those resources only work if the CRDs exist first.

---

## Checking the Installation

After installation, verify the components.

```bash
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager
kubectl get deployment -n cert-manager
```

You want to see:

- cert-manager controller running
- webhook running
- cainjector running

If one of them is missing, the system may not work correctly.

---

## AWS EKS Notes

On EKS, you should make sure:

- The cluster can reach external issuers like Let’s Encrypt if needed
- DNS records are set up correctly
- RBAC allows cert-manager to manage its resources
- The ingress controller is ready to use the generated secrets

That keeps the install from failing later during issuance.

---

## Common Install Mistakes

### Mistake 1: Forgetting the CRDs

If the CRDs are missing, cert-manager objects will not work.

### Mistake 2: Installing in the Wrong Namespace

cert-manager components should live where the cluster expects them.

### Mistake 3: Not Checking Pod Health

Sometimes the install command succeeds but one pod is stuck.

Always verify.

### Mistake 4: Assuming TLS Will Work Without Issuers

cert-manager is the engine, but it still needs an Issuer or ClusterIssuer to sign certificates.

---

## A Simple Deployment Story

You add cert-manager to the cluster.

The platform team creates a ClusterIssuer.

The app team creates a Certificate.

The Secret appears automatically.

The Ingress starts serving HTTPS.

That is the value of a good installation.

---

## Summary

Installing cert-manager means setting up the controller, CRDs, and supporting pods so the cluster can automate certificate work.

In AWS EKS, a clean install is the foundation for automatic TLS. Once the installation is stable, you can connect it to Let’s Encrypt and start issuing certificates with very little manual effort.

Next, the story moves to Let’s Encrypt integration, where cert-manager starts talking to a real public certificate authority.
