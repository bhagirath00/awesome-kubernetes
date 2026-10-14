# What is Kubernetes

## Introduction

Before Kubernetes existed, running applications in production was painful. You had to manually decide which server to deploy to, manually restart crashed processes, and manually scale when traffic spiked. Kubernetes was built to solve all of that automatically.

Kubernetes is an open-source container orchestration platform. It takes your containerized applications and handles deploying them, keeping them running, scaling them up or down, and replacing them when they fail — all without human intervention.

In this file, you will learn what Kubernetes actually is, why it was created, and what problems it solves that Docker alone cannot.

---

## The Problem Kubernetes Solves

Containers solved the problem of "it works on my machine." You package your application and all its dependencies into a container image, and it runs the same everywhere.

But containers alone do not solve the problem of running those containers in production at scale. Imagine you have 10 microservices, each running in Docker containers, deployed across 20 servers. Now ask yourself:

- Which server should each container run on?
- What happens when a container crashes at 3 AM?
- How do you roll out a new version without downtime?
- How do you scale container 5 from 2 copies to 10 copies when traffic spikes?
- How do containers talk to each other across different servers?
- How do you limit how much CPU and memory each container uses?

With plain Docker, the answer to all of these is: **you do it manually**. That does not work at scale.

Kubernetes answers all of these questions automatically. It acts as the operating system for your cluster — just like Linux manages processes on a single machine, Kubernetes manages containers across a fleet of machines.

---

## What Kubernetes Actually Does

Think of Kubernetes as a very smart supervisor for your containers.

You tell Kubernetes: "I want 3 copies of my web application running at all times, each needing 500MB of RAM and 0.5 CPU cores, and if any copy crashes, replace it immediately."

Kubernetes then:
1. Finds machines in your cluster with enough free resources
2. Starts the containers on those machines
3. Continuously checks that all 3 copies are running
4. Replaces any copy that crashes — automatically
5. Balances traffic across all 3 copies
6. Updates to a new version without taking the app offline

You describe what you want. Kubernetes figures out how to make it happen and keeps it that way.

This model is called **declarative management** — you declare the desired state, and Kubernetes reconciles the actual state to match it continuously.

---

## A Short History

Kubernetes was created by Google in 2014. Google had been running containers internally for over a decade using a system called **Borg**, which managed hundreds of thousands of containers across Google's data centers.

When Google open-sourced Kubernetes, they donated it to the **Cloud Native Computing Foundation (CNCF)** in 2016. Today Kubernetes is maintained by a massive community of contributors from Google, Microsoft, Red Hat, Amazon, and hundreds of other companies.

The name "Kubernetes" comes from the Greek word for helmsman or ship pilot — the person who steers the ship. The logo is a ship's wheel (helm). This is why the Kubernetes package manager is called **Helm**.

The abbreviation **K8s** (K + 8 letters + s) is used everywhere in the community.

---

## Kubernetes vs Docker — The Right Mental Model

A very common confusion: **Kubernetes and Docker are not competitors. They work together.**

| Tool | What It Does |
|------|-------------|
| **Docker** | Builds and runs a single container on a single machine |
| **Kubernetes** | Manages hundreds of containers across many machines |

Docker is the engine that creates and runs containers. Kubernetes is the platform that decides where to run them, keeps them healthy, and connects them together.

Think of it this way:
- Docker is the shipping container
- Kubernetes is the port that organizes, routes, and manages thousands of shipping containers

In a Kubernetes cluster, when Kubernetes needs to start a container, it tells the container runtime (Docker, or more commonly now **containerd**) to start it. Kubernetes itself does not start containers directly — it delegates that to the runtime.

```
Your App Code
     ↓
Docker Image (build once, run anywhere)
     ↓
Kubernetes (decides where, how many, keeps them running)
     ↓
containerd / Docker (actually starts the container)
     ↓
Linux Kernel (runs the process)
```

---

## The Core Kubernetes Concepts (Quick Overview)

You will go deep on each of these in later chapters. For now, understand what each concept is at a high level.

### Cluster
The entire Kubernetes environment — a group of machines (nodes) working together. Every Kubernetes deployment is a cluster.

### Node
A single machine (physical server or virtual machine) in the cluster. There are two types:
- **Control Plane nodes** — the brain of the cluster (scheduling, API, state)
- **Worker nodes** — where your application containers actually run

### Pod
The smallest deployable unit in Kubernetes. A Pod is a wrapper around one or more containers that share the same network and storage. Every container in Kubernetes runs inside a Pod.

```yaml
# The simplest possible Pod definition
apiVersion: v1
kind: Pod
metadata:
  name: my-web-app
spec:
  containers:
  - name: web
    image: nginx:1.25
    ports:
    - containerPort: 80
```

### Deployment
A Deployment manages Pods for you. You tell it how many copies (replicas) to run, what container image to use, and it handles the rest — including rolling updates and rollbacks.

```yaml
# A Deployment running 3 copies of nginx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Service
A Service gives your Pods a stable network address. Pods are temporary — they are created and destroyed constantly. A Service acts as a permanent, stable entry point that always routes traffic to healthy Pods, regardless of which specific Pods are running.

### Namespace
A way to divide one cluster into multiple virtual clusters. You might have a `dev` namespace, a `staging` namespace, and a `production` namespace — all on the same physical cluster, but logically separated.

---

## Where Kubernetes Runs

Kubernetes runs on any infrastructure:

| Environment | How You Use K8s |
|-------------|----------------|
| **AWS** | Amazon EKS (Elastic Kubernetes Service) — managed control plane |
| **Google Cloud** | GKE (Google Kubernetes Engine) — original managed K8s |
| **Azure** | AKS (Azure Kubernetes Service) |
| **On-premise** | Self-managed K8s on your own servers |
| **Local (development)** | Minikube, Kind, k3d — single-node clusters on your laptop |

In this course, AWS EKS is the primary platform we use. EKS runs the control plane for you — you only manage the worker nodes (or use Fargate to not even manage those).

---

## What Kubernetes Does NOT Do

It is equally important to understand what Kubernetes is not responsible for:

- **It does not build your Docker images** — that is Docker's job (or a CI/CD tool like Jenkins)
- **It does not store your source code** — that is Git's job
- **It does not provide a CI/CD pipeline** — that is Jenkins, GitLab CI, or GitHub Actions
- **It does not monitor your application metrics** — that is Prometheus, Datadog, CloudWatch
- **It does not manage your database data** — it manages the container running the database, not the data inside it

Kubernetes is one piece of a larger DevOps toolchain. It handles container orchestration. Other tools handle the rest.

---

## A Real-World Example

Here is how Kubernetes fits into a real DevOps workflow on AWS:

```
Developer pushes code to GitHub
         ↓
Jenkins pipeline triggers
         ↓
Jenkins builds Docker image
         ↓
Jenkins pushes image to Amazon ECR (container registry)
         ↓
Jenkins applies updated Deployment to EKS cluster
         ↓
Kubernetes schedules new Pods on worker nodes
         ↓
Kubernetes performs rolling update (zero downtime)
         ↓
Old Pods removed, new Pods serve traffic
         ↓
Users never notice the update happened
```

Every step in this flow has a specific tool. Kubernetes owns the last four steps — scheduling, updating, health-checking, and traffic routing.

---

## Why Every DevOps Engineer Must Know Kubernetes

Kubernetes has become the standard platform for running production workloads in the cloud. As of 2024:

- Over 5.6 million developers use Kubernetes
- 96% of organizations are using or evaluating Kubernetes (CNCF survey)
- Every major cloud provider offers a managed Kubernetes service
- Most DevOps and SRE job postings list Kubernetes as a required skill
- The salary premium for Kubernetes expertise is significant

If you are building a career in DevOps, cloud engineering, or platform engineering, Kubernetes knowledge is not optional — it is a core requirement.

---

## Summary

- Kubernetes is a container orchestration platform that automates deploying, scaling, and managing containers across a cluster of machines
- It was created by Google in 2014, based on their internal Borg system, and donated to the CNCF
- Docker builds and runs individual containers; Kubernetes manages containers at scale across many machines — they work together, not against each other
- The core Kubernetes objects are: Cluster, Node, Pod, Deployment, Service, and Namespace
- Kubernetes runs on AWS (EKS), Google Cloud (GKE), Azure (AKS), on-premise, or locally for development
- Kubernetes handles orchestration only — it does not build images, run CI/CD pipelines, or store code
- K8s knowledge is a required skill for modern DevOps roles

In the next file, `02-kubernetes-vs-docker-swarm.md`, you will compare Kubernetes with Docker Swarm — understanding when to use each and why Kubernetes won the orchestration war.
