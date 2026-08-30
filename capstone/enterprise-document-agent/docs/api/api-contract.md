# API contract

## Phạm vi và quy ước phiên bản

Mọi endpoint MVP nằm dưới `/api/v1`. Thay đổi tương thích ngược có thể được bổ
sung trong `v1`; thay đổi tên field, meaning, state hoặc error code cần version
API mới. API dùng JSON, trừ upload dùng `multipart/form-data`, và thời gian dùng
UTC ISO 8601. Identifier là opaque string; client không được suy ra hoặc tự tạo
`tenant_id`.

Authenticated identity và active `Membership` là nguồn duy nhất của
`user_id`, `tenant_id` và role. Mỗi handler phải kiểm tra theo thứ tự:

1. identity đã xác thực;
2. membership hiện hành còn active;
3. role có permission cho action;
4. resource, parent resource và mọi referenced resource thuộc tenant hiện hành;
5. resource state/version cho phép action.

Không request body, query, path, model output hoặc Agent tool argument nào được
dùng làm nguồn authoritative cho tenant identity. Lookup cross-tenant luôn trả
`404 RESOURCE_NOT_FOUND`, cùng envelope như resource không tồn tại, để không lộ
sự tồn tại. `401` chỉ dành cho unauthenticated identity; `403` chỉ dành cho
inactive membership/role denial không phụ thuộc resource existence.

## HTTP, request ID và envelopes

Client có thể gửi `X-Request-ID`; server validate format hoặc sinh mới, trả lại
`X-Request-ID` trên mọi response và ghi cùng trace. Một success response JSON có
envelope:

```json
{
  "data": {
    "case_id": "case_01J...",
    "state": "DRAFT"
  },
  "meta": {
    "request_id": "req_01J..."
  }
}
```

Collection dùng cursor opaque, thứ tự ổn định và giới hạn do server cap:

```json
{
  "data": [{"supplier_id": "sup_01J...", "legal_name": "Acme Ltd"}],
  "meta": {
    "request_id": "req_01J...",
    "next_cursor": "eyJ2IjoxLCJpZCI6Ii4uLiJ9",
    "has_more": true
  }
}
```

Query chung là `?cursor=<opaque>&limit=50`; `limit` mặc định `50`, tối đa `100`.
Cursor bind tenant, endpoint, filter và sort; cursor sai scope trả
`INVALID_CURSOR`. Không dùng offset để tránh skip/duplicate khi có ghi đồng thời.

Mọi lỗi dùng envelope ổn định, không chứa stack trace hoặc existence hint:

```json
{
  "error": {
    "code": "CASE_NOT_READY_FOR_REVIEW",
    "message": "The supplier case is not ready for review.",
    "request_id": "req_01J...",
    "details": {
      "current_state": "VALIDATION_REQUIRED"
    }
  }
}
```

`details` là optional, machine-readable và không chứa dữ liệu nhạy cảm. HTTP
status mang class lỗi; client branch theo stable `error.code`, không parse
`message`.

## Idempotency và optimistic concurrency

`Idempotency-Key` là header bắt buộc cho create supplier/case, upload,
reprocess, chat và review decision. Key là opaque, duy nhất trong scope
authenticated tenant + route + actor trong retention window. Canonical request
fingerprint gồm HTTP method + path, canonical payload, exact `If-Match` value và,
với upload, SHA-256 của byte stream đã validate. Filename, multipart boundary và
request ID không làm thay đổi fingerprint.

Sau khi xác minh identity, active membership, role permission và resource thuộc
authenticated tenant, handler lookup/reserve idempotency record **trước** mọi
mutable resource-state, active-snapshot hoặc optimistic-version guard. Cùng key
và cùng fingerprint trả nguyên HTTP status, headers cần thiết và body đã lưu;
cùng key với fingerprint khác trả `409 Conflict` và
`IDEMPOTENCY_KEY_REUSED`. Với upload, server phải stream có giới hạn để validate
và tính SHA-256 trước khi hoàn tất comparison; chưa có durable object/business
side effect cho đến khi replay/guards đã được giải quyết.

Write thành công lưu idempotency key, fingerprint, business record/result và
serialized response atomically trong cùng transaction. Vì vậy retry review cùng
key/fingerprint trả lại original `201` kể cả decision đầu đã đổi case thành
`APPROVED`/`REJECTED` và case version hiện tại không còn khớp `If-Match`. Chỉ
request mới đi tiếp tới mutable state/version guards. Key không thay worker
idempotency `document_id + pipeline_version`.

Case responses trả `ETag: "case-v12"`. Mutation document/reprocess/review mang
`If-Match: "case-v12"`; thiếu precondition trả `428 PRECONDITION_REQUIRED`, stale
version trả `409 Conflict` với `CASE_VERSION_CONFLICT`. Server compare-and-set
case version, active document/result set và active validation pointer trong
transaction; client không được dùng last-write-wins.

Membership detail response trả version và `ETag: "membership-v7"`.
`PATCH /api/v1/memberships/{membership_id}` bắt buộc `If-Match`; server
compare-and-set target membership version cùng role/active update và audit
event. Thiếu header trả `428 PRECONDITION_REQUIRED`; stale version trả `409 Conflict` với
`MEMBERSHIP_VERSION_CONFLICT` và không ghi đè update đồng thời.

## Endpoint catalog

| Method | Path | Mục đích | Success |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | Local JWT login; provider adapter có thể thay thế | `200` |
| `GET` | `/api/v1/auth/me` | Identity, active tenant membership và role hiện hành | `200` |
| `GET` | `/api/v1/memberships` | Tenant Admin liệt kê membership tenant | `200` |
| `GET` | `/api/v1/memberships/{membership_id}` | Tenant Admin đọc membership version/ETag | `200` |
| `PATCH` | `/api/v1/memberships/{membership_id}` | Tenant Admin đổi role/active status | `200` |
| `GET` | `/api/v1/suppliers` | Liệt kê supplier tenant-scoped | `200` |
| `POST` | `/api/v1/suppliers` | Tạo supplier, server gán tenant | `201` |
| `GET` | `/api/v1/suppliers/{supplier_id}` | Xem supplier cùng tenant | `200` |
| `GET` | `/api/v1/suppliers/{supplier_id}/cases` | Liệt kê cases của supplier | `200` |
| `POST` | `/api/v1/suppliers/{supplier_id}/cases` | Tạo `SupplierCase` ở `DRAFT` | `201` |
| `GET` | `/api/v1/cases` | Liệt kê cases trong tenant, filter state | `200` |
| `GET` | `/api/v1/cases/{case_id}` | Xem case state/version và active snapshot | `200` |
| `GET` | `/api/v1/cases/{case_id}/documents` | Liệt kê document versions/active marker | `200` |
| `POST` | `/api/v1/cases/{case_id}/documents` | Upload initial/additional/replacement PDF | `202 Accepted` |
| `GET` | `/api/v1/documents/{document_id}` | Document metadata và active version | `200` |
| `GET` | `/api/v1/documents/{document_id}/status` | Current processing-lineage leaf/checkpoint | `200` |
| `GET` | `/api/v1/documents/{document_id}/evidence` | Evidence metadata và short-lived access | `200` |
| `POST` | `/api/v1/documents/{document_id}/reprocess` | Tạo explicit reprocess `ProcessingRun` | `202 Accepted` |
| `GET` | `/api/v1/documents/{document_id}/fields` | Active successful extracted fields | `200` |
| `GET` | `/api/v1/cases/{case_id}/issues` | Issues của active `ValidationRun` | `200` |
| `POST` | `/api/v1/cases/{case_id}/chats` | Grounded non-streaming answer hoặc refusal | `200` |
| `GET` | `/api/v1/cases/{case_id}/decisions` | Review decision history | `200` |
| `POST` | `/api/v1/cases/{case_id}/decisions` | Human `APPROVED` hoặc `REJECTED` decision | `201` |
| `GET` | `/api/v1/auditevents` | Tenant Admin tenant-scoped audit timeline | `200` |

Collection endpoints hỗ trợ cursor pagination. Supplier/case/audit filters là
allowlist; mọi filter được kết hợp với mandatory tenant predicate. Không endpoint
nào nhận `tenant_id` làm authorization input.

## Representative contracts

### Authentication context

`GET /api/v1/auth/me`:

```json
{
  "data": {
    "user_id": "usr_01J...",
    "display_name": "R. Nguyen",
    "membership": {
      "membership_id": "mem_01J...",
      "tenant_id": "ten_01J...",
      "role": "Reviewer",
      "active": true
    }
  },
  "meta": {"request_id": "req_01J..."}
}
```

`tenant_id` ở response chỉ mô tả server-derived context; gửi lại giá trị này
không cấp quyền.

### Membership update

Tenant Admin đọc membership version/ETag rồi gọi
`PATCH /api/v1/memberships/{membership_id}` với
`If-Match: "membership-v7"`:

```json
{
  "role": "Reviewer",
  "active": true
}
```

Response `200` trả version mới và `ETag: "membership-v8"`. Server load target
bằng authenticated tenant predicate, chạy compare-and-set trên version `7`, cập
nhật membership và ghi audit atomically. Retry không được bỏ qua permission;
stale update trả `409 MEMBERSHIP_VERSION_CONFLICT`.

### Supplier và case

`POST /api/v1/suppliers` với `Idempotency-Key`:

```json
{
  "legal_name": "Acme Supplies Ltd",
  "external_reference": "ERP-1042"
}
```

Response `201` chứa `supplier_id`; server không chấp nhận `tenant_id` hoặc role
trong create payload. `POST /api/v1/suppliers/{supplier_id}/cases` nhận optional
`external_reference` và trả:

```json
{
  "data": {
    "case_id": "case_01J...",
    "supplier_id": "sup_01J...",
    "state": "DRAFT",
    "version": 1
  },
  "meta": {"request_id": "req_01J..."}
}
```

### Upload và processing status

`POST /api/v1/cases/{case_id}/documents` dùng `multipart/form-data`,
`Idempotency-Key` và `If-Match`. Fields chỉ gồm `document_type`, `file` và
optional `replaces_document_id`; `document_type` là một trong
`company_profile`, `business_registration`, `tax_registration`,
`bank_information_form`, `quotation`.

Server phân biệt hai command, không tự suy từ filename:

- **new-slot upload:** chưa có active `Document` cho
  `tenant_id + case_id + document_type`; `replaces_document_id` phải vắng mặt;
- **replacement:** đã có active document cho type đó;
  `replaces_document_id` là bắt buộc và phải trỏ chính xác active document cùng
  authenticated tenant, case và document type.

Nếu active slot đã tồn tại nhưng target bị thiếu/stale/inactive/sai case hoặc sai
type trong cùng tenant, server trả `409 DOCUMENT_REPLACEMENT_CONFLICT`. Target
cross-tenant luôn trả `404 RESOURCE_NOT_FOUND`. Case compare-and-set cùng unique
constraint “một active document mỗi `tenant_id + case_id + document_type`” ngăn
hai request đồng thời tạo hai active versions hoặc nối sai version lineage.

Với initial upload từ `DRAFT`, sau khi PDF ≤ 20 MB, ≤ 50 pages đã được xác minh,
object write durable, và `Document` + initial `ProcessingRun(QUEUED)` +
`DocumentUploaded` outbox row đã commit atomically, response là
`202 Accepted`:

```json
{
  "data": {
    "document_id": "doc_01J...",
    "status_url": "/api/v1/documents/doc_01J.../status",
    "case_state": "DOCUMENTS_UPLOADED",
    "case_version": 2
  },
  "meta": {"request_id": "req_01J..."}
}
```

`202 Accepted` không hứa processing thành công. `GET .../status` trả business
state riêng với technical status:

```json
{
  "data": {
    "document_id": "doc_01J...",
    "active": true,
    "processing_run_id": "run_01J...",
    "processing_status": "RETRY_PENDING",
    "checkpoint": "EXTRACTED",
    "retry_count": 1,
    "next_attempt_at": "2026-08-30T08:30:00Z",
    "case_state": "PROCESSING"
  },
  "meta": {"request_id": "req_01J..."}
}
```

Reprocess dùng cùng headers, không nhận pipeline/model/tenant override, và trả
`202 Accepted` với `processing_run_id`, `status_url`, incremented `case_version`
và `case_state: "PROCESSING"`. Nó tạo run mới; không mutate `FAILED` run trở lại
`RUNNING`.

Mọi document-set mutation sau initial upload — new-slot additional upload,
replacement hoặc reprocess — trong cùng transaction phải invalidate active
`ValidationRun`, clear affected active processing/index pointers, tăng case
version, tạo run/outbox cần thiết và đưa case về `PROCESSING`. Quy tắc này áp
dụng dù state trước là `DOCUMENTS_UPLOADED`, `PROCESSING`,
`VALIDATION_REQUIRED` hay `READY_FOR_REVIEW`; pointer đã null vẫn được xử lý
idempotently. Response `202 Accepted` của ba mutation này luôn trả
`case_state: "PROCESSING"` và version mới.

### Evidence và extracted fields

`GET /api/v1/documents/{document_id}/evidence` trả evidence của active document
và active successful result. Page number trong API là **one-based**. URL PDF là
signed URL thời hạn ngắn chỉ được tạo sau application-layer authorization:

```json
{
  "data": {
    "document_id": "doc_01J...",
    "document_version": 2,
    "view_url": "https://storage.example/signed-object",
    "view_url_expires_at": "2026-08-30T08:05:00Z",
    "items": [{
      "evidence_id": "ev_01J...",
      "page": 3,
      "evidence_snippet": "Account number ending in 4821",
      "bounding_region": {
        "coordinate_system": "pdf_points",
        "unit": "point",
        "x": 72.0,
        "y": 210.0,
        "width": 180.0,
        "height": 18.0
      }
    }]
  },
  "meta": {"request_id": "req_01J..."}
}
```

Với `bank_information_form`, `view_url` chỉ trỏ redacted derivative hoặc
authorization-enforcing proxy đã mask account number; API không phát signed URL
trực tiếp tới original bank PDF. Original object là internal-only cho worker và
authorized recovery path. Future unmask ngoài MVP phải có permission riêng,
step-up authentication, mục đích/TTL tối thiểu và immutable audit event; role
Reviewer/Tenant Admin hiện tại tự nó không cấp unmask.

`GET .../fields` returns `raw_value`, `normalized_value`, confidence,
schema/pipeline lineage và evidence references. Bank account values are always
masked outside the encryption boundary; stale/non-active run output is not
surfaced.

### Validation issues

`GET /api/v1/cases/{case_id}/issues?severity=ERROR` returns only the active
`ValidationRun`. During processing after snapshot invalidation it returns an
empty collection plus `validation_status: "PENDING"`, never historical issues
as current. A missing-document issue may have no evidence and instead includes
machine-readable `missing_scope`; mismatch issues cite evidence for operands.

### Grounded chat

`POST /api/v1/cases/{case_id}/chats`:

```json
{
  "question": "What is the quotation total?"
}
```

The server binds auth context, enforces read-only tools and budgets, then
validates every citation. A grounded response is:

```json
{
  "data": {
    "answer": "The quotation total is USD 12,400.00.",
    "refused": false,
    "citations": [{
      "document_id": "doc_01J...",
      "document_type": "quotation",
      "document_version": 1,
      "page": 2,
      "evidence_id": "ev_01J...",
      "evidence_snippet": "Total USD 12,400.00"
    }]
  },
  "meta": {"request_id": "req_01J..."}
}
```

Insufficient/invalid evidence returns `200` with `refused: true`, a safe reason
code such as `INSUFFICIENT_EVIDENCE`, and no unsupported claim. Tool/model
technical failure may return `503` `CHAT_GROUNDING_UNAVAILABLE`; it does not
change case or processing state.

### Review decision

`POST /api/v1/cases/{case_id}/decisions` requires Reviewer or Tenant Admin,
`Idempotency-Key` and current `If-Match`:

```json
{
  "decision": "APPROVED",
  "validation_run_id": "val_01J...",
  "reason": "Verified against the active document set."
}
```

Sau auth/membership/role/tenant-resource checks, server lookup idempotency record
trước khi đọc mutable state/version. Existing same-fingerprint record replay
original `201`; different fingerprint trả `IDEMPOTENCY_KEY_REUSED`. Chỉ request
mới verify `READY_FOR_REVIEW`, zero active `ERROR`, matching active
`ValidationRun` và case version. Transaction thành công atomically lưu
idempotency key + canonical fingerprint + serialized `201` response, immutable
`ReviewDecision`, audit event và transition:

```json
{
  "data": {
    "review_decision_id": "dec_01J...",
    "case_id": "case_01J...",
    "decision": "APPROVED",
    "case_state": "APPROVED",
    "case_version": 13
  },
  "meta": {"request_id": "req_01J..."}
}
```

The same contract accepts human `REJECTED`. Technical failures never submit or
infer a `REJECTED` decision.

## Explicit state guards

| Action | Allowed case state | Additional guard | Failure |
|---|---|---|---|
| Initial upload | `DRAFT` | No conflicting active document version | `CASE_STATE_CONFLICT` |
| Additional new-slot upload | `DOCUMENTS_UPLOADED`, `PROCESSING`, `VALIDATION_REQUIRED`, `READY_FOR_REVIEW` | Slot absent, no replacement target, current `If-Match`; invalidate snapshot/pointers, bump version, return `PROCESSING` | `DOCUMENT_REPLACEMENT_CONFLICT`, state or version conflict |
| Replacement | `DOCUMENTS_UPLOADED`, `PROCESSING`, `VALIDATION_REQUIRED`, `READY_FOR_REVIEW` | Target is exact active document in same tenant/case/type; invalidate snapshot/pointers, bump version, return `PROCESSING` | `DOCUMENT_REPLACEMENT_CONFLICT`, state or version conflict |
| Reprocess | `DOCUMENTS_UPLOADED`, `PROCESSING`, `VALIDATION_REQUIRED`, `READY_FOR_REVIEW` | Active document; in `PROCESSING`, no `QUEUED`, `RUNNING` or `RETRY_PENDING` run; invalidate snapshot/pointers, bump version, return `PROCESSING` | `PROCESSING_RUN_ACTIVE` or version/state conflict |
| Read current fields/evidence/issues/chat | Any same-tenant state | Only active document/result/snapshot; otherwise pending/refusal | `RESOURCE_NOT_FOUND` or safe pending result |
| Review decision, new key | `READY_FOR_REVIEW` | Reviewer/Tenant Admin, active validation snapshot, zero `ERROR`, current version | `CASE_NOT_READY_FOR_REVIEW` or `409 Conflict` |
| Review decision, same key/fingerprint | Any current state after original commit | Auth/membership/role/resource tenant rechecked; replay precedes mutable guards | Replay original `201` response |
| Document mutation after decision | Never in `APPROVED` or `REJECTED` | Future reopen requires separate contract | `CASE_STATE_CONFLICT` |

Additional upload, replacement và reprocess atomically invalidate active
validation and affected result/index pointers, increment case version and move
the case to `PROCESSING`. Stale worker output remains audit-only.

## Stable error catalog

| HTTP | Code | Meaning |
|---|---|---|
| `401` | `AUTHENTICATION_REQUIRED` | Missing, invalid or expired authentication |
| `403` | `MEMBERSHIP_INACTIVE` | Identity valid but tenant membership inactive |
| `403` | `PERMISSION_DENIED` | Active role lacks the action |
| `404` | `RESOURCE_NOT_FOUND` | Missing or not visible in authenticated tenant |
| `400` | `INVALID_REQUEST` | Schema/field validation failed |
| `400` | `INVALID_CURSOR` | Cursor invalid or bound to another query/scope |
| `415` | `UNSUPPORTED_DOCUMENT_TYPE` | Not an approved English PDF/document type |
| `422` | `INVALID_PDF` | Corrupted, password-protected or structurally invalid PDF |
| `413` | `DOCUMENT_TOO_LARGE` | File exceeds 20 MB |
| `422` | `DOCUMENT_PAGE_LIMIT_EXCEEDED` | PDF exceeds 50 pages |
| `409` | `CASE_STATE_CONFLICT` | Action not allowed in current business state |
| `409` | `CASE_VERSION_CONFLICT` | Optimistic-lock version is stale |
| `409` | `MEMBERSHIP_VERSION_CONFLICT` | Membership `If-Match` version is stale |
| `409` | `CASE_NOT_READY_FOR_REVIEW` | State/snapshot/active `ERROR` blocks decision |
| `409` | `PROCESSING_RUN_ACTIVE` | Reprocess would create a competing run |
| `409` | `DOCUMENT_REPLACEMENT_CONFLICT` | New-slot/replacement target conflicts with active document slot/lineage |
| `409` | `IDEMPOTENCY_KEY_REUSED` | Same key used with different canonical request |
| `428` | `PRECONDITION_REQUIRED` | Required `If-Match` is missing |
| `429` | `RATE_LIMITED` | Server-side tenant/actor budget exceeded |
| `503` | `DEPENDENCY_UNAVAILABLE` | Required durable dependency is unavailable |
| `503` | `CHAT_GROUNDING_UNAVAILABLE` | Grounded chat could not complete safely |

Error precedence không được dựa trên resource existence: missing/invalid
identity nhận `401`; active-membership/role denial nhận `403` trước resource
lookup; sau khi các checks này pass, mọi cross-tenant resource, parent hoặc
referenced-resource mismatch luôn nhận `404 RESOURCE_NOT_FOUND`.

All unexpected `5xx` responses still use the envelope and request ID. They
must not claim upload, review or processing success unless the documented
durability transaction committed and the idempotent replay can return it.

## Liên quan

- [Authorization matrix](authorization-matrix.md)
- [Security model](../security/security-model.md)
- [Failure modes](../security/failure-modes.md)
- [State machines](../data-model/state-machines.md)
