# Domain model

## Ngôn ngữ miền

Mô hình biểu diễn một tenant quản lý supplier onboarding cases. `Document` là
PDF có version; `ProcessingRun` tạo extracted fields/evidence; `ValidationRun`
đối chiếu snapshot active của case; con người ghi `ReviewDecision`. Business
state của case và technical processing status của run là hai khái niệm độc lập.

```mermaid
erDiagram
    TENANT ||--o{ MEMBERSHIP : has
    USER ||--o{ MEMBERSHIP : joins
    TENANT ||--o{ SUPPLIER : owns
    SUPPLIER ||--o{ SUPPLIER_CASE : has
    SUPPLIER_CASE ||--o{ DOCUMENT : contains
    DOCUMENT ||--o{ PROCESSING_RUN : processed_by
    PROCESSING_RUN ||--o{ EXTRACTED_FIELD : produces
    EXTRACTED_FIELD ||--o{ EVIDENCE : supported_by
    SUPPLIER_CASE ||--o{ VALIDATION_RUN : validates
    VALIDATION_RUN ||--o{ VALIDATION_ISSUE : reports
    VALIDATION_ISSUE ||--o{ EVIDENCE : supported_by
    SUPPLIER_CASE ||--o{ REVIEW_DECISION : receives
    TENANT ||--o{ AUDIT_EVENT : records
```

## Entities

### `Tenant`

Ranh giới isolation cao nhất. `tenant_id` do hệ thống sinh và là thành phần bắt
buộc của mọi business record, object path, queue/search/cache/log context và
Agent tool call. Xóa/đổi tenant không được suy ra từ payload của người dùng.

### `User`

Identity toàn cục từ `IdentityProvider`. `User` không tự mang quyền trên tenant;
quyền hiện hành đến từ active `Membership`.

### `Membership`

Liên kết `User` với `Tenant`, gồm role Operator, Reviewer hoặc Tenant Admin và
active status. Mỗi request dẫn xuất `tenant_id`, user và role từ authenticated
membership; thay đổi membership/role được audit và áp dụng ở request kế tiếp.

### `Supplier`

Supplier thuộc đúng một `Tenant`, là aggregate parent của các `SupplierCase`.
Mọi lookup supplier phải có tenant predicate trước khi dùng supplier để tạo case.

### `SupplierCase`

Hồ sơ onboarding thuộc một `Supplier` cùng tenant. Case giữ business state:
`DRAFT`, `DOCUMENTS_UPLOADED`, `PROCESSING`, `VALIDATION_REQUIRED`,
`READY_FOR_REVIEW`, `APPROVED` hoặc `REJECTED`, cùng optimistic-lock version.
Case chứa năm document type và là authorization/retrieval scope chính.

### `Document`

Một version bất biến của PDF trong case. Thuộc một trong:
`company_profile`, `business_registration`, `tax_registration`,
`bank_information_form`, `quotation`. Metadata gồm server-generated object key,
SHA-256, size, page count, document version, active marker và timestamps.

Thay file tạo `Document` mới với version tăng đơn điệu; không ghi đè object hoặc
record cũ. Với mỗi `tenant_id + case_id + document_type`, tối đa một version là
active. Version cũ và lineage được giữ cho audit nhưng không tham gia retrieval,
validation hay Agent answer hiện hành.

### `ProcessingRun`

Một lần xử lý kỹ thuật có `run_id`, `tenant_id`, `case_id`, `document_id`,
`pipeline_version`, retry/reprocess lineage, checkpoint và status `QUEUED`,
`RUNNING`, `SUCCEEDED`, `RETRY_PENDING` hoặc `FAILED`.

Khóa idempotency logic là `document_id + pipeline_version`; nó không thay
`run_id`. Duplicate broker delivery của cùng job dùng cùng run/checkpoint và
không tạo output trùng. Mỗi retry/reprocess được audit bằng `ProcessingRun`
riêng, liên kết predecessor/root run. Chỉ một successful run cho document
version/pipeline được chọn là active result; việc chọn xảy ra atomically sau
checkpoint `COMPLETED`.

### `ExtractedField`

Field có schema field name, `raw_value`, `normalized_value`, confidence,
critical/required metadata, schema version và `processing_run_id`. Record thuộc
tenant/case/document rõ ràng và không tồn tại nếu thiếu evidence cho field có
value. Account number được mã hóa at rest và masked ngoài trust boundary.

### `Evidence`

Lineage bất biến từ field hoặc issue về document version, page, short evidence
snippet, source block/chunk, content hash và bounding region khi parser cung cấp.
Evidence luôn có `tenant_id`, `case_id`, `document_id` và processing/version
references; không được tái sử dụng cross-tenant/case hoặc cho non-active answer.

### `ValidationRun`

Một deterministic evaluation của case tại thời điểm xác định. Run pin active
document versions, active successful `ProcessingRun`s, rule-catalog/schema/
confidence configuration versions và evaluation timestamp. Cùng input/config
phải tạo cùng rule outcomes.

### `ValidationIssue`

Kết quả một deterministic rule, gồm stable rule ID/version, severity `ERROR`,
`WARNING` hoặc `INFO`, machine-readable outcome/message arguments và evidence.
`ERROR` chặn `READY_FOR_REVIEW`; Agent/LLM không được sửa severity hoặc trạng thái
issue.

### `ReviewDecision`

Quyết định immutable `APPROVED` hoặc `REJECTED` do Reviewer/Tenant Admin cùng
tenant tạo khi case ở `READY_FOR_REVIEW` và không còn `ERROR`. Command dùng
idempotency key và optimistic locking; conflict trả `409`, không ghi đè decision
khác. Decision tham chiếu case version, active `ValidationRun`, actor và time.

### `AuditEvent`

Tenant-scoped append-only record cho membership, document/version, processing,
validation và review. Event lưu actor/action/resource/time/correlation cùng
lineage identifiers, không dùng full text hoặc sensitive value thay cho audit.

## Aggregate và ownership

```mermaid
flowchart TD
    T[Tenant boundary]
    T --> M[Membership]
    T --> S[Supplier]
    S --> C[SupplierCase aggregate]
    C --> D[Document versions]
    D --> P[ProcessingRun lineage]
    P --> F[ExtractedField + Evidence]
    C --> V[ValidationRun + ValidationIssue]
    C --> R[ReviewDecision]
    T --> A[AuditEvent]
```

- `SupplierCase` sở hữu business workflow; `ProcessingRun` không được tự quyết
  định case state.
- `Document` sở hữu version lineage; output sở hữu bởi run tạo ra nó nhưng chỉ
  active successful result được workflow đọc.
- `ValidationRun` là snapshot, không silently đổi khi document active thay đổi;
  thay đổi input cần run mới.
- `ReviewDecision` tham chiếu snapshot đã review để truy vết decision → issue →
  evidence → document version → processing run.

## Tenant invariants

1. Mọi business table có `tenant_id`; child tenant phải bằng parent tenant trên
   toàn bộ relationship.
2. Server derive tenant từ active membership và kiểm tra resource ownership;
   `tenant_id` trong body, model output hoặc tool argument không là nguồn quyền.
3. Database query/update/delete luôn có tenant predicate; foreign key/composite
   constraint ngăn liên kết cross-tenant khi khả thi.
4. Object key, queue message, search filter, cache key, structured log, trace và
   Agent tool context đều tenant-scoped.
5. Signed URL chỉ tạo sau application-layer authorization, thời hạn ngắn và chỉ
   cho object thuộc tenant/case hiện hành.
6. Không leak sự tồn tại của cross-tenant resource qua error, count, search,
   citation hoặc audit API.

## Versioning và uniqueness invariants

- Document version tăng trong scope
  `tenant_id + case_id + document_type`; record/object cũ bất biến.
- Chỉ một active `Document` cho mỗi document type trong case; chuyển active là
  transaction có audit.
- Idempotency output scope là `document_id + pipeline_version`; stage writes dùng
  unique/upsert constraints để redelivery không nhân bản.
- Mỗi retry/reprocess có `ProcessingRun` riêng; duplicate delivery không phải
  retry và không tạo run mới.
- Chỉ `ProcessingRun` đạt `SUCCEEDED` sau `COMPLETED` mới có thể active. Switching
  active result không xóa run trước.
- Extraction schema, normalization, confidence, rules, chunks và embeddings đều
  mang version để stale result không trộn với active result.
- `ValidationRun` pin toàn bộ input versions; review pin validation/case version.

## Write ownership invariants

- API use case ghi supplier/case/document/outbox/review sau authorization.
- Worker use case ghi run/checkpoint/extraction/evidence/index projection.
- Deterministic rule/workflow ghi validation issue và case transition.
- Agent tools là read-only; LLM không ghi entity, resolve issue, đổi severity,
  đổi active version hoặc approve/reject.
- Chỉ Reviewer/Tenant Admin có thể tạo `ReviewDecision`; Operator không có action
  này dù client cố gửi request trực tiếp.

## Durability và recovery

PDF gốc trong object storage và PostgreSQL metadata là recovery sources. Outbox
row commit cùng `Document`; RabbitMQ và search index có thể tái tạo. Không xóa
document version, run, validation evidence hoặc decision cần cho audit lineage.
