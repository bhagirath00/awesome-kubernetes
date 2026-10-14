# Writing Custom Falco Rules & Threat Detection

## 1. Architectural Concept & Deep Dive

Falco ships with hundreds of pre-configured, community-maintained rules (`falco_rules.yaml`) detecting general malicious behavior. However, enterprise environments have unique baselines, internal tooling, and CI/CD agents that trigger false positives unless rules are customized.

Falco provides a declarative, expressive domain-specific language (DSL) to author custom detection rules. Rule definitions are organized into three hierarchical constructs:
1. **Lists**: Named collections of items (e.g. approved shells, administrative users).
2. **Macros**: Reusable conditional expressions (e.g. `container and not spawned_by_ci`).
3. **Rules**: The executable detection logic binding conditions, priority levels, output templates, and MITRE ATT&CK tags.

```
+─────────────────────────────────────────────────────────────────────────+
|                        FALCO RULE EVALUATION ENGINE                     |
|                                                                         |
|   Inbound Syscall Event (e.g. openat, execve, connect)                  |
|          │                                                              |
|          ▼                                                              |
|   [ Lists & Macros ]                                                    |
|   • allowed_binaries = (nginx, redis-server, postgres)                  |
|   • is_container = (container.id != "host")                             |
|          │                                                              |
|          ▼                                                              |
|   [ Rule Condition ]                                                    |
|   • is_container and not proc.name in (allowed_binaries)                |
|          │                                                              |
|          ├── FALSE ──► Ignore Event (Zero overhead)                     |
|          └── TRUE                                                       |
|              ▼                                                          |
|   [ Output Formatting & Interpolation ]                                 |
|   "Unauthorized binary (%proc.name) executed in container %container.id"|
|              │                                                          |
|              ▼                                                          |
|   [ Emit Alert: Priority: WARNING / CRITICAL ]                          |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Anatomy of a Falco Rule

A standard Falco rule defines six key fields:

```yaml
- rule: Rule Name Identifier
  desc: Human-readable explanation of the threat
  condition: <Boolean logic using event fields, macros, and lists>
  output: <Formatted string with %field variable interpolations>
  priority: <EMERGENCY | ALERT | CRITICAL | ERROR | WARNING | NOTICE | INFO | DEBUG>
  tags: [container, mitre_tactic, team]
```

### Essential Syscall Filtering Fields

| Field Name | Type | Description |
| :--- | :--- | :--- |
| `evt.type` | String | The system call name (e.g. `execve`, `openat`, `connect`, `accept`). |
| `proc.name` | String | Binary name executing the action (e.g. `bash`, `curl`, `nc`). |
| `proc.cmdline` | String | Full command-line arguments (e.g. `curl -s http://attacker.com`). |
| `fd.name` | String | Target file path or network socket destination. |
| `container.id` | String | Container ID (`"host"` if running directly on bare metal). |
| `k8s.pod.name` | String | The originating Kubernetes Pod name. |
| `k8s.ns.name` | String | The originating Kubernetes Namespace. |

---

## 3. Mission-Critical Custom Rules

Custom rules are added to `/etc/falco/falco_rules.local.yaml` so they survive chart upgrades.

### Rule 1: Detect Package Manager Execution in Containers
Production containers should be immutable. Running `apt-get`, `apk`, or `yum` at runtime is a primary indicator of an attacker installing reconnaissance or attack tooling.

```yaml
- list: package_managers
  items: [apt, apt-get, yum, dnf, apk, pip, npm]

- rule: Package Management Process Launched in Container
  desc: Detects installation of software packages inside a running container
  condition: >
    spawned_process and 
    container and 
    proc.name in (package_managers)
  output: >
    Package manager executed inside container 
    (user=%user.name pod=%k8s.pod.name namespace=%k8s.ns.name binary=%proc.name cmdline=%proc.cmdline)
  priority: WARNING
  tags: [container, software_integrity, mitre_persistence]
```

### Rule 2: Detect Outbound Reverse Shell Connections
Catches interactive reverse shells created via `nc`, `bash -i`, or Python:

```yaml
- rule: Outbound Reverse Shell Detected
  desc: Detects attempt to redirect standard streams to a remote network socket
  condition: >
    evt.type in (dup, dup2, dup3) and 
    container and 
    fd.type = "ipv4" and 
    proc.name in (bash, sh, zsh, ksh, python, perl, ruby)
  output: >
    Reverse shell redirect detected in container 
    (pod=%k8s.pod.name ns=%k8s.ns.name process=%proc.name remote_ip=%fd.cip port=%fd.cport)
  priority: CRITICAL
  tags: [network, reverse_shell, mitre_command_and_control]
```

### Rule 3: Detect Theft of Kubernetes ServiceAccount Tokens
Monitors processes attempting to dump mounted service account credentials:

```yaml
- rule: Read Kubernetes ServiceAccount Token
  desc: Detects access to the mounted serviceaccount token by unexpected processes
  condition: >
    open_read and 
    container and 
    fd.name startswith "/var/run/secrets/kubernetes.io/serviceaccount/token" and 
    not proc.name in (my_authorized_app, curl)
  output: >
    Unauthorized read of ServiceAccount token 
    (user=%user.name pod=%k8s.pod.name ns=%k8s.ns.name binary=%proc.name file=%fd.name)
  priority: NOTICE
  tags: [container, credential_access, mitre_credential_access]
```

---

## 4. Handling False Positives with Exceptions

Production environments run legitimate software that triggers rules (e.g. backup scripts running `tar` or CI runners executing `sh`). 

Rather than disabling rules globally, Falco supports structured **`exceptions`**:

```yaml
- rule: Write below etc
  desc: An attempt to write to /etc directory
  condition: write_etc and not package_mgmt_procs
  exceptions:
    - name: fluentbit_certs
      fields: [k8s.pod.name, fd.name]
      comps: [startswith, startswith]
      values:
        - ["logging-fluentbit-", "/etc/ssl/certs/"]
```
- In the example above, writes to `/etc/ssl/certs/` originating from `logging-fluentbit-` pods are silently ignored, while all other writes to `/etc` trigger an alert.

---

## 5. Deploying Custom Rules via Helm

Update your `falco-values.yaml` to mount custom rules dynamically via a Kubernetes ConfigMap:

```yaml
customRules:
  enterprise-security.yaml: |-
    - rule: Disallowed Shell in Production
      desc: Detects interactive shells in production namespace
      condition: >
        spawned_process and 
        container and 
        k8s.ns.name = "production" and 
        proc.name in (bash, sh)
      output: >
        Production shell violation: %proc.name launched in %k8s.pod.name
      priority: ERROR
      tags: [production, compliance]
```

Apply updates:
```bash
helm upgrade falco falcosecurity/falco -n falco -f falco-values.yaml
```

---

## 6. Key Takeaways
- Falco rules evaluate system call conditions in real time using event fields, macros, and lists.
- High-priority rules should target package managers, reverse shells, sensitive file reads, and token theft.
- Never disable rules globally; use structured `exceptions` to whitelist specific pod prefixes and target directories.
- Manage custom enterprise rules as code using Helm values and ConfigMaps.
