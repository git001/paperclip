# Paperclip — ARO Deployment (Kustomize)

Deploys Paperclip on Azure Red Hat OpenShift (ARO) using Kustomize.

## Structure

```
deploy/
├── base/                        # shared resources for all environments
│   ├── buildconfig.yaml         # OpenShift BuildConfig (Docker strategy)
│   ├── imagestream.yaml         # internal ARO image registry
│   ├── objectbucketclaim.yaml   # MCG/NooBaa S3 bucket
│   ├── secretstore.yaml         # ESO → Azure Key Vault
│   ├── externalsecret.yaml      # syncs secrets from Key Vault into K8s
│   ├── configmap.yaml           # static env config
│   ├── pvc.yaml                 # persistent volume for /paperclip
│   ├── deployment.yaml          # app deployment (init container writes config.json)
│   ├── service.yaml
│   └── route.yaml               # OpenShift Route (TLS edge termination)
└── overlays/
    ├── dev/                     # namespace: paperclip-dev
    └── prod/                    # namespace: paperclip-prod
```

## Prerequisites

| Component | Purpose |
|-----------|---------|
| ARO 4.11+ | tested version |
| OpenShift Data Foundation (ODF) | provides MCG/NooBaa S3 storage |
| External Secrets Operator (ESO) | syncs Azure Key Vault secrets into K8s |

## Placeholders to fill in

Search for `<...>` in all files before applying:

| Placeholder | File | Description |
|-------------|------|-------------|
| `<ORG>/<REPO>` | `base/buildconfig.yaml` | Git repository |
| `<KEYVAULT_NAME>` | overlay `secretstore` patches | Key Vault name per env |
| `<AZURE_TENANT_ID>` | `base/secretstore.yaml` | Azure AD tenant ID |
| `<CLUSTER_APPS_DOMAIN>` | overlay `configmap` + `route` patches | e.g. `apps.cluster.example.com` |

## Apply

```sh
oc apply -k deploy/overlays/dev
oc apply -k deploy/overlays/prod
```

---

## Azure Key Vault Setup (Service Principal)

ESO authenticates against Azure Key Vault using a Service Principal
(`clientId` + `clientSecret`). The credentials are stored in a K8s Secret
`azure-sp-credentials` that must be created manually in each namespace
before applying Kustomize.

### Step-by-step setup

#### 1. Install External Secrets Operator (ESO)

```sh
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

#### 2. Create a Service Principal

```sh
az ad sp create-for-rbac \
  --name paperclip-eso \
  --skip-assignment \
  --output json
# Note down: appId (CLIENT_ID), password (CLIENT_SECRET), tenant (TENANT_ID)
```

#### 3. Grant read access to each Key Vault

```sh
# dev
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee <CLIENT_ID> \
  --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<KV_RG>/providers/Microsoft.KeyVault/vaults/<DEV_KV_NAME>

# prod
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee <CLIENT_ID> \
  --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<KV_RG>/providers/Microsoft.KeyVault/vaults/<PROD_KV_NAME>
```

#### 4. Create the SP credentials Secret in each namespace

This Secret must exist before `oc apply -k` — ESO reads it to authenticate.

```sh
# dev
oc -n paperclip-dev create secret generic azure-secret-sp \
  --from-literal=ClientID="<CLIENT_ID>" \
  --from-literal=ClientSecret="<CLIENT_SECRET>"

# prod
oc -n paperclip-prod create secret generic azure-secret-sp \
  --from-literal=ClientID="<CLIENT_ID>" \
  --from-literal=ClientSecret="<CLIENT_SECRET>"
```

> Rotate the Secret before the SP password expires (`az ad sp credential reset`).

#### 5. Required Key Vault secrets

The following secrets must exist in each Key Vault before deploying:

| Secret name | Content |
|-------------|---------|
| `paperclip-better-auth-secret` | random hex string (e.g. `openssl rand -hex 32`) |
| `paperclip-database-url` | full Azure PostgreSQL connection string |
| `paperclip-redis-url` | Azure Redis connection string |

```sh
az keyvault secret set --vault-name <KV_NAME> --name paperclip-better-auth-secret --value "$(openssl rand -hex 32)"
az keyvault secret set --vault-name <KV_NAME> --name paperclip-database-url      --value "postgresql://..."
az keyvault secret set --vault-name <KV_NAME> --name paperclip-redis-url         --value "rediss://..."
```

---

## Company CA Certificate

If your cluster routes outbound traffic through a corporate proxy with a
private CA, Node.js (and therefore npm/pnpm) will reject TLS connections
unless the CA certificate is trusted explicitly.

The CA certificate is stored in `deploy/base/company-ca-configmap.yaml` and
mounted into the container at `/etc/pki/ca-trust/custom/ca.crt`. The env var
`NODE_EXTRA_CA_CERTS` points Node.js to that file automatically.

### Adding your CA certificate

> **Important:** `company-ca-configmap.yaml` contains a placeholder only.
> The actual certificate must be added in your **private deployment repo** —
> never commit real certificates to the public paperclip repository.

1. Export your company root CA in PEM format:
   ```sh
   # Example: export from Windows cert store
   # Or obtain ca.pem from your IT/PKI team
   ```
2. Replace the placeholder in `deploy/base/company-ca-configmap.yaml`:
   ```yaml
   data:
     ca.crt: |
       -----BEGIN CERTIFICATE-----
       MII...your actual certificate...
       -----END CERTIFICATE-----
   ```
3. If you have a certificate chain, concatenate all PEM blocks in the same
   `ca.crt` value (root CA last).

---

## Plugins

Paperclip plugins are installed interactively via the web UI at runtime.
The plugin directory is configurable via `PAPERCLIP_PLUGIN_DIR`
(set to `/paperclip/plugins` in `base/configmap.yaml`), which lives on the
PersistentVolume and survives pod restarts.

Outbound npm registry access goes through the corporate proxy — the company
CA (see above) must be configured for plugin installation to work.

---

## Replicas and Horizontal Scaling

**`replicas: 1` is intentional. Do not increase this value.**

Paperclip is architecturally a single-instance application. Running multiple
pods simultaneously causes the following problems:

| Component | Problem |
|-----------|---------|
| **Heartbeat / Routine scheduler** | Each pod runs its own `setInterval`-based scheduler. Without a distributed lock, multiple pods pick up and execute the same jobs concurrently. |
| **Database migrations** | Migrations run on every pod start. There is no Postgres advisory lock protecting against concurrent execution — two pods starting simultaneously can corrupt the schema. |
| **Agent start locks** | Locks are held in-memory (`Map<string, Promise>`) per process only. The same agent can start on two pods at the same time. |
| **DB backup scheduler** | The `databaseBackupInFlight` guard is in-memory. Multiple pods trigger overlapping backups independently. |

**What is safe for multi-instance:** sessions are stored in PostgreSQL
(Better-Auth + Drizzle), so there is no in-memory session state.

### Rolling updates

During a rolling deployment, two pods briefly overlap (old + new). This is
acceptable because:
- The overlap lasts only seconds
- Scheduler race conditions during that window are unlikely to cause data loss
- Migration conflicts are avoided by ensuring the new pod's init container
  finishes before the old pod terminates (default rolling update behavior with
  `maxSurge: 1`, `maxUnavailable: 0`)

If zero-downtime deployments become a hard requirement in the future, the
prerequisite work is: distributed job locking (Postgres advisory locks or
Redis `SET NX`) on heartbeat/routine schedulers, and a dedicated migration
init container that runs only once per rollout.

---

## Storage — Multicloud Object Gateway (MCG)

The `ObjectBucketClaim` in `base/objectbucketclaim.yaml` requests a bucket from
NooBaa/MCG. ODF automatically creates a `ConfigMap` and a `Secret` both named
`paperclip-bucket` in the same namespace. The Deployment reads S3 credentials
directly from these resources — no manual secret management needed.

ODF/OCS must be installed and a `StorageSystem` with NooBaa must be running.
Verify with:

```sh
oc get noobaa -n openshift-storage
```

## Database — Azure PostgreSQL

The database connection string is pulled from Key Vault via ESO and written
into `/paperclip/instances/default/config.json` by the init container on every
pod start. No manual config file management needed.

Ensure the PostgreSQL server allows connections from the ARO egress IP range
and that SSL is enforced (`sslmode=require`).
