# Failure modes và degraded operation

## Bất biến khi failure

Business state `SupplierCase` và technical `ProcessingRun` status được lưu riêng.
Timeout, dependency outage, parser/model/schema/index/chat/citation failure chỉ
tạo technical error/status, retry hoặc refusal. Technical failure **không bao
giờ** tạo business `REJECTED`; `REJECTED` chỉ đến từ immutable human
`ReviewDecision` của Reviewer hoặc Tenant Admin khi case ở
`READY_FOR_REVIEW`.

`202 Accepted` chỉ được trả sau object durability confirmation và transaction
`Document` + initial `ProcessingRun(QUEUED)` + `DocumentUploaded` commit. Sau
acknowledgement, PDF gốc trong object storage và PostgreSQL metadata là recovery
source of truth. Outbox là source of truth cho event chưa publish; RabbitMQ và
search index không phải source of truth.

Retry policy chung cho timeout, rate limit, service unavailable và lỗi kết nối
tạm thời là tối đa ba retries sau lần đầu, exponential backoff + jitter. Adapter
không được tạo nested/unbounded retry. Permanent input/invariant/schema failure
không retry vô hạn. Một job lỗi không chặn worker/job khác.

## Failure-mode matrix

| Failure | Retryability / isolation | User-visible result | Recovery / source of truth | Telemetry |
|---|---|---|---|---|
| Object storage write/finalize lỗi trước acknowledge | Retry bounded ở request/adaptor nếu còn latency budget; nếu chưa có durability confirmation thì fail request, không tạo DB/outbox | `503 DEPENDENCY_UNAVAILABLE`; không trả `202 Accepted`, không có document/status giả | Không có durable business record. Cleanup tenant-safe xóa multipart/orphan theo retention | storage operation latency/error class, orphan/multipart count, request/trace/tenant/case IDs |
| Object storage read lỗi sau acknowledge | Transient retry tối đa ba; isolate run thành `RETRY_PENDING`, rồi `FAILED` khi hết budget | Status hiển thị retry/technical `FAILED`; upload vẫn tồn tại; reprocess có thể dùng sau recovery | PDF object + PostgreSQL metadata; verify checksum/key/tenant rồi reprocess, restore object từ protected backup nếu cần | run/stage error, retry count, object read latency, checksum mismatch, job age |
| PostgreSQL transaction/availability lỗi | Transaction rollback toàn bộ; connection/deadlock transient retry bounded, invariant/permanent không retry mù | Upload/review mutation trả stable `503`/`409`; không báo thành công nếu commit chưa xác định. Idempotent replay xác định outcome | PostgreSQL WAL/backup là authoritative metadata; object orphan được reconciler cleanup; audit/review/outbox atomic transaction quyết định outcome | DB availability/latency, rollback/deadlock, uncertain-commit reconciliation, outbox age |
| Outbox publisher dừng hoặc broker confirm thiếu | Retry publish từ unpublished row; duplicate publish được consumer idempotency cô lập | Upload đã `202`; status có thể ở `QUEUED` lâu, không bị chuyển `REJECTED` | PostgreSQL outbox là source of truth; publisher reclaim row và republish `event_id + run_id` | unpublished count/oldest age, publish attempts, confirm timeout, dispatcher heartbeat |
| RabbitMQ/queue unavailable, message delay hoặc redelivery | Producer retry bounded; persistent message + publisher confirm; consumer xử lý at-least-once idempotent. Queue outage không làm mất DB/outbox | Upload acknowledgement đã commit vẫn hợp lệ; status `QUEUED`; UI nêu processing delayed | Republish từ outbox; re-drive authorized DLQ. Queue không phải source of truth | queue availability/depth, oldest job age, redelivery/DLQ count, publish/consume rate |
| Worker crash, lease expiry hoặc một job treo | Broker redelivery/lease recovery; resume checkpoint, same run cho duplicate delivery; một process/job không dừng toàn pool | Status `RUNNING` rồi retry/recovered hoặc `FAILED`; không duplicate fields/evidence/chunks | PostgreSQL run/checkpoint + object; stage output/checkpoint atomic; scheduler tạo child retry run khi thực sự retry | worker heartbeat, lease age, checkpoint duration, crash/restart, duplicate suppression |
| Parser/OCR timeout/service unavailable | Transient retry tối đa ba từ checkpoint; sandbox/resource limit cô lập file | `RETRY_PENDING`, sau budget là `FAILED`; validation tạo `PROCESSING_RESULT_UNAVAILABLE` `ERROR`, không `REJECTED` | Original PDF + document/run metadata; authorized reprocess sau adapter recovery | parser latency/timeout, memory/CPU limit, retry class, run/document IDs |
| Parser phát hiện corrupted, password-protected, invalid hoặc >50 pages muộn | Permanent; không model repair/retry vô hạn; dead-letter theo policy | Technical `FAILED` với safe stable code; reviewer/operator được hướng thay file | Original immutable upload + safe diagnostics; replacement tạo document version mới | permanent parser error by safe class, DLQ count, document type/page/size buckets |
| Extraction/embedding/language model timeout, rate limit hoặc unavailable | Transient retry tối đa ba trong shared budget; circuit breaker/bulkhead cô lập provider và job | Processing `RETRY_PENDING`/`FAILED`; chat trả refusal hoặc `503 CHAT_GROUNDING_UNAVAILABLE`; upload/review APIs khác vẫn hoạt động | Parsed/checkpoint data + PDF/PostgreSQL. Reprocess hoặc chat retry sau recovery; no model output is authoritative state | provider latency/status, rate-limit, retries, circuit state, token/cost, Agent timeout |
| Model trả malformed/unsafe/unsupported output | Schema-validate; controlled repair có giới hạn. Không execute output/tool text | Processing tiếp tục nếu repaired; nếu không thì technical `FAILED`; chat refusal | Versioned schema, raw safe diagnostic hash và original evidence; fix adapter/schema then authorized reprocess | schema validation/repair attempts, model/version, safe error code, refusal count |
| Extraction schema vẫn invalid sau controlled repair hoặc schema/version mismatch | Permanent cho run hiện hành; không retry vô hạn qua model; stale checkpoint bị bỏ | `FAILED`; validation có deterministic blocking `ERROR`; không `REJECTED` | Versioned schemas + pinned `pipeline_version` + PDF/PostgreSQL; deploy compatible version rồi reprocess có audit | schema/pipeline version, invalid field/type counts, repair budget exhausted, DLQ |
| Search index write/outage/stale projection | Retry stage bounded; index activation chỉ sau complete write. Không fallback unfiltered hoặc surface stale result | Processing có thể `RETRY_PENDING`/`FAILED`; chat refusal/`503`; document/fields/review workspace từ PostgreSQL vẫn khả dụng theo contract | Rebuild index từ PDF gốc + PostgreSQL metadata/active pointers; search index không phải source of truth | index latency/error, backlog, active-vs-index lineage mismatch, zero/filtered result rate |
| Hybrid retrieval/reranker failure | Reranker có thể dùng pinned hybrid order theo server policy; retrieval failure/zero result dẫn refusal, không bỏ tenant filter | Grounded chat trả refusal an toàn hoặc `503`; không hallucinated answer | Retry request sau dependency recovery; evidence/document remain in PostgreSQL/object storage | retrieval latency, candidate counts, zero-result, reranker fallback/refusal, filter scope |
| Chat orchestrator/tool/budget failure | Request-level retry only if idempotency semantics safe; step/timeout/token/cost limit fail closed; isolated from ingestion/review | `200` refusal với safe reason hoặc `503 CHAT_GROUNDING_UNAVAILABLE`; case không đổi state | Re-run chat from active evidence after recovery; chat draft/model output không là source of truth | Agent steps, tool errors, timeout, token/cost budget, refusals by reason, request/case IDs |
| Citation missing, invalid, stale, wrong page/chunk/hash hoặc insufficient evidence | Không retry bằng cách bỏ validator. Có thể reretrieve trong same bounded chat budget; cuối cùng refusal | `200` với `refused: true`, không unsupported claim/citation | Backend re-check active evidence/chunk from PostgreSQL/search projection; rebuild/reretrieve nếu stale | citation validation reason, stale/invalid rate, correct-refusal metric, document/result lineage |
| Concurrent review decisions hoặc review vs document-set mutation | Sau auth/membership/role/tenant checks, lookup key/fingerprint trước mutable state/version guards. Same key/fingerprint replays original `201`; new keys dùng case/snapshot CAS và một transaction thắng | Exact replay nhận original `201` dù case đã terminal/version đổi; distinct request thua nhận `409 Conflict` | PostgreSQL atomically lưu key, canonical fingerprint, serialized response, `ReviewDecision`, case version và audit event; reload cho request mới | conflict, exact replay/key-mismatch count, actor/case/If-Match, decision latency, audit correlation |
| Concurrent membership PATCH | Required membership `If-Match`; compare-and-set role/active/version + audit, không last-write-wins | Stale request nhận `409 MEMBERSHIP_VERSION_CONFLICT`; no partial role/status update | Committed PostgreSQL membership version và audit event; reload ETag before deliberate retry | membership CAS conflict, actor/target/version, denied role attempts, audit correlation |
| Cross-tenant ID, parent-child mismatch, cursor/search/cache/citation/tool attempt | Không retry/fallback. Deny before read/model/write/signed URL; repeated attempts là security signal | Luôn `404 RESOURCE_NOT_FOUND`, không `401`/`403` dựa trên resource existence và không partial data | Server auth context + tenant-scoped PostgreSQL relationships are authoritative; investigate audit/security logs | unauthorized/cross-tenant attempt count, safe route/resource class, actor/tenant/request/trace, alert threshold |

## State behavior khi processing thất bại

Technical status đi độc lập:

```text
QUEUED → RUNNING → SUCCEEDED
                 ├→ RETRY_PENDING
                 └→ FAILED
```

Một transient run ở `RETRY_PENDING` có child retry `ProcessingRun(QUEUED)`; run
cũ không mutate về `QUEUED`. Scheduler atomically chuyển
`current_processing_run_id` sang child. Duplicate delivery dùng cùng run và
checkpoint, không tạo retry run mới.

Khi current leaf của mọi active document đã upload đạt `SUCCEEDED` hoặc
`FAILED`, validation vẫn chạy, không chờ đủ năm loại. Active document không có
successful result tạo deterministic `PROCESSING_RESULT_UNAVAILABLE` `ERROR`;
required document type absent tạo `COMPLETENESS_DOCUMENT_SET` `ERROR`. Case vào
`VALIDATION_REQUIRED`, không mắc kẹt mãi ở `PROCESSING` và không trở thành
`REJECTED`.

Mọi document-set mutation sau initial upload — additional new-slot upload,
replacement hoặc authorized reprocess — atomically invalidates active validation
và affected result/index pointers, tạo run/outbox cần thiết, tăng case version và
đưa case về `PROCESSING`, kể cả khi state trước đã là `PROCESSING` hoặc pointer
đang null. New-slot chỉ khi document type chưa có active document; replacement
phải target exact active document cùng tenant/case/type. Unique slot invariant +
case CAS ngăn hai active versions và wrong lineage. Historical failed/successful
run, issue và evidence vẫn audit-only.

## Degraded-operation policy

| Capability unavailable | Capability còn an toàn | Capability phải fail closed |
|---|---|---|
| Queue/outbox publisher | Đọc supplier/case/document/status đã commit | Không claim processing đã bắt đầu/hoàn tất; alert queued age |
| Worker/parser/extraction model | Auth, case/document reads, existing review workspace | New processing stays queued/retrying/failed; no synthetic fields |
| Search/index/reranker | Direct metadata, active extracted fields/issues/evidence reads | Chat must refuse/503; never unfiltered or stale retrieval |
| Language model/Agent | Upload, processing, deterministic validation, review decision | Chat refuses/503; no effect on case state |
| Object storage read | Metadata/status/audit reads where no PDF content required | PDF/evidence content and reprocess fail; no cross-source substitution |
| PostgreSQL | Static health information only | All business reads/writes requiring authorization/state fail; no queue/object-only authorization |

Availability dependency failure trả stable error envelope + request ID. Planned
maintenance và dependency failure vẫn tính unavailable khi gây timeout/`5xx`;
không che giấu bằng false success. UI phân biệt `processing delayed`,
`RETRY_PENDING`, technical `FAILED`, validation `ERROR`, chat refusal và human
decision.

## Recovery runbooks và ownership

### Reconcile acknowledged work

1. Query PostgreSQL for old unpublished outbox rows, `QUEUED`/`RUNNING` leases,
   current retry leaves và active document pointers theo tenant-safe batches.
2. Republish unpublished event hoặc recover expired lease bằng stable event/run
   identifiers; không tạo initial run từ dispatcher/worker.
3. Verify object checksum/metadata before worker resumes.
4. Confirm checkpoint/output idempotency and active-result compare-and-set.
5. Audit operator/service action and retain correlation IDs.

### Rebuild search

Rebuild only from authorized PostgreSQL active document/result metadata and PDF
/derived content backed by original object. Write a versioned shadow projection,
verify tenant/case/document counts and lineage, then atomically activate. Never
use existing unfiltered index as recovery source.

### Dead-letter/reprocess

DLQ is diagnostic/delivery state, not recovery truth. Operator cannot arbitrary
replay a message. Reprocess uses API authorization, active membership,
resource/state/version guards and idempotency key, then creates a new
`ProcessingRun` with predecessor/root lineage. Permanent bad PDF needs a new
document version; schema/provider fix may use an explicitly deployed
`pipeline_version`.

### Database/object reconciliation

If object durable but DB transaction failed, no `202 Accepted` was issued and a
tenant-safe orphan cleanup removes it after retention. If DB/outbox committed,
the document is acknowledged and must not be deleted merely because queue/index
is absent. An uncertain response is resolved by idempotency-key + canonical
fingerprint lookup before mutable state/version guards; fingerprint gồm path,
payload, exact `If-Match` và upload SHA-256 khi áp dụng. Review commit atomically
stores decision/key/fingerprint/response, nên exact retry replays original `201`
thay vì conflict với state/version do chính first execution tạo ra.

## Telemetry và alerts tối thiểu

- API dependency errors/latency, `5xx`, timeout, idempotent replay và
  `409 Conflict` rate;
- outbox oldest-unpublished age, queue depth/oldest-job age, redelivery và DLQ;
- worker leases/heartbeat, checkpoint/stage latency, retries/error class,
  success latency và failure-detection latency báo riêng;
- parser/model/index availability, schema repair, retrieval zero result,
  citation invalid/refusal và token/cost budgets;
- case counts theo exact business state, processing counts theo exact technical
  status; dashboard không trộn `FAILED` với `REJECTED`;
- unauthorized/cross-tenant/signed-access attempts với alert threshold và safe
  identifiers, không log sensitive payload.

Alert phải trỏ runbook và request/trace/tenant/case/document/run lineage đủ để
điều tra mà không cần full document hoặc bank data.

## Liên quan

- [API contract](../api/api-contract.md)
- [Security model](security-model.md)
- [Ingestion pipeline](../architecture/ingestion-pipeline.md)
- [State machines](../data-model/state-machines.md)
