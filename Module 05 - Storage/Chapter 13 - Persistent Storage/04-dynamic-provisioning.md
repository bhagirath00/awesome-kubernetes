# Dynamic Provisioning

## Introduction

Dynamic provisioning is the process by which Kubernetes automatically creates PersistentVolumes (and the underlying AWS storage) when a PVC is created. Before dynamic provisioning existed, cluster administrators had to manually provision every storage volume and pre-create every PV. Dynamic provisioning eliminates this entirely — developers request storage via PVCs, and it appears automatically.

On EKS, dynamic provisioning is handled by the **AWS EBS CSI driver** for block storage and the **AWS EFS CSI driver** for file storage.

---

## How Dynamic Provisioning Works — End to End

```
Step 1: Developer creates a PVC

Step 2: PVC controller watches for new PVCs
        → reads the storageClassName: gp3
        → finds the 'gp3' StorageClass
        → identifies the provisioner: ebs.csi.aws.com

Step 3: EBS CSI controller receives provisioning request
        → makes AWS API call: ec2:CreateVolume
        → parameters: 100Gi, gp3, us-east-1a, encrypted

Step 4: AWS creates the EBS volume
        → vol-0abc123def456789 created in us-east-1a

Step 5: EBS CSI driver creates a PersistentVolume in Kubernetes
        → PV represents the new EBS volume
        → PV status: Available

Step 6: PVC controller binds the PVC to the new PV
        → PVC status: Bound
        → PV status: Bound

Step 7: Pod is scheduled (if using WaitForFirstConsumer, happens before step 3)
        → Pod starts, volume is attached and mounted
```

The developer only creates the PVC. Steps 2–7 happen automatically.

---

## The Provisioning Request

```yaml
# Developer creates only this:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: production
spec:
  storageClassName: gp3
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

```bash
kubectl apply -f pvc.yaml

# Watch dynamic provisioning happen in real time
kubectl get pvc app-data-pvc -n production -w

# NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# app-data-pvc   Pending                                      gp3            1s
# app-data-pvc   Pending                                      gp3            3s
# app-data-pvc   Bound    pvc-xyz  50Gi        RWO            gp3            8s
# ← Bound in ~8 seconds — EBS volume created and PV bound automatically

# See the automatically created PV
kubectl get pv | grep app-data-pvc

# pvc-abc123-def456...   50Gi   RWO   Delete   Bound   production/app-data-pvc   gp3

# View the EBS volume ID
kubectl get pv pvc-abc123-def456 -o jsonpath='{.spec.csi.volumeHandle}'
# vol-0abc123def456789
```

---

## IAM Permissions Required for Dynamic Provisioning

The EBS CSI driver needs IAM permissions to create, attach, detach, and delete EBS volumes. On EKS, these are granted via IRSA:

```bash
# Install EBS CSI driver with IRSA (typical EKS setup)
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster my-cluster \
  --service-account-role-arn arn:aws:iam::123456789:role/EBSCSIDriverRole \
  --region us-east-1

# Required IAM policy actions for the EBS CSI driver role:
# ec2:CreateVolume
# ec2:DeleteVolume
# ec2:DescribeVolumes
# ec2:AttachVolume
# ec2:DetachVolume
# ec2:DescribeInstances
# ec2:CreateTags
# ec2:DescribeAvailabilityZones
```

```bash
# Verify the EBS CSI driver is running
kubectl get pods -n kube-system | grep ebs-csi

# NAME                                  READY   STATUS    RESTARTS
# ebs-csi-controller-abc12             6/6     Running   0
# ebs-csi-controller-def34             6/6     Running   0
# ebs-csi-node-4xkqp                   3/3     Running   0   (on every node)
# ebs-csi-node-7bmpv                   3/3     Running   0
```

---

## WaitForFirstConsumer — AZ-Aware Provisioning

With `volumeBindingMode: WaitForFirstConsumer`, the EBS volume is NOT created when the PVC is created. It is created only after the scheduler decides which node the Pod will run on:

```bash
# Create a PVC — stays Pending until a Pod is created
kubectl apply -f pvc.yaml

kubectl get pvc app-data-pvc -n production
# NAME           STATUS    VOLUME   CAPACITY   STORAGECLASS
# app-data-pvc   Pending                       gp3
# ← Pending — no Pod yet, no EBS volume yet

# Create a Pod that uses the PVC
kubectl apply -f pod.yaml

# Within seconds:
kubectl get pvc app-data-pvc -n production
# NAME           STATUS   VOLUME        CAPACITY   STORAGECLASS
# app-data-pvc   Bound    pvc-xyz...    50Gi       gp3
# ← Bound — EBS created in same AZ as the scheduled node
```

The EBS volume is guaranteed to be in the same AZ as the node. This eliminates the AZ mismatch problem.

---

## Volume Lifecycle During Pod Deletion and Recreation

```bash
# Delete the Pod — EBS volume is detached but NOT deleted
kubectl delete pod app-pod -n production

# EBS volume still exists, PVC still bound:
kubectl get pvc app-data-pvc -n production
# NAME           STATUS   VOLUME        CAPACITY
# app-data-pvc   Bound    pvc-xyz...    50Gi    ← still bound, data preserved

# Create a new Pod referencing the same PVC — volume reattaches
kubectl apply -f pod.yaml

kubectl get pod app-pod -n production
# NAME       READY   STATUS    RESTARTS
# app-pod    1/1     Running   0
# ← New Pod has all data from the previous Pod
```

The EBS volume reattaches to the new Pod automatically — potentially on a different node in the same AZ.

---

## EBS Volume Tags for Cost Allocation

The EBS CSI driver automatically tags EBS volumes with information about the PVC and cluster. You can add custom tags via the StorageClass:

```yaml
parameters:
  type: gp3
  tagSpecification_1: "Environment=production"
  tagSpecification_2: "Team=platform"
  tagSpecification_3: "ManagedBy=kubernetes"
```

Custom tags enable AWS Cost Explorer to break down EBS costs by environment, team, or application.

---

## Deleting a PVC — What Happens to the Data

```bash
# First delete the Pod
kubectl delete deployment postgres -n production

# Then delete the PVC (triggers reclaim policy)
kubectl delete pvc app-data-pvc -n production

# With reclaimPolicy: Delete:
kubectl get pv | grep app-data-pvc
# (no output) → PV deleted

# The EBS volume is also deleted:
aws ec2 describe-volumes \
  --volume-ids vol-0abc123def456789
# Error: InvalidVolume.NotFound  ← EBS volume deleted permanently

# With reclaimPolicy: Retain:
kubectl get pv
# pvc-abc123... Released → PV stays, EBS volume preserved
# Must manually delete the PV after archiving data
```

---

## Monitoring Dynamic Provisioning

```bash
# Watch provisioning events
kubectl get events -n production --sort-by='.lastTimestamp' | grep -i "provision"

# Normal   ProvisioningSucceeded  Successfully provisioned volume pvc-abc123 using ebs.csi.aws.com
# Warning  ProvisioningFailed     failed to provision volume with StorageClass "gp3": AccessDenied

# Check CSI driver logs for provisioning errors
kubectl logs -n kube-system -l app=ebs-csi-controller -c csi-provisioner --tail=30

# List all dynamically provisioned EBS volumes in AWS for a cluster
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/cluster/my-cluster,Values=owned" \
  --query 'Volumes[].{ID:VolumeId,Size:Size,State:State,AZ:AvailabilityZone,PVC:Tags[?Key==`kubernetes.io/created-for/pvc/name`].Value|[0]}' \
  --output table
```

---

## Summary

- Dynamic provisioning automatically creates EBS volumes when PVCs are created — no manual admin steps, no pre-created PVs
- The EBS CSI driver handles provisioning via AWS API calls — it needs IRSA permissions (`ec2:CreateVolume`, `ec2:AttachVolume`, etc.)
- `WaitForFirstConsumer` binding mode ensures the EBS volume is created in the same AZ as the Pod's node — prevents AZ mismatch
- Deleting a Pod detaches the EBS volume but does NOT delete it — PVC and volume persist, data survives
- Deleting a PVC with `Delete` reclaim policy permanently destroys the EBS volume — protect production PVCs with RBAC
- Tag EBS volumes via StorageClass parameters for AWS Cost Explorer cost allocation by team and environment

In the next file, `05-access-modes.md`, you will learn the access modes in depth — RWO, ROX, RWX, RWOP — what they mean, which AWS storage supports each, and how to choose the right mode for your workload.
