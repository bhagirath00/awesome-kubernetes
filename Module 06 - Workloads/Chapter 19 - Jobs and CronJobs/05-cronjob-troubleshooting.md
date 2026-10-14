# CronJob Troubleshooting

## Introduction

Troubleshooting CronJobs is notoriously difficult because the objects are transient. By the time you notice a failure, the Pod might have been deleted, the Job object might be gone, and the logs might have been purged. To be a senior DevOps engineer, you must master the "breadcrumb trail" left behind by the Kubernetes CronJob controller.

This guide provides a deep-dive into the diagnostics lifecycle, covering everything from simple schedule errors to complex resource starvation and `etcd` pressure issues.

---

## 1. The Diagnostic Workflow

When a CronJob fails, you must follow this exact order of operations to find the root cause:

1.  **Check the CronJob Object**: Did it even trigger?
2.  **Check the Job Object**: Was the Job created successfully?
3.  **Check the Pod Object**: Did the Pod start? Why did it exit?
4.  **Check the Container Logs**: What happened inside the code?
5.  **Check Cluster Events**: Are there resource or permission errors?

---

## 2. "Why isn't my Job starting?"

If you see that `LAST SCHEDULE` is empty or very old, the problem is at the controller level.

### A. The Schedule Syntax Error
Kubernetes cron syntax is strict. Even a single extra space can break it.
- **Action**: Use `kubectl describe cronjob <name>` and look at the `Schedule` field.
- **Common Mistake**: `0 0 1 * *` is "Midnight on the 1st of the month". If you wanted "Every minute on the 1st of January", you would be waiting a long time.
- **Fix**: Verify your string at [Crontab Guru](https://crontab.guru/).

### B. The "Too Many Missed Starts" Trap
Kubernetes has a hard limit: if a CronJob misses more than **100 consecutive start times**, the controller stops trying to schedule it entirely.
- **Cause**: This happens if the cluster was down for a long time, or if `startingDeadlineSeconds` is unset/too low. If the controller doesn't have a record of when it last ran (e.g., after a cluster upgrade), it calculates all missed runs since the creation time.
- **Symptoms**: `kubectl get cj` shows a schedule, but the `LAST SCHEDULE` field never updates.
- **Fix**: You must delete the CronJob and recreate it to reset the counter. To prevent this, always set `startingDeadlineSeconds` (e.g., `200`) so the controller only looks at recent misses.

```bash
# Check events for this specific error
kubectl get events -n <namespace> --field-selector involvedObject.kind=CronJob
# Output: "Too many missed start times (> 100). Set or increase .spec.startingDeadlineSeconds or check clock skew."
```

### C. Concurrency Policy Blocks
If your `concurrencyPolicy` is set to `Forbid`, and the previous run is "zombied" (stuck), no new runs will ever start.
- **Action**: Check if there is an active Job: `kubectl get jobs`.
- **Fix**: Find the stuck Pod and kill it manually. Then investigate why it got stuck (usually a missing timeout).

### D. The Timezone Trap
By default, Kubernetes CronJobs run in **UTC**. If your job was supposed to run at 2:00 AM local time but ran at 7:00 AM, you likely have a timezone mismatch.
- **K8s 1.27+ Solution**: You can now set the `.spec.timeZone` field (e.g., `timeZone: "America/New_York"`).
- **Older Clusters**: You must manually calculate the UTC offset (and remember to update it for Daylight Savings!).

---

## 3. "The Job exists but it failed."

If the Job was created but is marked as `Failed`, you need to look at the Pods.

### A. The `RestartPolicy` Confusion
In a CronJob, you must set `restartPolicy: OnFailure` or `Never`.
- **OnFailure**: The container restarts *inside* the same Pod. You lose the logs from the previous attempt!
- **Never**: Kubernetes creates a *new Pod* for each retry. This is much better for debugging.

```bash
# Get the failed pods (use -l to filter by job name)
kubectl get pods --selector=job-name=<job-name>

# Describe the failed pod to see the exit code
kubectl describe pod <failed-pod-name>
# Look for: "Exit Code: 137" (OOMKilled) or "Exit Code: 1" (App Error)
```

### B. Image and Pull Secret Issues
If your CronJob uses a private ECR image, the "ImagePullBackOff" is a common failure point.
- **Cause**: ECR token expired, or the ServiceAccount doesn't have the `imagePullSecrets` attached.
- **Action**: `kubectl describe pod <pod-name>` and look for "Failed to pull image".

---

## 4. Resource Starvation (OOMKilled)

This is the most common cause of silent failures in production batch jobs.

### The OOMKill Cycle
1. The Job starts processing data.
2. It hits its memory limit.
3. Linux kernel kills the process (Exit Code 137).
4. Kubernetes sees the exit code and retries (up to `backoffLimit`).
5. After several attempts, the Job is marked "Failed".

```bash
# How to detect OOMKill
kubectl get pods -n <namespace> -o json | jq '.items[].status.containerStatuses[].lastState.terminated.reason'
# Output: "OOMKilled"
```

**The Solution**: Increase the `memory` limits in the `jobTemplate`. Batch jobs often need 2x-4x more memory than web APIs because they load large datasets into memory at once.

---

## 5. Network and Permission Issues

Since CronJobs are automated, they often fail because of environment changes.

### A. Missing ConfigMaps or Secrets
If you update your application but forget to update the CronJob's `envFrom`, the Pod will never start.
- **Symptom**: Pod status is `CreateContainerConfigError`.
- **Fix**: `kubectl describe pod` will tell you exactly which Secret is missing.

### B. AWS Permission Denied
If your script talks to S3 or RDS and suddenly fails with "Access Denied":
- **Action**: Check if the IAM Role (via IRSA) was modified.
- **Action**: Ensure the ServiceAccount name in the CronJob YAML matches the one with the IAM annotation.

```bash
# Debugging AWS permissions inside the pod
kubectl exec -it <cronjob-pod> -- aws sts get-caller-identity
```

---

## 6. The "Sidecar Zombie" Problem

In modern clusters using Istio, Linkerd, or AWS App Mesh, a Job may finish its work but stay in `Running` status forever.

- **The Cause**: The sidecar container (e.g., `istio-proxy`) doesn't know the main container is finished, so it stays alive. Since the Pod is still "Running," the Job never completes.
- **The Symptom**: `kubectl get jobs` shows a Job that is 2 hours old but has 0/1 completions, even though the logs show the task finished in seconds.
- **The Fix**: 
    - **K8s 1.29+**: Use the native `SidecarContainers` feature (set `restartPolicy: Always` on the sidecar in the init container list).
    - **Workaround**: Add a `curl -X POST http://localhost:15020/quitquitquit` (for Istio) at the end of your script to kill the sidecar manually.

---

## 7. Cleanup and `etcd` Pressure

If you have 100 CronJobs running every minute, and you don't set history limits, your cluster will slow down significantly.

- **The Problem**: Every Job run is a record in the Kubernetes database (`etcd`). Millions of old Job objects will cause high memory usage on the control plane.
- **The Fix**: Always set `successfulJobsHistoryLimit` and `failedJobsHistoryLimit`.

```yaml
# A safe production setting
spec:
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
```

---

## 8. Preventing Zombies with Deadlines

To prevent a single stuck Job from blocking your entire pipeline (especially with `concurrencyPolicy: Forbid`), you should always use `activeDeadlineSeconds`.

- **activeDeadlineSeconds**: This is a timer that starts when the Pod begins. If the Job isn't finished by this time, Kubernetes forcibly kills it.
- **Example**: If your backup usually takes 10 minutes, set this to `1800` (30 mins). This ensures that even if the code hangs, the Job will be terminated, allowing the next scheduled run to start.

---

## 9. Advanced Debugging: The Manual Run

Sometimes you can't wait 24 hours for the next schedule to test a fix. You should manually trigger a Job from the CronJob definition.

```bash
# Manually trigger a Job from a CronJob
kubectl create job --from=cronjob/my-cronjob test-run-001 -n production

# This creates a "one-off" Job that is identical to the scheduled one.
# You can now watch it, log it, and describe it in real-time.
kubectl logs -f test-run-001-abcde
```

---

## Troubleshooting Summary Table

| Symptom | Probable Cause | Diagnostic Command |
| :--- | :--- | :--- |
| **No Job created** | Schedule syntax error or `suspend: true`. | `kubectl describe cj <name>` |
| **Job stays "Active"** | Code is stuck in infinite loop or waiting on network. | `kubectl top pod <pod-name>` |
| **Job never completes** | Sidecar zombie (Istio/AppMesh). | `kubectl describe pod` |
| **Runs at wrong time** | Timezone mismatch (UTC vs Local). | `kubectl describe cj` |
| **Exit Code 137** | OOMKilled — insufficient memory. | `kubectl describe pod` |
| **Exit Code 1** | Application-level logic error. | `kubectl logs <pod-name>` |
| **Pending Status** | No nodes have enough CPU/Memory. | `kubectl describe pod` |
| **ImagePullBackOff** | Credentials or image tag error. | `kubectl get events` |

---

## Summary

- **Transient nature**: You must act fast or use high history limits to debug CronJobs before they are garbage-collected.
- **Check the events**: Most "why didn't it start" questions are answered in the CronJob events.
- **Exit Code 137 is OOMKill**: Always monitor your memory usage. Batch jobs are memory-hungry.
- **Never wait for the schedule**: Use `kubectl create job --from=cronjob` to test your fixes immediately.
- **Manage History**: Prevent `etcd` bloat by keeping only the last few runs.
- **Termination and Timeouts**: Always use `activeDeadlineSeconds` so a single stuck job doesn't block your entire pipeline indefinitely.

This completes Module 6 — Workloads. You have now learned how to run every type of application on Kubernetes: from persistent **StatefulSets** to ephemeral **Jobs** and automated **CronJobs**. 

In the next Module, we will move on to **Networking**, where we will explore how these workloads communicate with each other and the outside world using Services and Ingress.
