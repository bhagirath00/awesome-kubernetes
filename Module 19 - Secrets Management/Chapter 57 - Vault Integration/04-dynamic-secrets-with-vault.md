# Dynamic Secrets with Vault: Zero Static Credentials

## 1. Architectural Concept & Deep Dive

In traditional software deployments, every replica of a microservice shares the exact same static database username and password (e.g. `app_user: mypassword123`).

This practice violates zero-trust principles:
- **Blast Radius**: If an attacker compromises a single container or dumps its memory, the entire database is compromised.
- **No Attribution**: Database audit logs show all transactions originating from `app_user`; it is impossible to identify which specific pod or developer performed an action.
- **Impossible Rotation**: Rotating a shared password requires synchronized rolling restarts and risks cascading database connection drops.

**Dynamic Secrets** represent the pinnacle of cloud-native secrets engineering:
1. Credentials **do not exist** until a pod explicitly requests them.
2. Vault dynamically connects to the database (PostgreSQL, MySQL, MongoDB, Oracle) and executes a `CREATE USER` command, generating a **cryptographically unique username and password with a strict Time-to-Live (TTL)**.
3. When the pod terminates or its lease expires, Vault automatically executes a `DROP USER` command, destroying the credential permanently.

```
+─────────────────────────────────────────────────────────────────────────+
|                       DYNAMIC SECRETS LIFECYCLE                         |
|                                                                         |
|  [ Microservice Pod (pod-abc-123) ]                                     |
|         │                                                               |
|         ├── 1. Requests DB Credential from Vault                        |
|         ▼                                                               |
|  [ HashiCorp Vault ]                                                    |
|         │                                                               |
|         ├── 2. Generates dynamic user: v-token-payment-1234             |
|         │   Generates random password: 9x$K!2mP8q                       |
|         │   Sets lease TTL: 1 Hour                                      |
|         ▼                                                               |
|  [ PostgreSQL Database ]                                                |
|         │                                                               |
|         ├── 3. Executes: CREATE ROLE "v-token-..." WITH PASSWORD "..."  |
|         ├── 4. Executes: GRANT SELECT, INSERT ON orders TO "v-token-..."|
|         ▼                                                               |
|  [ HashiCorp Vault ]                                                    |
|         │                                                               |
|         ├── 5. Returns dynamic credential + Lease ID to Pod             |
|         ▼                                                               |
|  [ Microservice Pod connects to PostgreSQL ]                            |
|         │                                                               |
|         ▼ (After 1 hour or when Pod terminates)                         |
|  [ Vault Lease Manager ]                                                |
|         │                                                               |
|         ├── 6. Executes: REVOKE & DROP ROLE "v-token-..."               |
|         ▼                                                               |
|  Credential permanently dead! Zero leftover credentials in production!  |
+─────────────────────────────────────────────────────────────────────────+
```

---

## 2. Configuring the Vault Database Secrets Engine

Let's configure Vault to generate on-demand credentials for a PostgreSQL cluster.

### Step 1: Enable the Database Secrets Engine
```bash
vault secrets enable database
```

### Step 2: Configure Connection to the Database
Vault requires an administrative account to provision dynamic users:

```bash
vault write database/config/postgresql-cluster \
    plugin_name=postgresql-database-plugin \
    allowed_roles="payment-dynamic-role" \
    connection_url="postgresql://{{username}}:{{password}}@postgres.production.svc:5432/orders?sslmode=verify-full" \
    username="vault_admin" \
    password="AdminMasterPassword99"
```

### Step 3: Define the Dynamic Role & Creation Statements
Specify the SQL statements Vault executes when generating new users:

```bash
vault write database/roles/payment-dynamic-role \
    db_name=postgresql-cluster \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    revocation_statements="REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\"; \
        DROP ROLE IF EXISTS \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

---

## 3. Injecting Dynamic Secrets into Kubernetes Pods

Combine the Vault Agent Injector with the dynamic secrets path `database/creds/<ROLE>`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: production
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: payment-api
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "payment-k8s-role"
        
        # 1. Target the Dynamic Secrets Engine
        vault.hashicorp.com/agent-inject-secret-db.env: "database/creds/payment-dynamic-role"
        
        # 2. Template the dynamic credentials into environment variables
        vault.hashicorp.com/agent-inject-template-db.env: |
          {{- with secret "database/creds/payment-dynamic-role" -}}
          export DB_USER="{{ .Data.username }}"
          export DB_PASSWORD="{{ .Data.password }}"
          {{- end -}}
    spec:
      serviceAccountName: payment-sa
      containers:
        - name: app
          image: myorg/payment:v2.0
          command: ["/bin/sh", "-c"]
          args: ["source /vault/secrets/db.env && exec ./payment-server"]
```

---

## 4. Verification & Lifecycle Observation

### Step 1: Inspect Injected Credentials on Pod 1
```bash
kubectl exec -it deployment/payment-api -n production -c app -- cat /vault/secrets/db.env
```
Output:
```
export DB_USER="v-token-payment--a8b9c1d2e"
export DB_PASSWORD="R@nd0mP@ssw0rd99881!"
```

### Step 2: Inspect Injected Credentials on Pod 2
```bash
kubectl exec -it deployment/payment-api -n production -c app -- cat /vault/secrets/db.env
```
Output:
```
export DB_USER="v-token-payment--f4e3d2c1b"
export DB_PASSWORD="An0th3rR@nd0mK3y776!"
```
- **Notice**: Pod 1 and Pod 2 received completely different, unique database usernames and passwords!

### Step 3: Verify Automated Revocation
When a pod is deleted (`kubectl delete pod <NAME>`), the Vault lease expires and Vault automatically executes the `DROP ROLE` command inside PostgreSQL.

Inspect PostgreSQL user list:
```sql
SELECT usename, valuntil FROM pg_user WHERE usename LIKE 'v-token-%';
```
The user is automatically removed once the lease concludes.

---

## 5. Key Takeaways
- Dynamic secrets eliminate shared passwords and static credentials entirely from production.
- Every pod receives a unique, temporary database username and password generated on the fly.
- Leases are automatically renewed by the Vault Agent sidecar while the pod is running.
- When pods terminate or leases expire, Vault immediately drops the database role, ensuring zero credential persistence.
