# Hardening the Kubelet: Worker Node Agent Security

## 1. Architectural Concept & Deep Dive

The **Kubelet** is the primary node agent running as a systemd daemon on every worker node in a Kubernetes cluster. It watches for PodSpecs assigned by the scheduler, interacts with the Container Runtime Interface (CRI: containerd / CRI-O) to launch containers, monitors health, and manages volume mounts.

The Kubelet exposes an HTTPS API on port **10250**. Historically, many Kubernetes installations left this port unauthenticated or set to `AlwaysAllow`. 

### The Tesla Kubernetes Breach (Real-World Case Study)
In 2018, attackers discovered an unauthenticated Kubernetes dashboard and exposed Kubelet port 10250 in Tesla's cloud environment. Attackers invoked Kubelet's `/run/` and `/exec/` endpoints directly, executing code inside running containers without touching the API server, stealing AWS credentials, and deploying cryptocurrency miners across their fleet.

```
+─────────────────────────────────────────────────────────────────────────+
|                        KUBELET PORT 10250 HARDENING                     |
|                                                                         |
|   Inbound Connection (Port 10250)                                       |
|          │                                                              |
|          ▼                                                              |
|   1. ANONYMOUS ACCESS CHECK                                             |
|      • anonymous.enabled: false ──► Reject unauthenticated requests     |
|          │                                                              |
|          ▼                                                              |
|   2. CLIENT CERTIFICATE VERIFICATION                                    |
|      • Verify caller has valid cert signed by Kubernetes Cluster CA     |
|          │                                                              |
|          ▼                                                              |
|   3. WEBHOOK AUTHORIZATION (mode: Webhook)                              |
|      • Kubelet calls kube-apiserver: SubjectAccessReview                |
|      • Verifies caller has RBAC permissions to execute command          |
|          │                                                              |
|          ▼                                                              |
|   [ Authorized: Allow Exec / Logs / Metrics ]                           |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Hardening `/var/lib/kubelet/config.yaml`

Modern Kubernetes clusters manage Kubelet configuration via the declarative YAML file at `/var/lib/kubelet/config.yaml`.

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# 1. Disable Anonymous Access (CIS 4.2.1)
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
    cacheTTL: "2m"
  x509:
    clientCAFile: "/etc/kubernetes/pki/ca.crt"

# 2. Enforce Webhook Authorization (CIS 4.2.2)
# NEVER leave this as AlwaysAllow!
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: "5m"
    cacheUnauthorizedTTL: "30s"

# 3. Disable the Insecure Read-Only Port 10255 (CIS 4.2.4)
readOnlyPort: 0

# 4. Protect Host Kernel Tunables (CIS 4.2.6)
# Ensures kubelet refuses to start if kernel parameters conflict with security
protectKernelDefaults: true

# 5. Restrict IPTables Masquerading & Streaming
makeIPTablesParseable: true
streamingConnectionIdleTimeout: "4h"

# 6. Enable Automatic Certificate Rotation (CIS 4.2.11)
rotateCertificates: true
serverTLSBootstrap: true
```

---

## 3. Dissecting the Hardening Directives

### 1. `authorization.mode: Webhook`
By default, some legacy distributions set Kubelet authorization to `AlwaysAllow`. 
When `authorization.mode: Webhook` is configured:
- When someone runs `kubectl exec -it <POD> -- sh`, the API server connects to the Kubelet's port 10250.
- The Kubelet issues a **`SubjectAccessReview`** back to the API server to confirm the client has explicit RBAC permission to exec into pods on this specific node.
- Direct attacker connections to port 10250 fail immediately.

### 2. `readOnlyPort: 0`
Historically, Kubelet opened an unauthenticated, plaintext HTTP port on **10255** that exposed cluster architecture, pod specs, and performance metrics. Setting `readOnlyPort: 0` disables this port completely.

### 3. `protectKernelDefaults: true`
Forces the Kubelet to validate that critical Linux kernel sysctls (such as `vm.overcommit_memory`, `kernel.panic`, and `kernel.panic_on_oops`) match expected production values rather than silently overriding them at runtime.

---

## 4. Securing Kubelet Filesystem Permissions

In accordance with CIS Benchmark Section 4.1:

```bash
# 1. Protect kubelet service file
sudo chmod 600 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
sudo chown root:root /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

# 2. Protect kubelet configuration file
sudo chmod 600 /var/lib/kubelet/config.yaml
sudo chown root:root /var/lib/kubelet/config.yaml

# 3. Protect client/server PKI certificates
sudo chmod 700 /var/lib/kubelet/pki
sudo chmod 600 /var/lib/kubelet/pki/*.key
```

---

## 5. Verification & Troubleshooting Runbook

### Testing Kubelet Port Security
Test whether port 10250 allows anonymous access:

```bash
# Attempt to query Kubelet pods anonymously
curl -k -s https://<NODE_IP>:10250/pods
```
Expected output:
```
Unauthorized
```
*(If the command returns a JSON array of all running pods, anonymous authentication is enabled and the node is vulnerable!)*

Test whether the read-only port is closed:
```bash
curl -s http://<NODE_IP>:10255/pods
```
Expected output:
```
curl: (7) Failed to connect to <NODE_IP> port 10255: Connection refused
```

### Restarting Kubelet After Modification
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
sudo systemctl status kubelet
```

---

## 6. Key Takeaways
- The Kubelet API on port 10250 must be strictly protected against unauthenticated execution (`anonymous.enabled: false`).
- Always set `authorization.mode: Webhook` so all requests require API server RBAC validation.
- Disable the legacy unauthenticated read-only port (`readOnlyPort: 0`).
- Enforce strict `0600` permissions on `/var/lib/kubelet/config.yaml` and PKI certificate directories.
