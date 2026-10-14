# Volumes in Kubernetes

## Introduction

A **Volume** in Kubernetes is a directory that is accessible to containers in a Pod. Unlike a container's default filesystem (which is wiped on restart), a volume has a lifetime defined by its type — some last as long as the Pod, some survive the Pod, and some are backed by external storage that persists indefinitely.

Volumes are defined in the Pod spec and mounted into containers at specific paths. Multiple containers in the same Pod can share the same volume.

---

## How Volumes Work — The Basic Structure

Every volume in Kubernetes involves two parts in the Pod spec:

**1. The volume definition** — declared in `spec.volumes`. This is the "what" — what kind of storage source is this?

**2. The volume mount** — declared in `spec.containers[].volumeMounts`. This is the "where" — where inside the container is the volume accessible?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  namespace: production
spec:
  # PART 1: Define the volume (once, at Pod level)
  volumes:
  - name: shared-data          # volume name — referenced by mounts
    emptyDir: {}               # volume type — emptyDir in this case
  
  - name: app-config           # second volume
    configMap:
      name: app-config-cm

  containers:
  - name: web
    image: nginx:1.25
    # PART 2: Mount the volume (per container)
    volumeMounts:
    - name: shared-data        # references the volume name above
      mountPath: /var/shared   # path inside THIS container
    - name: app-config
      mountPath: /etc/nginx
      readOnly: true
  
  - name: sidecar
    image: busybox:1.35
    volumeMounts:
    - name: shared-data        # SAME volume — shared with web container
      mountPath: /data         # different path — but same files
```

Both containers (`web` and `sidecar`) access the same `shared-data` volume. A file written by `web` at `/var/shared/output.txt` is immediately readable by `sidecar` at `/data/output.txt`.

---

## Volume Lifetime

A volume's lifetime depends on its type:

```
Pod-scoped volumes (die with the Pod):
  emptyDir     → created when Pod starts, deleted when Pod stops

Node-scoped volumes (survive Pod, tied to the node):
  hostPath     → uses a directory on the node's filesystem

Cluster-scoped volumes (survive Pod and node):
  PersistentVolumeClaim → backed by EBS, EFS, NFS, etc.

External volumes (fully independent):
  awsElasticBlockStore → EBS volume directly (legacy, replaced by PVC)
  nfs                  → NFS share
```

---

## Volume Types — The Full List

Kubernetes supports many volume types. Here are the most important ones for EKS:

| Volume Type | Backed By | Lifetime | Best For |
|------------|-----------|----------|---------|
| `emptyDir` | Node disk or RAM | Pod | Scratch space, container sharing |
| `hostPath` | Node filesystem | Node | DaemonSet node access, dangerous otherwise |
| `configMap` | Kubernetes ConfigMap | Pod (but config auto-updates) | Config files |
| `secret` | Kubernetes Secret | Pod (but secret auto-updates) | Credentials, TLS certs |
| `persistentVolumeClaim` | EBS/EFS/NFS/etc. | Independent (PV lifetime) | All persistent data |
| `projected` | Multiple sources combined | Pod | Combine secret + configMap + SA token |
| `downwardAPI` | Pod metadata | Pod | Expose Pod info as files to the app |
| `csi` | Any CSI driver | Depends on driver | Modern extensible storage |

---

## Sharing Volumes Between Containers

Volumes are the correct mechanism for inter-container file sharing within a Pod:

```yaml
spec:
  volumes:
  - name: log-share
    emptyDir: {}

  containers:
  # Main app writes logs to a file
  - name: app
    image: my-app:v1.0
    volumeMounts:
    - name: log-share
      mountPath: /var/log/app

  # Log shipper reads from same directory and sends to CloudWatch
  - name: log-shipper
    image: amazon/aws-for-fluent-bit:latest
    volumeMounts:
    - name: log-share
      mountPath: /var/log/app   # reads same files
```

---

## The `projected` Volume — Combining Multiple Sources

A `projected` volume maps several existing volume sources into a single directory:

```yaml
spec:
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: db-credentials      # files: username, password
      - configMap:
          name: app-config           # files: settings.json, nginx.conf
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600    # short-lived projected SA token
          audience: "https://kubernetes.default.svc"
      - downwardAPI:
          items:
          - path: pod-name
            fieldRef:
              fieldPath: metadata.name    # file: pod-name

  containers:
  - name: app
    volumeMounts:
    - name: all-in-one
      mountPath: /etc/app
      # /etc/app/username     ← from secret
      # /etc/app/password     ← from secret
      # /etc/app/settings.json ← from configmap
      # /etc/app/token        ← projected SA token
      # /etc/app/pod-name     ← from downwardAPI
```

This is commonly used in EKS IRSA (IAM Roles for Service Accounts) — the projected service account token is what the AWS SDK uses to assume the IAM role.

---

## The `downwardAPI` Volume — Exposing Pod Metadata as Files

The Downward API lets you expose Pod metadata (name, namespace, labels, resource limits) as files inside the container:

```yaml
spec:
  volumes:
  - name: pod-info
    downwardAPI:
      items:
      - path: "pod-name"
        fieldRef:
          fieldPath: metadata.name
      - path: "pod-namespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "pod-ip"
        fieldRef:
          fieldPath: status.podIP
      - path: "cpu-limit"
        resourceFieldRef:
          containerName: app
          resource: limits.cpu
      - path: "memory-request"
        resourceFieldRef:
          containerName: app
          resource: requests.memory
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels   # all labels as key=value lines

  containers:
  - name: app
    volumeMounts:
    - name: pod-info
      mountPath: /etc/podinfo
```

Inside the container:
```bash
cat /etc/podinfo/pod-name
# web-pod-abc123

cat /etc/podinfo/pod-ip
# 10.0.1.45

cat /etc/podinfo/labels
# app="web-server"
# environment="production"
# version="2.1.0"
```

Applications use this to include Pod metadata in their logs and metrics without talking to the Kubernetes API.

---

## Volume Mount Options

Several options control how a volume is mounted inside a container:

```yaml
volumeMounts:
- name: data-volume
  mountPath: /data           # where to mount in the container
  readOnly: false            # false = read-write (default); true = read-only

- name: config-volume
  mountPath: /etc/app/config.json
  subPath: config.json       # mount only this key as a single file
  readOnly: true

- name: shared-volume
  mountPath: /app/data
  mountPropagation: HostToContainer  # propagate mounts from host into container
  # None = default, no propagation
  # HostToContainer = host mounts visible in container
  # Bidirectional = container can make mounts visible to host (requires privileged)
```

`mountPropagation` is an advanced feature used by storage drivers (CSI node components) that need to see mounts made by the host.

---

## Verifying Volume Mounts

```bash
# See volumes defined for a Pod
kubectl describe pod web-pod -n production | grep -A 20 "Volumes:"

# Volumes:
#   shared-data:
#     Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
#     Medium:     Memory
#   app-config:
#     Type:      ConfigMap (a volume populated by a ConfigMap)
#     Name:      app-config-cm

# See mounted paths inside the running container
kubectl exec web-pod -n production -- df -h
# /dev/xvda1      20G   2G  18G  10%  /          (root filesystem)
# tmpfs           512M  0   512M  0%  /var/shared  (emptyDir Memory)
# /dev/xvdbf      20G   1G  19G   5%  /data        (EBS PVC)

# List files in a mounted volume
kubectl exec web-pod -n production -- ls -la /data/
```

---

## Summary

- Volumes bridge the Pod spec to a storage source — defined in `spec.volumes` and mounted in `spec.containers[].volumeMounts`
- Multiple containers in the same Pod can share the same volume — writes by one container are immediately visible to others
- Volume lifetime depends on type: `emptyDir` lives as long as the Pod; `hostPath` as long as the node; PVC-backed volumes live independently
- `projected` volumes combine multiple sources (secrets, configmaps, SA tokens, downwardAPI) into a single mount directory
- `downwardAPI` volumes expose Pod metadata (name, IP, labels, resource limits) as files — useful for logging and monitoring without API calls
- `readOnly: true` and `subPath` are the two most common mount options — read-only for security, subPath to mount a single file without replacing the directory

In the next file, `03-volume-types.md`, you will get a comprehensive reference of the most important volume types with full YAML examples and guidance on when to use each.
