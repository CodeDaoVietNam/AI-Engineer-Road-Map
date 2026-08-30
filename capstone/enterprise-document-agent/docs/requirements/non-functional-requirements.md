# Yêu cầu phi chức năng

## Quy ước đo lường

Các target là design target MVP. P95 được đo ở server với workload đại diện; upload acknowledgement không tính thời gian truyền file từ client. Evaluation dùng synthetic dataset tách theo supplier để tránh leakage. Mỗi ID phải liên kết test, dashboard hoặc evaluation report trước khi milestone liên quan được nghiệm thu.

## PERF — Performance

### PERF-001 — Upload acknowledgement

API phải trả upload acknowledgement P95 dưới **500 ms**, đo từ khi server nhận byte cuối của PDF đến khi server hoàn tất tạo response `202 Accepted` với `document_id` và `status_url`. Khoảng thời gian client truyền các byte PDF đến server bị loại trừ; timer vẫn bao gồm server validation, object-storage finalization/durability confirmation, SHA-256, database và outbox commit, cùng việc tạo response.

### PERF-002 — Document processing

Document processing P95 thành công phải dưới **hai phút**, tính từ khi `DocumentUploaded` được published cho đến khi `ProcessingRun` đạt `SUCCEEDED`. SLI này chỉ dùng successful runs; retry và terminal technical failure không được đưa vào mẫu để failure nhanh không cải thiện success latency.

### PERF-003 — Retrieval và chat

Retrieval P95 phải dưới **một giây**. Chat không streaming P95 phải dưới **10 giây**, gồm retrieval, tool execution và citation validation.

### PERF-004 — Capacity

Thiết kế phải chịu 10 tenants, 100 users mỗi tenant, 1.000 cases mỗi tenant, trung bình năm documents mỗi case, khoảng 1.000 uploads/ngày, peak 20 uploads/phút và 50 chat requests đồng thời mà không bỏ tenant boundary hoặc hạ latency class.

### PERF-005 — Failure-detection latency

Latency phát hiện failure phải được đo và báo cáo riêng, tính từ `DocumentUploaded` được published đến `FAILED` hoặc dead-letter queue. Metric này không có quyền thay thế hoặc làm thay đổi PERF-002; report phải tách count/retry class của successful và failed runs.

## AVAIL — Availability

### AVAIL-001 — Availability target

Availability target là **99,5%** trong mỗi rolling 30-day window cho các API flows được công bố: auth/me; suppliers; supplier cases; document upload acknowledgement, status, evidence và reprocess; extracted fields; validation issues; case chat; review decisions; audit events. SLI bằng số request hợp lệ hoàn thành trước gateway timeout với response nghiệp vụ hợp lệ (bao gồm intended authorization/business `4xx` và `409`) chia cho tổng request hợp lệ của các flows đó; `5xx` và timeout là unavailable. Planned maintenance và dependency failure vẫn được tính vào denominator và là unavailable khi gây `5xx`/timeout; chúng chỉ được ghi chú riêng, không bị loại khỏi SLI. Dependency không khả dụng phải trả error code ổn định, không báo sai processing/review thành công.

### AVAIL-002 — Failure isolation

Agent failure không được làm hỏng upload, extraction hoặc review. Một job lỗi không được dừng worker xử lý job khác; status lỗi phải quan sát được.

## CONS — Consistency và durability

### CONS-001 — Durable acknowledged upload

Không mất document sau khi API acknowledge. `Document` và `DocumentUploaded` outbox event phải cùng transaction; queue là delivery mechanism, không phải source of truth.

### CONS-002 — Idempotency/versioning

Redelivery/retry không tạo extraction hoặc chunk trùng trong scope
`run_id + checkpoint/stage`. `document_id + pipeline_version` giữ lineage và
active-result comparison, không suppress một reprocess mới; version document
cũ không bị ghi đè, mỗi retry/reprocess có `ProcessingRun` riêng và chỉ một
successful run là active.

### CONS-003 — Review concurrency

Review dùng idempotency key và optimistic locking. Concurrent conflict phải trả `409`, không ghi đè decision đã tồn tại.

## SEC — Security

### SEC-001 — Tenant isolation

Tenant isolation áp dụng database, object path, queue message, search filter, cache key, logs và Agent tools. Tenant-isolation security tests phải đạt **100%**; mọi cross-tenant access bị từ chối.

### SEC-002 — Authorization

Server kiểm tra authenticated user, active membership, role permission, resource tenant và resource state. API không dùng `tenant_id` từ body làm nguồn quyền; chỉ Reviewer/Tenant Admin approve/reject.

### SEC-003 — Upload và Agent boundary

Storage key do server tạo; signed URL có thời hạn ngắn và authorization được kiểm tra ở application layer. PDF là untrusted data; Agent chỉ có registered read-only tools, với step, timeout, token và cost budget server-side.

## PRIV — Privacy

### PRIV-001 — Bank-data protection

Bank account phải mã hóa khi lưu và masked khi hiển thị. Full bank-account value không xuất hiện trong logs, traces, fixtures hoặc generated reports.

### PRIV-002 — Sensitive-data hygiene

Không log secrets, tokens, credentials, full document text hoặc sensitive prompt. Committed configuration chỉ có non-secret names/safe defaults; customer documents không được commit.

## AUDIT — Auditability

### AUDIT-001 — Audit trail

Audit tenant-scoped phải bao phủ membership, document/version, processing run, validation và review decision. Decision phải truy vết được về evidence/document version; Tenant Admin chỉ xem audit tenant mình.

### AUDIT-002 — Citation traceability

Citation phải có document, page, evidence snippet và bounding region khi có. Backend chỉ chấp nhận citation tồn tại, thuộc tenant/case đúng, trỏ active document và snippet đúng page/chunk.

## REC — Recoverability

### REC-001 — Recovery source

PDF gốc trong object storage và metadata PostgreSQL là nguồn khôi phục chính. Search index phải rebuild được từ hai nguồn; RabbitMQ queue không phải source of truth.

### REC-002 — Retry classification

Timeout, rate limit, service unavailable và lỗi kết nối retry tối đa ba lần với exponential backoff + jitter. Permanent PDF failure hoặc schema vẫn sai sau controlled repair không retry vô hạn, chuyển dead-letter queue khi vượt giới hạn.

## OPS — Operability

### OPS-001 — Structured telemetry

Structured log phải có request/trace/tenant/case/document/run identifiers, event, duration và error code, đồng thời tuân thủ PRIV-002. Trace ingestion nối API → outbox → queue → worker → external AI service.

### OPS-002 — Operational metrics

Theo dõi API latency, queue depth/job age, processing stages, retry rate, retrieval zero-result, tokens/cost, Agent steps, case states và unauthorized attempts.

## MAINT — Maintainability

### MAINT-001 — Code boundaries

`packages/core` framework-independent, không import FastAPI, SQLAlchemy hay Azure SDK; `packages/contracts` sở hữu schemas/events/ports; `packages/adapters` có local/Azure implementations. API/worker dùng chung core, không nhân bản domain logic.

### MAINT-002 — Testability

Testing gồm unit, integration với services thật trong container, API contract, React component, end-to-end, security, failure và performance tests. Deterministic rule engine dùng fixtures xác định và đạt **100%** deterministic test cases.

## PORT — Portability

### PORT-001 — Local-first

Local stack dùng React/Vite, FastAPI, PostgreSQL 16 + pgvector, MinIO, RabbitMQ, PyMuPDF, local JWT và model provider adapters. Azure Foundry không là dependency bắt buộc để chạy local; deterministic tests dùng mock/fake model adapters.

### PORT-002 — Azure boundary

Azure deployment chỉ bắt đầu sau local MVP có evaluation baseline. Adapter mapping là Container Apps, Azure Database for PostgreSQL, Blob Storage, Service Bus, Document Intelligence, AI Search, Microsoft Foundry/Azure OpenAI, Entra ID, Key Vault, Azure Monitor/Application Insights và Azure Container Registry; business logic không import Azure SDK.

## COST — Cost và quality budget

### COST-001 — Agent budget

Agent phải có step, timeout, token và cost budget enforce server-side. Telemetry phân bổ tokens/cost theo request, tenant/case/document/run khi phù hợp để phát hiện vượt budget.

### COST-002 — Quality gates

Evaluation báo cáo từng tầng với document classification macro F1 ≥ **0,90**; field extraction F1 ≥ **0,85**; critical-field recall ≥ **0,95**; Retrieval Recall@5 ≥ **0,90**; citation correctness ≥ **0,90**; answer faithfulness ≥ **0,90**; correct refusal ≥ **0,85**; Agent tool-selection accuracy ≥ **0,90**. Dataset khoảng 20 suppliers × 5 document types tách development, validation, final test theo supplier.

## Liên quan

- [Yêu cầu sản phẩm](product-requirements.md)
- [Yêu cầu chức năng](functional-requirements.md)
