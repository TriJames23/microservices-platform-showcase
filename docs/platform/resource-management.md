# Resource Management

**Module**: `shared-services-infrastructure-resource`

The platform's abstraction for binary resource (file/blob) storage — provider-agnostic at the port level.

---

## Semantic Split

Resource semantics are deliberately split into three kernel contracts:

| Contract | Responsibility |
|---|---|
| `IResourcesService` | Logical resource lifecycle ownership around `ResourceId` |
| `IResourceStore` | Byte-level provider/storage operations |
| `IResourceAccessTokenService` | Signed/presigned access-token issuance |

A single port owning all three is a **design violation** — the semantics are distinct and each has different caller audiences.

---

## Supported Storage Providers

Configured via `app.storage.provider`:

| Provider | Value | Use Case |
|---|---|---|
| Local filesystem | `LOCAL` | Local development sandbox |
| Amazon S3 | `S3` | Cloud production |
| Azure Blob Storage | `AZURE` | Azure production |

```yaml
app:
  storage:
    provider: LOCAL       # LOCAL / S3 / AZURE
    local:
      base-path: ./storage
    s3:
      bucket: my-bucket
      region: ap-southeast-1
```

---

## E2E Flow

```
Application depends on ISampleResourcePort
       │
       ▼
TemplateResourceAdaptersConfig wires service-owned adapter
       │
       ▼
TemplateResourceAccessPolicy checks ownership/access rules
       │
       ▼
IResourceStore → provider-specific SDK (hidden behind port)
       │
       ▼
IResourceAccessTokenService issues presigned URL for client download
```

---

## Key Rules

| ✅ Correct | ❌ Forbidden |
|---|---|
| Provider-specific SDK hidden behind `IResourceStore` | Provider SDK types in application or domain |
| Access rules checked at adapter boundary | Resource access rules in controllers |
| `IResourcesService` owns lifecycle | One mixed port for lifecycle + storage + tokens |
| Download/upload token rules in resource module | Token rules in transport modules |

---

## Persistence Integration

Resource metadata (ownership, `ResourceId`, status) is persisted through the standard persistence lane. The `IResourceStore` handles byte storage only — it does not own domain metadata.

Aggregate reconstitution must restore resource references correctly. See [aggregate reconstitution](../deep-dive/aggregate-reconstitution.md).
