# Falco Overview: eBPF-Powered Kubernetes Runtime Security

## 1. Architectural Concept & Deep Dive

Static security tools (vulnerability scanners, image linters, admission controllers) provide **preventative security**: they inspect code and manifests before deployment.

However, once a pod is running in production, static tools are completely blind to live adversarial activity:
- An attacker exploits a zero-day remote code execution (RCE) vulnerability in a web framework.
- An attacker spawns an interactive shell (`/bin/sh`) inside an Nginx container.
- An attacker reads `/etc/shadow` or attempts to download a crypto-miner binary via `curl`.
- An attacker mounts a host filesystem escape vector.

**`Falco`** is the CNCF graduated, de-facto standard **runtime security detection engine** for Linux and Kubernetes. Created by Sysdig, Falco acts as a behavioral security camera running on every node, continuously monitoring kernel system calls and Kubernetes audit events to detect anomalous and malicious behavior in real time.

```
+─────────────────────────────────────────────────────────────────────────+
|                           FALCO ARCHITECTURE                            |
|                                                                         |
|  Containerized Application (Node.js / Java / Go)                        |
|         │                                                               |
|         ├── Executes: execve("/bin/sh") or openat("/etc/shadow")        |
|         ▼                                                               |
|  [ Linux Kernel (Ring 0) ]                                              |
|         │                                                               |
|         ├── Modern eBPF Probe (CO-RE: Compile Once - Run Everywhere)    |
|         │   Hooks raw tracepoints (sys_enter / sys_exit)                |
|         ▼                                                               |
|  [ Lockless Perf Ring Buffer ]                                          |
|         │                                                               |
|         ▼ (Nanosecond transfer to userspace)                            |
|  [ Falco Userspace Daemon (DaemonSet) ]                                 |
|         │                                                               |
|         ├── Enriches event with Kubernetes metadata (Namespace, Pod, IP)|
|         ├── Evaluates against declarative Rules Engine                  |
|         ▼                                                               |
|  [ Alert Channels: Syslog / Stdout / Falcosidekick ──► Slack / SIEM ]   |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Kernel Drivers: Modern eBPF vs Legacy Kernel Module

Falco intercepts system calls using one of three kernel driver drivers:

| Driver Type | Mechanism | Kernel Compatibility | Pros & Cons |
| :--- | :--- | :--- | :--- |
| **Modern eBPF (`libbpf`)** (Recommended) | Ring buffer + CO-RE (Compile Once - Run Everywhere). | Linux 5.8+ (Ubuntu 22.04+, RHEL 9+, modern EKS AMIs) | **Gold Standard**. Safest, lowest overhead, requires zero host kernel headers or compilers. |
| **Legacy eBPF Probe** | Pre-compiled BPF bytecode loaded into kernel. | Linux 4.14+ | Requires downloading matching kernel probe artifacts or compiling locally. |
| **Kernel Module (`kmod`)**| Dynamic kernel module (`falco.ko`). | Any Linux kernel | High performance, but can cause kernel panic if driver crashes; blocked in hardened cloud environments. |

---

## 3. How Falco Detects Attacks: The System Call Stream

Every single user action on Linux generates system calls. Falco parses the stream and correlates it with container and Kubernetes context:

```
Syscall Stream:
1. clone()                   -> Process creates a new thread group
2. execve("/bin/bash")       -> Process overwrites memory space with a shell
3. read("/etc/shadow")       -> Process attempts to read password hashes
4. connect("198.51.100.2")   -> Process opens TCP socket to external C2 server
```

When Falco detects this pattern inside a container whose normal profile is a static web server:
- It enriches the event: `pod=web-frontend-79b8d namespace=production container=nginx image=nginx:alpine`.
- It triggers a **`Critical`** alert within milliseconds.

---

## 4. Top Runtime Threats Caught by Falco

1. **Terminal Shell Opened Inside a Container**:
   - Legitimate microservices run entrypoints (`node server.js`); they do not invoke interactive bash shells.
2. **Access to Sensitive Host Files**:
   - Accessing `/etc/shadow`, `/etc/sudoers`, or reading the host `/proc` file tree.
3. **Container Escape & Privilege Escalation**:
   - Unsharing namespaces or modifying kernel modules.
4. **Outbound Network Connection from Ephemeral Binaries**:
   - Downloading and executing payloads from `/tmp` or `/dev/shm`.
5. **Tampering with Kubernetes ServiceAccount Tokens**:
   - Dumping `/var/run/secrets/kubernetes.io/serviceaccount/token` to query the API server.

---

## 5. Falco Ecosystem: Falcosidekick & Automated Response

By default, Falco outputs alerts to stdout and syslog. In enterprise production, Falco is paired with **Falcosidekick**:

```
+─────────────+       +──────────────────+       +──────────────────────────+
| Falco Engine| ──►   |  Falcosidekick   | ──►   | Slack / PagerDuty        |
| (DaemonSet) | gRPC  | (Fanout Router)  |       | Datadog / Splunk / Loki  |
+─────────────+       +─────────┬────────+       +──────────────────────────+
                                │
                                ▼
                      +──────────────────+
                      | Falcosidekick-UI | (Real-time Web Dashboard)
                      +──────────────────+
```

### Automated Remediation (Response Engine)
Falcosidekick can trigger serverless functions (AWS Lambda, Kubeless, or Tekton) to **automatically isolate or terminate compromised pods**:
```
Falco detects reverse shell ──► Falcosidekick ──► Calls API Server ──► Deletes Pod instantly!
```

---

## 6. Key Takeaways
- Falco delivers real-time runtime threat detection, closing the visibility gap that static scanners cannot address.
- Uses modern eBPF CO-RE to safely inspect system calls with minimal kernel overhead (< 1-2% CPU).
- Correlates raw Linux syscalls (`execve`, `openat`, `connect`) with Kubernetes metadata (Pods, Namespaces, Images).
- Integrates with Falcosidekick to fan out alerts to SIEMs and trigger automated incident response.
