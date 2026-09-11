# Langfuse Setup

Langfuse is initialized headlessly. The organization, project, user and API key
are declared up front through `LANGFUSE_INIT_*` variables. Langfuse creates them
on boot if they do not already exist. `vault-init` writes the same key to Vault,
where the LLM Orchestration Service reads it.

There is no manual UI step. You do not create an API key in the Langfuse
interface.

---

## How the values flow

You supply the values in **one place** at deploy time. They reach both consumers
from there:

```
--set / values file
        │
        ▼
Langfuse-Web/values.yaml  (env: LANGFUSE_INIT_*)
        │
        ├──→ langfuse-web container
        │      creates org, project, user and API key on boot
        │
        └──→ secret.yaml renders them into the
             langfuse-web-secrets Secret
                    │
                    └──→ Vault-Init Job reads it via envFrom
                           writes to Vault at secret/data/langfuse/config
                                  │
                                  └──→ LLM Orchestration Service reads it
                                       at startup
```

Nothing needs to be kept in sync by hand. The key exists in one place.

---

## Before you deploy

### 1. Generate the credentials

```bash
PK="pk-lf-$(uuidgen | tr '[:upper:]' '[:lower:]')"
SK="sk-lf-$(uuidgen | tr '[:upper:]' '[:lower:]')"
PW=$(openssl rand -base64 24)

echo "PUBLIC KEY:   $PK"
echo "SECRET KEY:   $SK"
echo "ADMIN PASSWD: $PW"
```

Save these somewhere. You will need the password to log into the Langfuse UI.

**Do not commit them.** Gitleaks runs as a pre-commit hook and on every push, and
will block you.

### 2. Create a local values file

Create `values-secrets.yaml` in the `kubernetes/` directory:

```yaml
Langfuse-Web:
  env:
    LANGFUSE_INIT_PROJECT_PUBLIC_KEY: "pk-lf-..."
    LANGFUSE_INIT_PROJECT_SECRET_KEY: "sk-lf-..."
    LANGFUSE_INIT_USER_PASSWORD: "..."
```

Add it to `.gitignore`:

```bash
echo "values-secrets.yaml" >> .gitignore
```

The non-sensitive fields (org ID, org name, project ID, project name, user email,
user name) are already set in `charts/Langfuse-Web/values.yaml` and do not need
to be supplied.

**Why the key fields ship empty:** `LANGFUSE_INIT_*` only creates resources that
do not already exist. If the chart shipped a placeholder key, Langfuse would
create the project bound to that placeholder, and a real key supplied later
would be silently ignored — the only recovery would be wiping the database.
Shipping empty means a missed override logs a skip line instead of permanently
binding the project to a wrong key.

---

## Deploy

```bash
cd kubernetes
helm dependency build
helm install <release> . -n <namespace> -f values.yaml -f values-secrets.yaml
```

Alternatively, without a values file:

```bash
helm install <release> . -n <namespace> \
  --set Langfuse-Web.env.LANGFUSE_INIT_PROJECT_PUBLIC_KEY="$PK" \
  --set Langfuse-Web.env.LANGFUSE_INIT_PROJECT_SECRET_KEY="$SK" \
  --set Langfuse-Web.env.LANGFUSE_INIT_USER_PASSWORD="$PW"
```

---

## Verify

### 1. Pods are up

```bash
kubectl get pods -n <namespace>
```

All of the following must be `Running` or `Completed`:

| Pod | Purpose |
|---|---|
| `rag-search-db-0` | PostgreSQL (hosts `rag-search` and `langfuse-db`) |
| `minio-*` | Object storage for Langfuse events and media |
| `redis-*` | Queue backend for the Langfuse worker |
| `clickhouse-*` | Analytics database for Langfuse ingestion |
| `langfuse-worker-*` | Must be `Running` before web starts |
| `langfuse-web-*` | UI; runs database migrations on first boot |
| `vault` | Secret storage |
| `vault-init` | Unseals Vault and writes the Langfuse config |

On first startup `langfuse-web` runs database migrations, which takes one to two
minutes. Langfuse health probes are disabled by default, so `Running` alone is
not proof the UI is up.

### 2. The key reached Vault

```bash
kubectl logs job/vault-init -n <namespace> | grep -i langfuse
```

Expected:
```
Langfuse config stored at secret/data/langfuse/config
```

If you see `Langfuse init variables not set, skipping Langfuse config`, the
values did not reach the Secret — check step 2 above.

To read the stored value directly:

```bash
kubectl exec -it vault-0 -n <namespace> -- sh
# root token is in /vault/file/unseal-keys.json
vault kv get secret/langfuse/config
```

### 3. LLM Orchestration Service picked it up

```bash
kubectl logs deploy/llm-orchestration-service -n <namespace> | grep -i langfuse
```

Expected: `Langfuse client initialized successfully`

If you see `Langfuse secrets not found in Vault`, see the startup ordering note
below.

### 4. End to end

Port-forward the UI:

```bash
kubectl port-forward svc/langfuse-web 3005:3005 -n <namespace>
```

Open `http://localhost:3005`, log in with `LANGFUSE_INIT_USER_EMAIL` and the
password you generated. Confirm the project exists and the API key under
Settings matches your public key.

Then send one real request to the LLM Orchestration Service (`POST /orchestrate`)
and confirm a trace appears in the Langfuse UI within a few seconds.

**Logs alone are not proof.** A key can be present in Vault and still be wrong.
The trace appearing is the only check that covers the whole chain.

---

## Known limitations

**Startup ordering.** The LLM Orchestration Service reads
`secret/data/langfuse/config` once, at startup. The Vault-Init Job and the LLM
deployment start in parallel with no wait between them. If the Job has not
finished writing when the service starts, tracing stays off until the pod
restarts — silently, with no error beyond the log line in step 3.

Workaround:

```bash
kubectl rollout restart deployment llm-orchestration-service -n <namespace>
```

A proper fix would be an init container that polls Vault until the path responds.

**Vault-Init Job immutability.** The Job has a fixed name and a `batch/v1` Job
spec is immutable in Kubernetes. `helm upgrade` alone will not rerun it. To force
a rerun:

```bash
kubectl delete job vault-init -n <namespace>
helm upgrade <release> . -n <namespace> -f values.yaml -f values-secrets.yaml
```

**Two copies of the init script.** `vault-init.sh` at the repository root and the
embedded copy in `charts/Vault-Init/templates/configmap.yaml` are the same script
maintained by hand in two places. Any change must be applied to both.

---

## Notes

In Kubernetes the Langfuse-Web service port is `3005`, mapped to container port
`3000`. `LANGFUSE_HOST` is therefore set explicitly to
`http://langfuse-web:3005` in `charts/Vault-Init/values.yaml`. In Docker Compose
service-to-service traffic uses the container port, so `3000` is correct there.
The two defaults differ on purpose.

`charts/Vault-Init/values.yaml` holds only `host` and `secretName` under
`langfuse:`. It must not contain the key values themselves — the Job reads those
from the Secret via `envFrom`. Duplicating them across two values files
reintroduces the risk of Vault holding a key Langfuse never created.