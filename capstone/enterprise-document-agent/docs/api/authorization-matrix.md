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

Với idempotent mutation replay, bốn bước đầu vẫn chạy lại. Sau đó server so
idempotency key/fingerprint trước mutable state/version guard: same-fingerprint
record trả original response, còn chỉ first execution mới chạy bước 5. Replay
không là một business transition mới và không được thất bại chỉ vì first
execution đã đổi state/version.

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
| Đổi role/active membership | — | — | ✓ | Target cùng tenant; required membership `If-Match`; CAS version + change + audit atomic |
| Liệt kê/xem supplier | ✓ | ✓ | ✓ | Supplier cùng tenant |
| Tạo supplier | ✓ | ✓ | ✓ | Server gán tenant; idempotency key hợp lệ |
| Liệt kê/xem case | ✓ | ✓ | ✓ | Supplier/case cùng tenant |
| Tạo case | ✓ | ✓ | ✓ | Parent supplier cùng tenant; case mới luôn `DRAFT` |
| Liệt kê document/version | ✓ | ✓ | ✓ | Case/document cùng tenant; history không tự thành active result |
| Initial upload | ✓ | ✓ | ✓ | Case `DRAFT`; PDF/type/size/page valid; current case version |
| Additional new-slot upload | ✓ | ✓ | ✓ | Nonterminal state; slot type chưa có active document; invalidate snapshot/pointers, bump case version, về `PROCESSING` |
| Replacement | ✓ | ✓ | ✓ | Nonterminal state; target đúng active document cùng tenant/case/type; invalidate/bump/`PROCESSING` |
| Xem processing status | ✓ | ✓ | ✓ | Document/case cùng tenant; current lineage leaf only |
| Xem extracted fields | ✓ | ✓ | ✓ | Active document + active successful `ProcessingRun`; sensitive values masked |
| Xem evidence/PDF | ✓ | ✓ | ✓ | Active document/result; bank viewer chỉ redacted derivative/proxy, không signed original |
| Reprocess active document | ✓ | ✓ | ✓ | Nonterminal state; active document; no competing run; invalidate snapshot/pointers, bump version, về `PROCESSING` |
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
quản trị được phê duyệt và yêu cầu `If-Match` của membership version. Server dùng
compare-and-set để ghi role/active status, increment version và audit actor,
before/after role/status, target, request/trace ID atomically. Thiếu `If-Match`
trả `428 PRECONDITION_REQUIRED`; stale version trả
`409 MEMBERSHIP_VERSION_CONFLICT`. Reviewer không thừa hưởng quyền admin này.

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
- new-slot upload chỉ khi chưa có active document cho requested type và không có
  `replaces_document_id`;
- nếu type đã có active document, replacement phải gửi
  `replaces_document_id` trỏ chính xác record active cùng tenant/case/type;
  same-tenant target missing/stale/inactive/wrong case/type trả
  `DOCUMENT_REPLACEMENT_CONFLICT`, cross-tenant target trả
  `RESOURCE_NOT_FOUND`;
- reprocess cần active document; trong `PROCESSING` không được có run
  `QUEUED`, `RUNNING` hoặc `RETRY_PENDING` cho cùng document;
- `APPROVED`/`REJECTED`: mọi document mutation bị từ chối.

Unique constraint và case CAS bảo đảm tối đa một active version cho
`tenant_id + case_id + document_type` và ngăn sai lineage khi hai request đua.
Mọi additional new-slot upload, replacement và reprocess phải atomically
invalidate active validation, clear affected result/index pointers, tăng case
version rồi về `PROCESSING`, bất kể nonterminal state trước đó. Decision đồng
thời không thể dùng stale snapshot.

Queue message chỉ mang identifiers/context do API đã xác minh. Worker vẫn đối
chiếu `tenant_id + case_id + document_id + run_id` với database trước khi đọc
object hoặc ghi stage output; verified message không miễn kiểm tra resource
relationship.

### Fields, evidence, issues và chat

Mọi read chỉ surface active document version, active successful processing
result và active validation snapshot. Search bắt buộc filter
`tenant_id + case_id + active document` trước keyword lẫn vector query.
Evidence/signed URL chỉ được tạo sau authorization; URL possession không thay
application permission. Với `bank_information_form`, viewer chỉ nhận signed URL
cho redacted derivative hoặc authorization-enforcing proxy; original bank PDF là
internal-only. Future unmask cần explicit permission, step-up authentication và
immutable audit event, không tự đến từ Reviewer/Tenant Admin role.

Agent nhận auth context từ orchestrator, không nhận tenant override từ model.
Bảy tools `get_case_summary`, `list_case_documents`, `get_document_metadata`,
`get_extracted_fields`, `list_validation_issues`, `search_case_evidence` và
`explain_validation_issue` lặp lại tenant/resource guards và đều read-only.
Citation validator từ chối evidence không tồn tại, sai tenant/case, stale
document/result hoặc sai page/chunk/hash.

### Review decision

Chỉ Reviewer hoặc Tenant Admin được gọi decision endpoint. Trong một transaction
server phải:

1. xác minh identity, active membership và Reviewer/Tenant Admin role trước
   resource lookup;
2. load case bằng authenticated tenant predicate;
3. lookup idempotency record bằng tenant + actor + route + key và compare
   canonical fingerprint gồm path, payload và exact `If-Match`;
4. nếu same fingerprint đã commit, replay original `201` trước khi kiểm tra
   current case state/version; nếu khác fingerprint, trả
   `IDEMPOTENCY_KEY_REUSED`;
5. chỉ với request mới, xác minh case `READY_FOR_REVIEW`, compare `If-Match` với
   case version và xác minh request `validation_run_id` chính là active snapshot
   với zero `ERROR`;
6. reserve idempotency key và đảm bảo chưa có conflicting decision;
7. atomically ghi key, fingerprint, serialized response, immutable
   `ReviewDecision`, case transition và audit event.

Same-key/fingerprint retry trả original `201` kể cả case hiện đã
`APPROVED`/`REJECTED` hoặc version đã tăng. Nó vẫn phải pass identity,
membership, role và tenant-resource checks; replay không bypass revoked
membership/permission.

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

- `401` chỉ khi identity chưa xác thực; `403` chỉ khi active membership hoặc role
  permission bị từ chối độc lập với việc resource có tồn tại hay không;
  mọi cross-tenant resource/parent/reference mismatch luôn trả
  `404 RESOURCE_NOT_FOUND`;
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
