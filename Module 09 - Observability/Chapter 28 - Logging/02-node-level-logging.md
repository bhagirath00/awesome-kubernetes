# Node-Level Logging

## How Kubelet Captures Logs

Every node runs kubelet. Kubelet's job includes capturing container stdout/stderr.

```
Container stdout/stderr
        ↓
    Docker daemon
        ↓
    /var/lib/docker/containers/
        (container_id.log)
        ↓
    kubelet reads file
        ↓
    /var/log/containers/
        (pod_namespace_containerid.log)
        ↓
kubectl logs can read this
```

---

## Node Log Directory Structure

```bash
# SSH into node and check logs
ssh ec2-user@node-ip

# View container logs on node
ls -la /var/log/containers/

# Example:
# app-pod_default_container-12ab34@sha256:xxxxx.log
# app-pod_default_container-12ab34@sha256:xxxxx.log.1
# app-pod_default_container-12ab34@sha256:xxxxx.log.2
```

Each line is one log entry:

```json
{
  "log": "INFO: User logged in\n",
  "stream": "stdout",
  "time": "2024-01-15T10:30:45.123456789Z"
}
```

---

## Log Rotation

Logs are rotated to prevent disk full:

```bash
# Check kubelet config
cat /etc/kubernetes/kubelet/kubeconfig

# Or check kubelet arguments
ps aux | grep kubelet | grep log
```

Default rotation:
```
Max file size: 100MB
Max files kept: 5 rotations
```

When max size reached:
```
container.log     → container.log.1
container.log.1   → container.log.2
container.log.2   → container.log.3
container.log.3   → container.log.4
container.log.4   → deleted
```

---

## Viewing Node Logs

```bash
# SSH to node
ssh ec2-user@10.0.1.100

# Find pod logs
find /var/log/containers -name "*app-pod*"

# View raw logs
cat /var/log/containers/app-pod_default_*.log | tail -100

# Parse JSON logs
cat /var/log/containers/app-pod_default_*.log | \
  jq '.log' | head -50
```

---

## Log Disk Usage

Node storage can fill up with logs:

```bash
# Check disk usage
df -h /var/log/

# Size of container logs
du -sh /var/log/containers/

# Too many logs? Manually cleanup
rm /var/log/containers/*.log.* # Keep only current
```

**Common problem:**
```
Node disk 90% full
New logs can't write
Pods behave strangely
Investigate → "Not enough space"
```

---

## Real-World Scenario: Debugging a Crash

```
Pod crashes repeatedly
You need to understand why

Step 1: Pod is gone
  kubectl logs crashed-pod
  → "not found" (pod already deleted)

Step 2: Check node logs (quickly, before rotation)
  ssh node-ip
  cat /var/log/containers/crashed-pod_*.log

Step 3: View last lines before crash
  tail -100 /var/log/containers/crashed-pod_*.log
  
Result: See stack trace, understand cause
```

---

## Multi-Container Pods

Pod with multiple containers, each has separate logs:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: web
    image: nginx
  - name: sidecar
    image: filebeat
```

Node logs:

```
/var/log/containers/web-app_default_web-xxxxx.log
/var/log/containers/web-app_default_sidecar-xxxxx.log
```

View specific container:

```bash
kubectl logs web-app -c web      # Web logs
kubectl logs web-app -c sidecar  # Sidecar logs
```

---

## Common Node Logging Issues

### Issue 1: Disk Full

```bash
# Check available space
df -h /var/log/

# Solution: Increase node size OR use external logging
# Temporary: Clean old logs
docker logs $(docker ps -a -q) > /dev/null
```

### Issue 2: Logs Lost After Rotation

```
Logs older than rotation window are gone
Can't investigate issues older than 24 hours

Solution: Centralized logging (Module 9 next chapters)
```

### Issue 3: Can't Find Pod Logs

```bash
# Pod name and namespace
kubectl get pod -n default

# Log file name includes pod name and namespace
ls /var/log/containers/ | grep "pod-name_default"
```

---

## Best Practices

1. **Log Rotation Configuration**
   - Ensure kubelet has enough space
   - Monitor /var/log/ disk usage

2. **Container Log Best Practices**
   - Use `maxSize` and `maxFile` in Docker config

3. **Debug Quickly**
   - SSH to node immediately after crash
   - Logs deleted after rotation

4. **Plan for Centralized Logging**
   - Node logs are temporary
   - For long-term retention, use external system

---

## Summary

Node-level logging:

**How it works:**
- kubelet captures container stdout/stderr
- Stored in /var/log/containers/
- Accessible via kubectl logs

**Rotation:**
- Default: 100MB max file, 5 rotations
- Old logs deleted automatically

**Limitations:**
- Temporary storage only
- Deleted after rotation
- Disk space can fill up

**Next:** File 03 covers cluster-level centralized logging.
