# Graceful Shutdown

## The Problem

Your pod is receiving 1000 requests per second. Suddenly, Kubernetes sends SIGTERM (shutdown signal). Your app:

```
Option A: Immediately stops → Requests in flight lost → Customers angry
Option B: Waits for requests to finish → Takes 30 seconds → Clean exit
```

Graceful shutdown allows Option B.

---

## Termination Grace Period

Default timeout before Kubernetes force-kills a pod:

```
1. Kubernetes sends SIGTERM signal
   ↓ (app has time to respond)
2. App gracefully shuts down (stops accepting requests, finishes in-flight)
   ↓ (grace period elapses)
3. Kubernetes sends SIGKILL (force kill)
   └─ Pod dies immediately if grace period exceeded
```

---

## Grace Period Configuration

### Basic Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  terminationGracePeriodSeconds: 30  # Default is 30 seconds
  containers:
  - name: app
    image: my-app:latest
```

The pod gets 30 seconds to shut down gracefully.

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60  # 60 seconds for graceful shutdown
      containers:
      - name: app
        image: my-app:latest
```

---

## How Apps Handle Graceful Shutdown

### Step 1: App Receives SIGTERM

```python
# Python example
import signal
import time

def signal_handler(sig, frame):
    print("Received SIGTERM, starting graceful shutdown...")
    # Stop accepting new requests
    # Finish in-flight requests
    # Close database connections
    # Save state if needed
    time.sleep(5)  # Wait for in-flight requests
    print("Shutdown complete")
    exit(0)

signal.signal(signal.SIGTERM, signal_handler)

# Main app loop
while True:
    # Serve requests
    pass
```

### Step 2: App Cleans Up

```
1. Stop accepting new requests
2. Wait for in-flight requests to complete
3. Close database/cache connections
4. Flush logs
5. Exit cleanly (exit 0)
```

### Step 3: Kubernetes Checks Exit

```
App exits with code 0 → Clean exit ✓
App still running after grace period → SIGKILL ❌
```

---

## PreStop Hook

Execute custom cleanup before pod termination:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  terminationGracePeriodSeconds: 45
  containers:
  - name: app
    image: my-app:latest
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15 && /opt/app/cleanup.sh"]
```

Execution sequence:

```
1. Kubernetes sends SIGTERM
2. preStop hook runs: sleep 15 && /opt/app/cleanup.sh
3. App handles SIGTERM signal
4. Wait until 45 seconds elapsed or app exits
5. SIGKILL if still running
```

---

## Real-World Example: Web Server

### Without Graceful Shutdown

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        # Default: terminationGracePeriodSeconds: 30
```

During drain:

```
Time 0s: SIGTERM sent
Time 0-3s: Requests in flight get disconnected
Time 5s: Pod dies
Result: Users see connection errors ❌
```

### With Graceful Shutdown

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  template:
    spec:
      terminationGracePeriodSeconds: 45
      containers:
      - name: nginx
        image: nginx:latest
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15 && nginx -s quit; while killall -0 nginx 2>/dev/null; do sleep 1; done"]
```

During drain:

```
Time 0s: SIGTERM sent
Time 0s: preStop runs: sleep 15
  └─ Gives 15 seconds for load balancer to remove this pod from rotation
Time 15s: preStop finishes
Time 15-45s: App finishes in-flight requests
Time 45s: Pod dies if still running
Result: No connection errors ✓
```

---

## Readiness vs Termination

```
Readiness probe: Can this pod handle requests?
├─ Checks during normal operation
├─ Removes unhealthy pods from load balancer
└─ Run frequently (every 10s)

Termination: How long to wait for shutdown?
├─ Checked during pod termination
├─ Gives app time to finish requests
└─ Runs once at end of pod life
```

---

## Database Connection Graceful Shutdown

### Scenario: Database Pod Terminating

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  terminationGracePeriodSeconds: 120  # 2 minutes for graceful DB shutdown
  containers:
  - name: postgres
    image: postgres:14
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - |
            # Postgres graceful shutdown
            # Close connections
            # Wait for ongoing transactions
            pg_ctl stop -m fast
```

Database needs long grace period for:
- Flushing data to disk
- Closing connections
- Replaying WAL logs

---

## Job Termination

### Scenario: Long-Running Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 300  # 5 minutes
      containers:
      - name: backup
        image: backup-tool:latest
        command: ["/opt/backup.sh"]
      restartPolicy: Never
```

Why long grace period?

- Backup in progress
- Finalizing archive
- Uploading to S3
- 5 minutes prevents data corruption

---

## Load Balancer Draining

Use preStop to give load balancers time:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: api
        image: web-api:latest
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - |
                # Give ALB/NLB time to drain connections
                # They check for terminating=true and remove from LB
                sleep 15
                
                # Then gracefully shutdown app
                # Send SIGTERM to app process
                kill -TERM 1
```

AWS ALB/NLB behavior:

```
1. Pod marked for termination
2. ALB detects terminating=true
3. ALB waits for connection draining (configurable timeout)
4. New requests no longer sent to this pod
5. Existing connections allowed to complete
6. Pod preStop sleep gives ALB time to detect and react
```

---

## Checking Grace Period

```bash
# See grace period in deployment
kubectl get deployment my-app -o yaml | grep -A 5 terminationGracePeriod

# Check running pod
kubectl get pod my-app-pod-xyz -o yaml | grep terminationGracePeriod

# Check events during termination
kubectl describe pod terminating-pod

# Shows:
# Events:
#   Type    Reason     Age    Message
#   ----    ------     ---    -------
#   Normal  Killing    10s    Stopping container app
```

---

## Common Issues

### Issue 1: Grace Period Too Short

```bash
# Pod terminated before cleanup finished
# Error: Terminating after 30s but job still running

# Solution: Increase grace period
terminationGracePeriodSeconds: 120
```

### Issue 2: App Ignores SIGTERM

```bash
# App doesn't handle SIGTERM signal
# App force-killed, data corrupted

# Solution: App must handle SIGTERM
# Implement signal handler in your app
```

### Issue 3: preStop Too Long

```yaml
# preStop waits 5 minutes
lifecycle:
  preStop:
    exec:
      command: ["sleep", "300"]

# If pod has multiple on same node, waiting stalls drain
# Solution: Keep preStop reasonable (15-30s max)
```

---

## Best Practices

1. **Always Set Grace Period**
   ```yaml
   terminationGracePeriodSeconds: 30  # At minimum
   ```

2. **App Should Handle SIGTERM**
   - Implement signal handlers
   - Stop accepting requests
   - Finish in-flight requests
   - Exit cleanly

3. **Use preStop Wisely**
   - Give load balancer time to detect termination
   - Keep it under 30 seconds
   - Then signal app to shutdown

4. **Test Termination**
   ```bash
   # Delete pod to test graceful shutdown
   kubectl delete pod my-app-xyz --grace-period=30
   
   # Should see app log "shutting down..."
   # Then pod terminates cleanly
   ```

5. **Adjust Per App**
   - Web servers: 30-60 seconds
   - Databases: 120-300 seconds
   - Jobs: 5-10 minutes

---

## Summary

Graceful shutdown:

**Components:**
- terminationGracePeriodSeconds (timeout)
- preStop hook (cleanup before SIGTERM)
- App signal handler (responds to SIGTERM)

**Process:**
- SIGTERM sent
- preStop executes (optional)
- App exits cleanly
- SIGKILL after grace period if needed

**Duration:**
- Web apps: 30-60 seconds
- Databases: 2-5 minutes
- Jobs: 5-10 minutes

**Best Practice:**
- Implement graceful shutdown in app code
- Use preStop for load balancer draining
- Set appropriate grace period per app type

**Next:** File 04 covers complete maintenance workflow.
