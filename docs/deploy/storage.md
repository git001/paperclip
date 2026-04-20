---
title: Storage
summary: Local disk, S3-compatible, and Azure Blob Storage
---

Paperclip stores uploaded files (issue attachments, images) using a configurable storage provider.

## Local Disk (Default)

Files are stored at:

```
~/.paperclip/instances/default/data/storage
```

No configuration required. Suitable for local development and single-machine deployments.

## S3-Compatible Storage

For production or multi-node deployments, use S3-compatible object storage (AWS S3, MinIO, Cloudflare R2, etc.).

Configure via CLI:

```sh
pnpm paperclipai configure --section storage
```

## Azure Blob Storage

For deployments on Azure or AKS, Paperclip has a native Azure Blob Storage provider (uses the `@azure/storage-blob` SDK directly — no S3 compatibility layer needed).

### Authentication

Either a connection string **or** an account name + key is required:

**Option A — Connection string:**

```sh
PAPERCLIP_STORAGE_PROVIDER=azure_blob
PAPERCLIP_STORAGE_AZURE_BLOB_CONNECTION_STRING="DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...;EndpointSuffix=core.windows.net"
PAPERCLIP_STORAGE_AZURE_BLOB_CONTAINER=paperclip
```

**Option B — Account name + key:**

```sh
PAPERCLIP_STORAGE_PROVIDER=azure_blob
PAPERCLIP_STORAGE_AZURE_BLOB_ACCOUNT_NAME=mystorageaccount
PAPERCLIP_STORAGE_AZURE_BLOB_ACCOUNT_KEY=<base64-key>
PAPERCLIP_STORAGE_AZURE_BLOB_CONTAINER=paperclip
```

### All Azure env vars

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `PAPERCLIP_STORAGE_PROVIDER` | yes | `local_disk` | Set to `azure_blob` |
| `PAPERCLIP_STORAGE_AZURE_BLOB_CONNECTION_STRING` | one of A or B | — | Full Azure connection string |
| `PAPERCLIP_STORAGE_AZURE_BLOB_ACCOUNT_NAME` | one of A or B | — | Storage account name |
| `PAPERCLIP_STORAGE_AZURE_BLOB_ACCOUNT_KEY` | one of A or B | — | Storage account key (base64) |
| `PAPERCLIP_STORAGE_AZURE_BLOB_CONTAINER` | yes | `paperclip` | Container name |
| `PAPERCLIP_STORAGE_AZURE_BLOB_PREFIX` | no | `` | Optional blob key prefix |

The container must already exist before starting Paperclip.

## Configuration

| Provider | Best For |
|----------|----------|
| `local_disk` | Local development, single-machine deployments |
| `s3` | Production, multi-node, AWS/MinIO/R2 deployments |
| `azure_blob` | Production, Azure / AKS deployments |

Storage configuration is stored in the instance config file:

```
~/.paperclip/instances/default/config.json
```
