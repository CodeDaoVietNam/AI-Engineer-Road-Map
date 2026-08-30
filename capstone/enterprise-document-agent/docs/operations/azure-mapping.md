# Azure port/adapter mapping

## Phạm vi và điều kiện bắt đầu

Azure là deployment option ở M7, không phải dependency của local MVP. Migration
chỉ bắt đầu sau khi local evaluation baseline đạt toàn bộ offline quality,
security và retry/idempotency gates. Local development, deterministic tests và
baseline evaluation không cần Azure/Microsoft Foundry credentials.

Tài liệu này là mapping qua ports/adapters, không phải cam kết provisioned
capacity. **Azure for Students không bảo đảm service availability, region/SKU,
model access hoặc quota** cho Container Apps, Document Intelligence, AI Search,
Microsoft Foundry/Azure OpenAI hay dịch vụ khác. Trước mỗi gate phải kiểm tra
subscription, region, provider registration, model deployment, quota, cost và
data-residency thực tế. Nếu thiếu quota/service, hệ thống giữ local adapter hoặc
chọn provider-neutral adapter khác; không nới quality/security gate.

## Dependency boundary

`packages/contracts` sở hữu stable ports; `packages/adapters` implement local và
Azure adapters; `apps/api`/`apps/worker` composition roots chọn implementation
theo environment. `infrastructure/azure` sở hữu deployment/IaC configuration.

`packages/core` chỉ phụ thuộc standard library + `packages/contracts` và **không
được import Azure SDK**, resource name, Azure credential type hoặc adapter
implementation. Business rule, state machine, tenant isolation, retry budget,
idempotency `document_id + pipeline_version`, citation validation và human review
không thay đổi khi đổi provider.

```text
apps/api, apps/worker -> packages/core + packages/contracts + packages/adapters
packages/core         -> standard library + packages/contracts
packages/adapters     -> packages/contracts + provider SDKs
infrastructure/azure  -> resource/IaC configuration, không domain logic
```

Azure SDKs chỉ xuất hiện trong Azure adapter/infrastructure boundary. Adapter
chuyển provider errors thành stable taxonomy; nó không tự quyết định severity,
case state, `REJECTED` hoặc retry vô hạn ngoài shared policy.

## Platform mapping

| Local/application capability | Azure target | Boundary/contract preserved |
|---|---|---|
| React/Vite `web`, FastAPI `api`, asynchronous `worker` containers | Azure Container Apps | Vẫn ba application processes; Agent là API module, không tạo microservice/multi-agent topology |
| PostgreSQL 16 + pgvector business data, outbox, audit | Azure Database for PostgreSQL | Tenant predicates, transactions/CAS, outbox, immutable versions và durable audit không đổi; migration/backup/runtime roles tách biệt |
| MinIO | Azure Blob Storage | `ObjectStorage`; private containers, server-generated tenant-scoped keys, durability confirmation, encryption/TLS, short-lived authorized access; original bank PDF internal-only |
| RabbitMQ | Azure Service Bus | `MessageQueue`; persistent versioned identifiers-only messages, at-least-once handling, retry/DLQ, trace context; queue không là source of truth |
| PyMuPDF parser | Azure Document Intelligence adapter | `DocumentParser`; page/block/table/bounding-region normalization về provider-neutral schema; PDF vẫn untrusted và provider output không là business state |
| PostgreSQL/pgvector local hybrid search | Azure AI Search adapter | `SearchIndex`; keyword + vector, mandatory `tenant_id + case_id + active document` trước candidate selection, rebuildable projection, top-20/rerank/top-5 configuration |
| Local/model-provider extraction adapter | Microsoft Foundry/Azure OpenAI adapter | `ExtractionModel`; structured schema, controlled repair/budget/version lineage; model không quyết định rule/severity/state |
| Local embedding adapter | Microsoft Foundry/Azure OpenAI embedding adapter | `EmbeddingModel`; versioned dimensions/model and active-result lineage; no unfiltered indexing |
| Local grounded-answer adapter | Microsoft Foundry/Azure OpenAI adapter | `LanguageModel`; seven read-only tools, server-side auth/budget and backend citation validation/refusal |
| Local JWT | Microsoft Entra ID | `IdentityProvider` chỉ xác minh identity/token; application vẫn đọc active `Membership` và derive tenant/role, không tin claims/client tenant selector như authorization hoàn chỉnh |
| Ignored local secrets/runtime credentials | Azure Key Vault + managed identity/workload identity | Secret không vào source/image/log/queue/prompt; least privilege và rotation/audit; local profile không phụ thuộc Key Vault |
| Local OpenTelemetry-compatible logs/metrics/traces | Azure Monitor + Application Insights | Redaction trước export, approved structured fields/trace propagation/SLOs; operational telemetry không thay durable audit |
| Local-built container images | Azure Container Registry (ACR) | Immutable digest/signing/scan/promotion policy; image không chứa secret/customer document |

Microsoft Foundry/Azure OpenAI là adapter options, không được hard-code vào core
hoặc prompt contract. Một deployment có thể dùng khác provider cho extraction,
embedding và language model; run manifest/telemetry phải pin provider, model/
deployment alias và policy version.

## Port mapping chi tiết

### `ObjectStorage` → Azure Blob Storage

Adapter implement bounded stream write/read, durability confirmation, checksum,
metadata và short-lived access. Container private, public listing tắt; application
re-authorize trước mỗi URL. Browser-facing bank access chỉ qua redacted
derivative/authorization-enforcing proxy. Blob/version/retention policy không
được xóa immutable document lineage. PostgreSQL metadata + original object vẫn
là recovery sources.

### `MessageQueue` → Azure Service Bus

Adapter map exchange/queue semantics thành topic/queue/subscription phù hợp nhưng
giữ `DocumentUploaded` schema, `event_id`, verified tenant/case/document/run/
pipeline identifiers, trace/retry metadata và no-PDF payload. Complete message
chỉ sau checkpoint commit; duplicate delivery dùng cùng run/idempotency scope.
Dead-letter không tự replay hoặc đổi case thành `REJECTED`; authorized reprocess
vẫn đi qua API/state/version guards.

### `DocumentParser` → Azure Document Intelligence

Adapter data-minimize request, enforce timeout/resource budget và normalize
provider pages/tables/coordinates về provider-neutral page/evidence contract.
Page number API vẫn one-based. Output được schema-validate; unsupported/
malformed output dùng stable transient/permanent error classification. Provider
không nhận storage credential rộng và không được thực thi instruction trong PDF.

### Model ports → Microsoft Foundry/Azure OpenAI

`ExtractionModel`, `EmbeddingModel` và `LanguageModel` adapters tách biệt, dù có
thể dùng cùng provider. Chúng enforce configured deployment, timeout, token/cost
budget, output schema và safe telemetry. Azure SDK response không đi vào core;
raw prompt/completion không vào logs. Agent tool allowlist, tenant bind,
read-only boundary và citation validator vẫn ở application/server code.

### `SearchIndex` → Azure AI Search

Index schema chứa tenant/case/document version, active processing/index lineage,
page/chunk/hash và safe text/vector fields. Adapter áp mandatory metadata filter
cho keyword và vector query trước merge/rerank; không fallback unfiltered. Index
là shadow/rebuildable projection từ PostgreSQL metadata + original/derived
document content; stale projection không là recovery truth.

### `IdentityProvider` → Microsoft Entra ID

Entra adapter validate issuer, audience, signature, expiry và mapped identity.
Entra group/claim không thay server-side active `Membership`, role permission,
resource tenant và state/version checks. Client/model claim `tenant_id` không
authoritative; cross-tenant lookup vẫn `404 RESOURCE_NOT_FOUND` sau identity/
membership/role checks.

### `AuditPublisher` và telemetry

`AuditPublisher` tiếp tục ghi/publish append-only tenant-scoped audit lineage qua
provider-neutral persistence/outbox contract; PostgreSQL record là authoritative.
Azure Monitor/Application Insights nhận redacted operational telemetry và có thể
nhận safe audit notification, nhưng không thay `AuditEvent` hoặc chứa full bank/
document/prompt data.

## Identity, secrets và network posture

- Container Apps dùng managed identity/workload identity với scope riêng cho
  Blob, Service Bus, Key Vault, AI services và telemetry; không dùng one shared
  broad credential.
- Key Vault access, secret rotation và configuration changes là privileged,
  audited actions. Secret references resolve runtime; không bake vào ACR image.
- PostgreSQL, Blob, Service Bus, model/search/parser endpoints dùng TLS và private
  connectivity/network restrictions khi subscription/region hỗ trợ; public
  fallback cần explicit security review, không tự bật.
- Egress allowlist theo adapter dependency; PDF/user URLs không tạo SSRF hoặc
  arbitrary provider call.
- Data residency, retention, backup/restore, provider training/data-use settings
  và diagnostic export phải được review trước customer data. Baseline/canary chỉ
  dùng synthetic data đến khi gate này pass.

## Reversible migration gates

Mỗi gate tạo evidence/report và có rollback owner. Không cut over nhiều adapters
cùng lúc nếu lỗi không thể quy về một boundary.

### Gate 0 — Local baseline frozen

- Tất cả offline gates đạt: classification macro F1 ≥ **0.90**, field extraction
  F1 ≥ **0.85**, critical-field recall ≥ **0.95**, Retrieval Recall@5 ≥
  **0.90**, citation correctness ≥ **0.90**, answer faithfulness ≥ **0.90**,
  correct refusal ≥ **0.85**, tool-selection accuracy ≥ **0.90**, deterministic
  rules và tenant-isolation tests **100%**.
- Retry/idempotency scenarios không mất acknowledged document hoặc tạo duplicate
  extraction/evidence/chunk. Local SLO report được ghi riêng khỏi quality.
- Freeze dataset/supplier split, run manifest, port contracts và local adapter
  baseline. Nếu fail, không provision/cut over để che lỗi.

### Gate 1 — Feasibility và provision isolated environment

Kiểm tra subscription/region/service/SKU/model quota/cost/data residency thực
tế; document kết quả thay vì giả định Azure for Students entitlement. Provision
isolated synthetic staging bằng IaC, Key Vault/managed identity, network và ACR
digest. Rollback là destroy/disable isolated resources; local stack không đổi.

### Gate 2 — Adapter contract parity

Chạy cùng provider-neutral contract, security, failure và telemetry tests cho
từng Azure adapter với synthetic fixtures. Verify error taxonomy, at-least-once
delivery, tenant filters, redaction, trace propagation, rebuild/restore và
least-privilege denial. Rollback là deselect adapter và giữ local implementation.

### Gate 3 — Shadow evaluation

Replay immutable synthetic inputs sang một Azure adapter/projection ở shadow
environment; không dual-write authoritative business decisions. So outputs và
mọi approved offline gate với frozen local baseline, báo cost/latency/SLO riêng.
AI Search shadow index không phục vụ user traffic. Regression hoặc quota
instability quay lại Gate 2/local adapter.

### Gate 4 — Staging composition/canary

Compose full Azure staging qua ports, chạy end-to-end, tenant isolation 100%,
failure injection, backup/restore, DLQ/reprocess, audit và SLO workload. Canary
chỉ dùng synthetic/non-customer environment cho đến khi data governance pass.
Feature/config switch phải cho phép revert adapter; rollback runbook xác nhận
source-of-truth pointers và không surface stale index/result.

### Gate 5 — Production cutover decision

Chỉ sau explicit security/data/cost/operations approval, quota reservation và
tested data migration/rollback plan. PostgreSQL/object authoritative writes
không dual-write tùy tiện. Cutover theo bounded cohort/adapter, monitor SLO burn,
quality samples, tenant isolation, retry/idempotency và cost. Bất kỳ security/
durability invariant fail nào lập tức fail closed và rollback; quality metric
khác không bù được.

## SLO và quality sau mapping

Azure deployment giữ cùng operational targets: upload acknowledgement P95 dưới
500 ms, successful processing P95 dưới hai phút, retrieval P95 dưới một giây,
non-streaming chat P95 dưới 10 giây và availability 99,5%. Các target này được
đo ở Azure environment riêng và không thay offline quality gates. Provider quota
thấp hoặc service unavailable là deployment blocker/degraded operation, không
là lý do đổi metric definition, bỏ tenant filter hoặc phát unsupported answer.

## Liên quan

- [Tổng quan kiến trúc](../architecture/system-overview.md)
- [Security model](../security/security-model.md)
- [Chiến lược evaluation](../evaluation/evaluation-strategy.md)
- [Local development](local-development.md)
- [Observability](observability.md)
