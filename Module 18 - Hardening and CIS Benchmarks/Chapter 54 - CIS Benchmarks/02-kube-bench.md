# Automated CIS Scanning with kube-bench

## 1. Architectural Concept & Deep Dive

Manually auditing hundreds of CIS benchmark controls across dozens of Kubernetes master and worker nodes is error-prone and unsustainable.

**`kube-bench`** is an open-source Go tool developed by Aqua Security that automates the execution of the CIS Kubernetes Benchmark. It inspects running processes, configuration files, and systemd parameters on the host, matching them against declarative YAML test definitions that mirror the official CIS specifications.

```
+─────────────────────────────────────────────────────────────────────────+
|                         KUBE-BENCH ARCHITECTURE                         |
|                                                                         |
|   +───────────────────────────────────+                                 |
|   |         Target Environment        |                                 |
|   |  • /etc/kubernetes/manifests      |                                 |
|   |  • /var/lib/kubelet/config.yaml   |                                 |
|   |  • Running Process CLI Flags      |                                 |
|   +─────────────────┬─────────────────+                                 |
|                     │                                                   |
|                     ▼ (Mounted host volumes)                            |
|   +───────────────────────────────────+                                 |
|   |          kube-bench Job           |                                 |
|   |   Reads cfg/cis-1.24/master.yaml  |                                 |
|   |   Executes checks & regex tests   |                                 |
|   +─────────────────┬─────────────────+                                 |
|                     │                                                   |
|                     ▼                                                   |
|   [ Report Output: PASS / FAIL / WARN / INFO with Remediation Steps ]   |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Running kube-bench in Kubernetes as a Job

The standard method to scan a live cluster is deploying `kube-bench` as a Kubernetes `Job` or `DaemonSet`. Because `kube-bench` must inspect host files and processes, it mounts host directories (`/etc/kubernetes`, `/var/lib/kubelet`, `/etc/systemd`).

### Job Manifest: Control Plane Scan (`kube-bench-master.yaml`)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench-master
  namespace: kube-system
spec:
  template:
    spec:
      hostPID: true
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      restartPolicy: Never
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:v0.7.3
          command: ["kube-bench", "run", "--targets", "master,controlplane,etcd,policies"]
          volumeMounts:
            - name: var-usr-bin
              mountPath: /usr/local/mount-from-host/bin
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
            - name: etc-systemd
              mountPath: /etc/systemd
              readOnly: true
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
      volumes:
        - name: var-usr-bin
          hostPath:
            path: /bin
        - name: etc-kubernetes
          hostPath:
            path: /etc/kubernetes
        - name: etc-systemd
          hostPath:
            path: /etc/systemd
        - name: var-lib-kubelet
          hostPath:
            path: /var/lib/kubelet
```

Run the scan:
```bash
kubectl apply -f kube-bench-master.yaml
```

View the generated report:
```bash
kubectl logs -n kube-system job/kube-bench-master
```

---

## 3. Dissecting the kube-bench Output Report

The output organizes findings by section with clear status codes:

```
[INFO] 1 Control Plane Security Configuration
[INFO] 1.1 Control Plane Node Configuration Files
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 600 or more restrictive (Automated)
[PASS] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root (Automated)
[FAIL] 1.1.12 Ensure that the etcd data directory permissions are set to 700 or more restrictive (Automated)
[WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false (Automated)

== Remediations master ==
1.1.12 On the etcd server node, get the etcd data directory, passed as the --data-dir parameter
to the etcd daemon. Run the below command (based on the etcd data directory found above):
chmod 700 /var/lib/etcd

1.2.1 Edit the API server pod specification file /etc/kubernetes/manifests/kube-apiserver.yaml
on the control plane node and set the below parameter:
--anonymous-auth=false

== Summary master ==
45 checks PASS
1 checks FAIL
2 checks WARN
0 checks INFO
```

---

## 4. Running kube-bench in Managed Clouds (AWS EKS & GCP GKE)

For AWS EKS worker nodes, `kube-bench` provides tailored benchmark targets:

```bash
# Run standalone docker container on an EKS EC2 worker node
docker run --rm -v /etc/systemd:/etc/systemd:ro \
  -v /var/lib/kubelet:/var/lib/kubelet:ro \
  -v /usr/bin:/usr/bin:ro \
  aquasec/kube-bench:latest \
  run --targets "node" --version "eks-1.0"
```

---

## 5. Integrating kube-bench into CI/CD & Security Pipelines

To feed scan results into SIEM systems (Splunk, Datadog) or fail automated pull requests in CI pipelines:

```bash
# Output results in machine-readable JSON format
kube-bench run --output-style json --output-filename /tmp/report.json

# Check if any FAIL status exists using jq
FAIL_COUNT=$(jq '.Totals.total_fail' /tmp/report.json)

if [ "$FAIL_COUNT" -gt 0 ]; then
  echo "Security Gate Failed: $FAIL_COUNT CIS Benchmark violations found!"
  exit 1
fi
```

---

## 6. Key Takeaways
- `kube-bench` is the standard open-source tool for automating CIS Kubernetes Benchmark compliance audits.
- It can run as a Kubernetes Job, DaemonSet, standalone binary, or Docker container.
- Reports classify checks into `[PASS]`, `[FAIL]`, and `[WARN]`, providing exact command-line remediation steps for every failure.
- Export results as JSON to enforce automated compliance gates in CI/CD pipelines.
