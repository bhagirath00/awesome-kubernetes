# Vault Agent Sidecar Injector: Seamless Secret Delivery

## 1. Architectural Concept & Deep Dive

The **Vault Agent Injector** is a Kubernetes Mutating Admission Webhook that intercepts Pod creation requests and dynamically alters the PodSpec to inject two lightweight containers:
1. **`vault-agent-init` (Init Container)**: Authenticates to Vault using the Pod's ServiceAccount token, fetches the requested secrets, writes them to a shared `tmpfs` in-memory volume (`/vault/secrets/`), and terminates *before* the application container starts.
2. **`vault-agent` (Sidecar Container)**: Runs continuously alongside the application container to renew Vault authentication tokens and automatically refresh secrets before their leases expire.

```
+─────────────────────────────────────────────────────────────────────────+
|                    VAULT AGENT INJECTOR PIPELINE                        |
|                                                                         |
|  kubectl apply -f pod.yaml (Annotated with vault.hashicorp.com/...)     |
|                 │                                                       |
|                 ▼                                                       |
|      [ Kubernetes API Server ]                                          |
|                 │                                                       |
|                 ▼ (MutatingAdmissionWebhook)                            |
|      [ Vault Agent Injector Webhook ]                                   |
|                 │                                                       |
|                 ├── Patches PodSpec:                                    |
|                 │   + Injects shared tmpfs volume: /vault/secrets       |
|                 │   + Injects vault-agent-init container                |
|                 │   + Injects vault-agent sidecar container             |
|                 ▼                                                       |
|      [ Scheduled to Worker Node ]                                       |
|                 │                                                       |
|                 ▼ Step 1: vault-agent-init runs                         |
|      Fetches secrets from Vault ──► Writes /vault/secrets/db-config     |
|                 │                                                       |
|                 ▼ Step 2: Application Container Starts                  |
|      Reads /vault/secrets/db-config directly from local memory!         |
|                 │                                                       |
|                 ▼ Step 3: vault-agent sidecar runs                      |
|      Continuously renews token lease & rotates dynamic secrets          |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Core Injection Annotations

Developers control secret injection entirely through Kubernetes metadata annotations:

| Annotation | Description | Example Value |
| :--- | :--- | :--- |
| `vault.hashicorp.com/agent-inject` | Enables the mutating webhook for this pod. | `"true"` |
| `vault.hashicorp.com/role` | The Vault role to authenticate against. | `"payment-app-role"` |
| `vault.hashicorp.com/agent-inject-secret-<NAME>` | Path of the secret inside Vault. | `"secret/data/production/database"` |
| `vault.hashicorp.com/agent-inject-template-<NAME>` | Custom Go/Consul template formatting output. | `See template syntax below` |
| `vault.hashicorp.com/agent-pre-populate-only` | Run init container only; omit running sidecar. | `"true"` (Useful for static CronJobs) |

---

## 3. Hands-on Lab: Deploying an Application with Injected Secrets

### Step 1: Deploy Workload with Annotations
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        # 1. Enable Vault Agent Injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "order-service-role"

        # 2. Specify Secret Path
        vault.hashicorp.com/agent-inject-secret-database.env: "secret/data/production/orders/database"

        # 3. Format Secret with Consul Template Syntax
        vault.hashicorp.com/agent-inject-template-database.env: |
          {{- with secret "secret/data/production/orders/database" -}}
          DATABASE_HOST={{ .Data.data.host }}
          DATABASE_PORT={{ .Data.data.port }}
          DATABASE_USER={{ .Data.data.username }}
          DATABASE_PASSWORD={{ .Data.data.password }}
          {{- end -}}
    spec:
      serviceAccountName: order-service-sa
      containers:
        - name: app
          image: mycompany/orders:v2.1
          command: ["/bin/sh", "-c"]
          # Source the injected secret file into environment variables!
          args: ["source /vault/secrets/database.env && exec ./order-binary"]
          ports:
            - containerPort: 8080
```

### Step 2: Apply and Verify Pod Architecture
```bash
kubectl apply -f deployment.yaml
```

Inspect the running pod containers:
```bash
kubectl get pod -l app=order-service -n production -o jsonpath="{.items[0].spec.containers[*].name}"
```
Output:
```
vault-agent app
```
*(Notice that `vault-agent` was automatically injected alongside your `app` container!)*

Verify the written secret file:
```bash
kubectl exec -it deployment/order-service -n production -c app -- cat /vault/secrets/database.env
```
Output:
```
DATABASE_HOST=postgres.production.internal
DATABASE_PORT=5432
DATABASE_USER=order_admin
DATABASE_PASSWORD=P@ssw0rd!Secure
```

---

## 4. Advanced Templating: JSON, YAML, and Signals

Consul Template allows formatting secrets into configuration formats required by legacy applications:

### Generating a Spring Boot `application.properties` File
```yaml
vault.hashicorp.com/agent-inject-template-application.properties: |
  {{- with secret "secret/data/production/app" -}}
  spring.datasource.url=jdbc:postgresql://{{ .Data.data.db_host }}/orders
  spring.datasource.username={{ .Data.data.username }}
  spring.datasource.password={{ .Data.data.password }}
  {{- end -}}
```

### Triggering Service Reloads upon Secret Rotation (`agent-inject-command`)
When Vault rotates a secret, tell the application process to reload gracefully:
```yaml
vault.hashicorp.com/agent-inject-command-database.env: "kill -HUP 1"
```

---

## 5. Verification & Troubleshooting Runbook

### Health Verification Commands
```bash
# Check Vault Agent sidecar logs
kubectl logs -l app=order-service -n production -c vault-agent --tail=50

# Check Init container logs if pod is stuck in Init:0/1
kubectl logs -l app=order-service -n production -c vault-agent-init
```

### Common Pitfalls & Solutions
1. **Pod stuck in `Init:0/1` or `CrashLoopBackOff` on `vault-agent-init`**:
   - Cause: ServiceAccount token failed authentication with Vault, or the Vault Role does not grant access to the requested path.
   - Solution: Inspect `vault-agent-init` logs. Common errors: `permission denied` (HCL policy missing read capability) or `role not found`.
2. **Secrets not updated when changed in Vault**:
   - Cause: `secret/data` KV v2 engine caching or static secrets lack a TTL.
   - Solution: KV v2 secrets are static. Dynamic secrets (Database/AWS engines) auto-rotate. For static secrets, re-trigger a rollout: `kubectl rollout restart deployment/<NAME>`.

---

## 6. Key Takeaways
- The Vault Agent Injector provides a zero-code-change solution for delivering secrets into pods.
- Injects an init container for startup synchronization and a sidecar for continuous token renewal.
- Secrets are written to an ephemeral in-memory `tmpfs` volume (`/vault/secrets`), never touching worker node disks or etcd.
- Consul Template syntax enables transformation of raw Vault data into `.env`, `.properties`, or `.json` files.
