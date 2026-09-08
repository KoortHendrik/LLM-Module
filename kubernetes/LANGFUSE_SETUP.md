# Langfuse Setup

Langfuse is initialized headlessly. The org, project, user and API key are
declared up front through the `LANGFUSE_INIT_*` variables in
`charts/Langfuse-Web/values.yaml` (non-sensitive fields) and the
`langfuse-web-secrets` Secret (`LANGFUSE_INIT_PROJECT_SECRET_KEY`,
`LANGFUSE_INIT_USER_PASSWORD`). Langfuse creates them on boot if they do not
already exist.

`vault-init` writes the same key to Vault at `secret/data/langfuse/config`
(fields `public_key`, `secret_key`, `host`), where the LLM Orchestration Service
reads it. This runs on every deployment, so there is no manual step.

> The public key in `charts/Vault-Init/values.yaml` must match the one in
> `charts/Langfuse-Web/values.yaml`, and the secret key is read from the
> `langfuse-web-secrets` Secret by both charts.

> **Note:** In Kubernetes the Langfuse-Web service port is `3005` (mapped to
> container port 3000), so `LANGFUSE_HOST` is set explicitly to
> `http://langfuse-web:3005`.

## 1. Verify Required Pods

```bash
kubectl get pods -n your-namespace
```

All of the following must be `Running` or `Completed` — Langfuse will not start without them:

| Pod | Purpose |
|---|---|
| `rag-search-db-0` | PostgreSQL (hosts `rag-search` and `langfuse-db`) |
| `minio-*` | Object storage for Langfuse events/media |
| `redis-*` | Queue backend for Langfuse worker |
| `clickhouse-*` | Analytics DB for Langfuse ingestion |
| `langfuse-worker-*` | Must be `Running` before web starts |
| `langfuse-web-*` | UI + runs DB migrations on first boot |
| `vault` | Secret storage |
| `vault-init` | Unseals Vault and writes the Langfuse config |

On first startup `langfuse-web` runs database migrations, which takes 1–2
minutes. To confirm the key reached Vault, check the `vault-init` logs for
`Langfuse config stored at secret/data/langfuse/config`, and the LLM
Orchestration Service logs for `Langfuse client initialized successfully`.
