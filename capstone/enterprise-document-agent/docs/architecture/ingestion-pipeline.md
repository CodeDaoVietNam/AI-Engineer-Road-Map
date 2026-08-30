# Thiết kế ingestion pipeline

## Mục tiêu và bất biến

Ingestion tiếp nhận PDF tiếng Anh của năm document type, xác nhận bền vững trước
khi trả response và xử lý nền an toàn dưới at-least-once delivery. Các bất biến:

- mỗi file tối đa **20 MB** và **50 pages**;
- API không acknowledge nếu object chưa durable hoặc database/outbox chưa commit;
- `Document` và `DocumentUploaded` outbox event được tạo trong cùng database
  transaction;
- message chỉ chứa verified identifiers, không chứa PDF binary;
- worker idempotent theo `document_id + pipeline_version`;
- redelivery/retry không tạo extracted field, evidence hoặc chunk trùng;
- technical failure chỉ đổi processing status, không tự đổi business state thành
  `REJECTED`.

## Upload và `202 Accepted`

```mermaid
sequenceDiagram
    actor U as Authenticated user
    participant API as FastAPI API
    participant OS as ObjectStorage
    participant DB as PostgreSQL
    participant OB as Outbox publisher
    participant MQ as RabbitMQ

    U->>API: Upload PDF + case/document type
    API->>API: Verify identity, active membership, role
    API->>DB: Verify tenant ownership + case state
    API->>API: Validate type, 20 MB, PDF, 50 pages
    API->>OS: Stream to server-generated object key + SHA-256
    OS-->>API: Durability confirmation
    API->>DB: BEGIN
    API->>DB: Insert Document + DocumentUploaded outbox event
    API->>DB: COMMIT
    DB-->>API: Durable commit
    API-->>U: 202 Accepted {document_id, status_url}
    OB->>DB: Claim unpublished outbox row
    OB->>MQ: Publish verified identifiers
    MQ-->>OB: Publisher confirm
    OB->>DB: Mark outbox row published
```

### Kiểm tra trước write

API thực hiện theo thứ tự fail-fast:

1. Xác minh token, authenticated user và active membership; dẫn xuất
   `tenant_id`, role và user từ server-side context.
2. Kiểm tra supplier/case/document thuộc cùng tenant, role được upload và case
   state cho phép nhận hoặc thay document.
3. Chỉ chấp nhận `company_profile`, `business_registration`, `tax_registration`,
   `bank_information_form` hoặc `quotation`.
4. Giới hạn stream ở 20 MB; kiểm tra PDF signature/content thay vì tin MIME hay
   filename từ client.
5. Parse cấu trúc đủ để từ chối corrupted, password-protected, invalid PDF và
   file vượt 50 pages. File ngoài phạm vi PDF tiếng Anh nhận lỗi ổn định, không
   tạo job ingestion.
6. Sinh object key ở server với tenant/case/document identifiers; không dùng
   filename client làm path.

API stream một lượt có giới hạn, đồng thời tính SHA-256. Chỉ sau durability
confirmation của object storage mới mở/hoàn tất database transaction tạo
`Document` và outbox row. Nếu object write lỗi, không tạo record. Nếu database
transaction lỗi sau object write, API không trả `202`; orphan object không được
tham chiếu và được một cleanup job tenant-safe loại bỏ sau retention window.

Response `202 Accepted` chỉ gồm `document_id` và `status_url` cần để polling;
không hứa processing đã hoàn tất. P95 acknowledgement dưới **500 ms** được đo từ
byte cuối server nhận đến khi tạo response, bao gồm validation, storage
finalization, SHA-256, database/outbox commit và response generation.

## Transactional outbox và RabbitMQ

`DocumentUploaded` dùng PascalCase và chứa tối thiểu `event_id`, `tenant_id`,
`case_id`, `document_id`, `document_version`, `pipeline_version`, object
identifier và trace identifiers đã được API xác minh. Event không chứa PDF
binary, full document text hoặc bank-account value.

Outbox publisher:

1. claim unpublished row bằng cơ chế chống hai publisher cùng sở hữu;
2. publish persistent message với publisher confirm;
3. chỉ đánh dấu published sau confirm;
4. retry publish khi chưa có confirm; duplicate publish vẫn an toàn vì consumer
   idempotent.

PostgreSQL outbox là source of truth cho event chưa phát. RabbitMQ dùng durable
queue và at-least-once delivery nhưng không phải source of truth hoặc recovery
source. PDF gốc trong object storage và metadata PostgreSQL phải đủ để rebuild
search index và reprocess.

Mỗi queue message giữ `tenant_id`, `case_id`, `document_id`, `run_id`,
`pipeline_version`, `event_id`, correlation/trace identifiers và retry metadata.
Worker không nhận tenant từ model hay PDF; nó đối chiếu identifiers với database
trước khi đọc object hoặc ghi kết quả.

## Idempotency, run và checkpoint

Khóa idempotency logic là **`document_id + pipeline_version`**. `ProcessingRun`
có `run_id` riêng để audit từng retry/reprocess, nhưng mọi write kết quả phải
được bảo vệ trong scope của khóa logic này và stage hiện hành. Duplicate
delivery của cùng job không tạo run/output mới; một explicit retry hoặc
reprocess tạo `ProcessingRun` riêng, liên kết run trước, rồi tái sử dụng kết quả
checkpoint hợp lệ hoặc ghi version kết quả mới có lineage.

```mermaid
flowchart LR
    FV[FILE_VALIDATED] --> P[PARSED]
    P --> C[CLASSIFIED]
    C --> E[EXTRACTED]
    E --> V[VALIDATED]
    V --> CH[CHUNKED]
    CH --> I[INDEXED]
    I --> CO[COMPLETED]
```

Mỗi checkpoint được commit atomically cùng output của stage và thông tin
`run_id`, `document_id`, `pipeline_version`. Consumer ack message chỉ sau khi
checkpoint/kết quả cần thiết đã commit. Khi message bị redeliver:

- nếu `COMPLETED`, worker không chạy model lại và trả kết quả idempotent;
- nếu checkpoint trung gian hợp lệ, worker tiếp tục từ stage kế tiếp;
- nếu output không khớp schema/pipeline version hiện hành, worker không dùng nó
  như checkpoint hợp lệ;
- unique/upsert constraints trong idempotency scope ngăn duplicate field,
  evidence và chunk.

Mỗi retry/reprocess có `ProcessingRun` riêng cho audit; chỉ một run
`SUCCEEDED` được chọn làm active result. Việc chọn active chỉ xảy ra atomically
sau `COMPLETED`, không làm mất run hoặc document version cũ.

## Phân loại lỗi và retry

### Transient failure

Các lỗi được retry tối đa **ba lần** là timeout, rate limit, service unavailable
và lỗi kết nối tạm thời. Workflow:

1. lưu error code an toàn và đánh dấu run `RETRY_PENDING`;
2. tính exponential backoff + jitter từ retry count trong server-controlled
   policy;
3. phát job retry có cùng idempotency scope, tenant context và checkpoint;
4. tạo lineage `ProcessingRun` cho lần retry nhưng không nhân bản output.

Retry budget là ba lần retry sau lần chạy đầu; không adapter nào được tự tạo
vòng retry vô hạn bên ngoài budget chung. Khi hết budget, run cuối là `FAILED`
và message được đưa vào dead-letter queue.

### Permanent failure

Không tự retry các lỗi: corrupted/password-protected/invalid PDF bị phát hiện
muộn, document type không hỗ trợ, invariant/authorization scope sai, hoặc schema
vẫn invalid sau một controlled repair trong extraction policy. Run chuyển
`FAILED`; message chuyển dead-letter queue với identifiers, error class và
lineage, không chứa nội dung nhạy cảm.

### Dead-letter và khôi phục

- Dead-letter queue tách theo message contract/version và có alert theo queue
  depth/job age.
- Operator không replay trực tiếp message tùy ý. Reprocess đi qua API, xác minh
  active membership, tenant ownership và case/document state, rồi tạo
  `ProcessingRun` mới.
- Replay giữ original event/run references và cùng `pipeline_version`, hoặc dùng
  pipeline version mới được triển khai rõ ràng; idempotency vẫn áp dụng.
- Dead-letter không đổi case thành `REJECTED`. UI hiển thị technical `FAILED`
  và hướng xử lý/reprocess riêng.

## Processing status và business state

Processing status độc lập với case business state:

```text
QUEUED → RUNNING → SUCCEEDED
                 ├→ RETRY_PENDING
                 └→ FAILED
```

Upload hợp lệ có thể đưa case từ `DRAFT` sang `DOCUMENTS_UPLOADED`; deterministic
workflow đưa case sang `PROCESSING` khi ingestion bắt đầu. Chỉ validation
workflow mới đánh giá `VALIDATION_REQUIRED` và `READY_FOR_REVIEW`. Lỗi kỹ thuật
giữ business case có thể quan sát/recover, không bao giờ được ánh xạ thành
`REJECTED`.

## Telemetry và mục tiêu vận hành

Structured telemetry nối API → outbox → RabbitMQ → worker → external AI service
với request/trace/tenant/case/document/run identifiers, stage, duration và error
code; không log full PDF text hay sensitive values. Metrics gồm outbox age,
queue depth/job age, checkpoint latency, retry count/class, dead-letter count và
processing stage.

Processing P95 dưới **hai phút** tính từ lúc `DocumentUploaded` được published
đến `ProcessingRun` đạt `SUCCEEDED`, chỉ trên successful runs. Failure-detection
latency từ publish đến `FAILED` hoặc dead-letter được báo riêng để failure nhanh
không làm đẹp success latency.
