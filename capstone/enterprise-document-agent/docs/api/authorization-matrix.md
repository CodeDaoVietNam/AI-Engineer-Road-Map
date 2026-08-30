# Authorization matrix

## Nguyên tắc bắt buộc

RBAC chỉ là một lớp của authorization, không phải toàn bộ quyết định. Mỗi API
request, background command được khởi tạo từ API và Agent tool call phải xác minh
server-side đủ năm điều kiện sau:

1. authenticated identity hợp lệ;
2. active tenant membership tại thời điểm request;
3. role permission cho action;
4. resource, parent và referenced resources cùng tenant với membership;
5. resource state/version thỏa guard của action.

Server derive `user_id`, `tenant_id` và role từ identity + active `Membership`.
Client-supplied tenant identity trong body, path, query, header, filename, model
output hoặc tool argument không bao giờ authoritative. `tenant_id` chỉ xuất hiện
trong internal context/server-created event; không endpoint nào cho client chọn
tenant để mở rộng scope.

Lookup phải kết hợp tenant predicate ngay từ query đầu tiên. ID hợp lệ nhưng
thuộc tenant khác trả `RESOURCE_NOT_FOUND`, không trả metadata, count, state,
signed URL hoặc error khác có thể xác nhận sự tồn tại.

## Role inheritance

- **Operator** tạo supplier/case, upload/thay/reprocess document và đọc kết quả.
- **Reviewer** có toàn bộ quyền Operator, thêm review workspace và quyết định
  `APPROVED`/`REJECTED`.
- **Tenant Admin** có toàn bộ quyền Reviewer, thêm quản lý membership và xem
  tenant audit events.

Frontend có thể ẩn action không được phép nhưng đó chỉ là UX. API luôn thực thi
matrix kể cả client gọi endpoint trực tiếp.

## Action-by-role matrix

Ký hiệu: `✓` được phép sau khi mọi server guard đạt; `—` luôn bị từ chối.

| Resource/action | Operator | Reviewer | Tenant Admin | Server-side resource/state guard |
|---|:---:|:---:|:---:|---|
| Đăng nhập, đọc `auth/me` | ✓ | ✓ | ✓ | Token hợp lệ; membership hiện hành active |
| Liệt kê membership | — | — | ✓ | Chỉ membership trong authenticated tenant |
| Đổi role/active membership | — | — | ✓ | Target cùng tenant; change được audit và áp dụng request kế tiếp |
| Liệt kê/xem supplier | ✓ | ✓ | ✓ | Supplier cùng tenant |
| Tạo supplier | ✓ | ✓ | ✓ | Server gán tenant; idempotency key hợp lệ |
| Liệt kê/xem case | ✓ | ✓ | ✓ | Supplier/case cùng tenant |
| Tạo case | ✓ | ✓ | ✓ | Parent supplier cùng tenant; case mới luôn `DRAFT` |
| Liệt kê document/version | ✓ | ✓ | ✓ | Case/document cùng tenant; history không tự thành active result |
| Initial upload | ✓ | ✓ | ✓ | Case `DRAFT`; PDF/type/size/page valid; current case version |
| Additional upload/replacement | ✓ | ✓ | ✓ | `DOCUMENTS_UPLOADED`, `PROCESSING`, `VALIDATION_REQUIRED` hoặc `READY_FOR_REVIEW`; replacement target active/same case; optimistic lock |
| Xem processing status | ✓ | ✓ | ✓ | Document/case cùng tenant; current lineage leaf only |
| Xem extracted fields | ✓ | ✓ | ✓ | Active document + active successful `ProcessingRun`; sensitive values masked |
| Xem evidence/PDF | ✓ | ✓ | ✓ | Active document/result, application-layer authorization trước signed URL |
| Reprocess active document | ✓ | ✓ | ✓ | Nonterminal state; active document; current version; không competing nonterminal run trong `PROCESSING` |
| Xem validation issues | ✓ | ✓ | ✓ | Active `ValidationRun`; invalidated snapshot không surface như current |
| Dùng grounded chat | ✓ | ✓ | ✓ | Case cùng tenant; registered read-only tools; active evidence/citation only |
| Xem review decision workspace/history | — | ✓ | ✓ | Case/snapshot cùng tenant; mọi value nhạy cảm masked |
| Approve case | — | ✓ | ✓ | `READY_FOR_REVIEW`, zero `ERROR`, active `ValidationRun`, current case version, idempotency key |
| Reject case | — | ✓ | ✓ | Cùng guards với approve; đây là human business decision |
| Xem audit events | — | — | ✓ | Tenant-scoped cursor/filter; sensitive values redacted |
| Gọi Agent write/state/review tool | — | — | — | Không có tool như vậy; deterministic workflows sở hữu writes/transitions |
| Upload/reprocess terminal case | — | — | — | `APPROVED`/`REJECTED` là terminal; future reopen cần contract riêng |
| Cross-tenant read/write/search/review | — | — | — | Bị từ chối trước action, không leak existence |

## Authorization paths theo nhóm endpoint

### Authentication và membership

`auth/me` không tin tenant selector từ client. Nếu một identity có nhiều
membership trong phiên bản tương lai, tenant switch phải là một explicit,
server-validated membership selection contract; MVP không suy ra từ body tùy ý.

Tenant Admin có thể quản lý role/active status trong tenant mình. Use case phải
đọc lại membership target bằng `tenant_id + membership_id`, kiểm tra invariant
quản trị được phê duyệt, dùng optimistic locking và ghi audit actor, before/after
role/status, target, request/trace ID. Reviewer không thừa hưởng quyền admin này.

### Supplier và case

Tạo case phải load parent bằng authenticated `tenant_id + supplier_id`. Tạo
resource ghi `tenant_id` từ server context và không map field cùng tên nếu client
gửi. List/detail/filter/count đều có tenant predicate; Agent không được dùng một
supplier/case ID để bypass query này.

Case business state chỉ thay đổi theo state machine. API user action có thể khởi
tạo deterministic workflow, nhưng client, role và Agent không được gửi arbitrary
target state.

### Document upload, replacement và reprocess

Trước object write, API xác minh identity, active membership, permission,
supplier/case tenant và case state/version. Server tạo storage key; filename và
metadata PDF không chọn tenant/path. Guards chính xác:

- initial upload: `DRAFT`;
- additional upload/replacement/reprocess: `DOCUMENTS_UPLOADED`, `PROCESSING`,
  `VALIDATION_REQUIRED`, `READY_FOR_REVIEW`;
- reprocess cần active document; trong `PROCESSING` không được có run
  `QUEUED`, `RUNNING` hoặc `RETRY_PENDING` cho cùng document;
- `APPROVED`/`REJECTED`: mọi document mutation bị từ chối.

Replacement/reprocess từ `VALIDATION_REQUIRED` hoặc `READY_FOR_REVIEW` phải
atomically invalidate active validation và affected result/index pointers, tăng
case version rồi về `PROCESSING`. Decision đồng thời không thể dùng stale
snapshot.

Queue message chỉ mang identifiers/context do API đã xác minh. Worker vẫn đối
chiếu `tenant_id + case_id + document_id + run_id` với database trước khi đọc
object hoặc ghi stage output; verified message không miễn kiểm tra resource
relationship.

### Fields, evidence, issues và chat

Mọi read chỉ surface active document version, active successful processing
result và active validation snapshot. Search bắt buộc filter
`tenant_id + case_id + active document` trước keyword lẫn vector query.
Evidence/signed URL chỉ được tạo sau authorization; URL possession không thay
application permission.

Agent nhận auth context từ orchestrator, không nhận tenant override từ model.
Bảy tools `get_case_summary`, `list_case_documents`, `get_document_metadata`,
`get_extracted_fields`, `list_validation_issues`, `search_case_evidence` và
`explain_validation_issue` lặp lại tenant/resource guards và đều read-only.
Citation validator từ chối evidence không tồn tại, sai tenant/case, stale
document/result hoặc sai page/chunk/hash.

### Review decision

Chỉ Reviewer hoặc Tenant Admin được gọi decision endpoint. Trong một transaction
server phải:

1. load case bằng authenticated tenant predicate;
2. xác minh role, active membership và case `READY_FOR_REVIEW`;
3. compare `If-Match` với case version;
4. xác minh request `validation_run_id` chính là active snapshot và zero
   `ERROR`;
5. reserve idempotency key và đảm bảo chưa có conflicting decision;
6. ghi immutable `ReviewDecision`, case transition và audit event atomically.

Stale/concurrent request trả `409 Conflict`; request chưa đạt review gate trả
`CASE_NOT_READY_FOR_REVIEW`. Operator luôn nhận `PERMISSION_DENIED`, kể cả case
đã sẵn sàng. Technical `FAILED`, queue/parser/model error hoặc Agent output không
bao giờ được chuyển thành business `REJECTED`.

### Audit

Tenant Admin chỉ xem audit của tenant authenticated. Audit query, cursor,
resource filters và export trong tương lai đều tenant-bound. Audit event là
append-only, redacted và không thay log bằng full document/bank data. Privileged
membership/review/reprocess/signed-access actions phải có actor, action,
resource, result, time, request/trace và lineage identifiers.

## Denial và audit policy

- `401` khi chưa xác thực; `403` khi membership inactive hoặc permission thiếu;
  `404 RESOURCE_NOT_FOUND` cho resource missing/cross-tenant;
  `409 Conflict` cho state/version/idempotency/concurrency conflict.
- Authorization failure xảy ra trước signed URL, model call, object read, queue
  publish hoặc write side effect.
- Ghi telemetry cho denied action bằng hashed/opaque identifiers và safe reason;
  không log token, full payload, document text hoặc bank account.
- Cross-tenant attempt tăng security metric và alert theo threshold; response
  vẫn không xác nhận resource tồn tại.

## Liên quan

- [API contract](api-contract.md)
- [Security model](../security/security-model.md)
- [State machines](../data-model/state-machines.md)
