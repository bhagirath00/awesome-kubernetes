# Installing the AWS Load Balancer Controller

## Getting the Controller Into EKS

Once you know what the AWS Load Balancer Controller does, the next step is installation.

This is where the story turns into real platform work.

You need the controller, the right IAM permissions, and a cluster that can talk to AWS safely.

---

## Why IAM Matters

The controller must create and manage AWS resources.

That means it needs permissions for things like:

- Load balancers
- Target groups
- Listeners
- Security groups

You should not give it broad access just because it is convenient.

Use least privilege.

---

## Install Flow

The usual flow is:

1. Create an IAM policy
2. Create a service account with IRSA
3. Install the controller manifest or Helm chart
4. Check pod health
5. Confirm it can create AWS resources

That keeps AWS and Kubernetes connected in a controlled way.

---

## IRSA Concept

IRSA means IAM Roles for Service Accounts.

It lets a pod assume an IAM role without storing static AWS keys in a secret.

### Why This Is Good

- Better security
- Easier rotation
- Less secret sprawl

This is the standard AWS way to give Kubernetes workloads AWS permissions.

---

## Basic Install Example

```bash
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
	-n kube-system \
	--set clusterName=my-eks-cluster \
	--set serviceAccount.create=false \
	--set serviceAccount.name=aws-load-balancer-controller
```

This is the kind of install flow many AWS teams use.

---

## What to Check After Install

```bash
kubectl get pods -n kube-system | grep load-balancer
kubectl get deployment -n kube-system | grep load-balancer
```

You want the controller pod to be running cleanly.

---

## AWS EKS Notes

In EKS, the controller usually works together with:

- A cluster IAM role
- An OIDC provider
- AWS VPC networking

If those pieces are missing, the controller may start but fail when it tries to create a load balancer.

---

## Common Install Mistakes

### Mistake 1: Wrong IAM Policy

If the controller cannot create load balancers, the policy is likely too small or missing actions.

### Mistake 2: Wrong Cluster Name

The controller must know which EKS cluster it belongs to.

### Mistake 3: Missing OIDC Setup

Without OIDC, IRSA does not work the way it should.

---

## A Small Story

You install the controller in the morning.

By afternoon, a team adds an Ingress.

AWS creates the ALB automatically.

No one had to log into the console and wire things by hand.

That is the kind of automation that saves time every week.

---

## Summary

Installing the AWS Load Balancer Controller means connecting Kubernetes to AWS with the right IAM setup, then letting the controller manage ALBs and related objects for you.

IRSA is the key idea to remember because it avoids static credentials. Next, Chapter 46 moves to ALB configuration so you can see how the Ingress maps to AWS behavior.
