# Yêu cầu chức năng

## Quy ước

ID là ổn định để milestone, test và API contract tham chiếu. Mỗi requirement nêu `Actor`, `Hành vi`, `Ranh giới quyền` do server kiểm tra và `Kết quả quan sát được`. Mọi request phải có authenticated user và active membership; `tenant_id` trong request body không bao giờ là nguồn ủy quyền.

## AUTH — Authentication và membership

### AUTH-001 — Xác định identity

- **Actor:** Operator, Reviewer hoặc Tenant Admin.
- **Hành vi:** Đăng nhập và gọi `auth/me` để lấy identity, tenant và role hiện hành.
- **Ranh giới quyền:** Token hợp lệ và active membership là bắt buộc.
- **Kết quả quan sát được:** Request thành công chỉ trả context của chính user; token không hợp lệ hoặc membership inactive bị từ chối.

### AUTH-002 — Derive authorization context

- **Actor:** API.
- **Hành vi:** Dẫn xuất tenant, role và user từ authenticated membership cho từng request.
- **Ranh giới quyền:** Không tin `tenant_id`, role hay user identifier do client gửi.
- **Kết quả quan sát được:** Giả mạo context trong payload không mở rộng quyền.

## TEN — Tenant isolation

### TEN-001 — Scope resource

- **Actor:** Authenticated user.
- **Hành vi:** Đọc, tạo, cập nhật, search hoặc review resource trong tenant hiện hành.
- **Ranh giới quyền:** Server kiểm tra resource `tenant_id` khớp context trước action.
- **Kết quả quan sát được:** Cross-tenant access bị từ chối qua ID, URL, queue/search context và Agent tool.

### TEN-002 — Manage membership

- **Actor:** Tenant Admin.
- **Hành vi:** Quản lý membership và xem audit của tenant.
- **Ranh giới quyền:** Chỉ Tenant Admin cùng tenant được thay đổi membership.
- **Kết quả quan sát được:** Membership/role thay đổi được audit và áp dụng ở request kế tiếp.

## SUP — Supplier

### SUP-001 — Create supplier

- **Actor:** Operator, Reviewer hoặc Tenant Admin.
- **Hành vi:** Tạo supplier thuộc tenant hiện hành.
- **Ranh giới quyền:** Cần active membership; server gán `tenant_id` từ context.
- **Kết quả quan sát được:** Supplier chỉ xuất hiện trong danh sách của tenant sở hữu.

### SUP-002 — View supplier

- **Actor:** Operator, Reviewer hoặc Tenant Admin.
- **Hành vi:** Liệt kê và xem supplier trong tenant.
- **Ranh giới quyền:** Chỉ resource cùng tenant và active membership được đọc.
- **Kết quả quan sát được:** Kết quả không chứa supplier của tenant khác.

## CASE — Supplier case

### CASE-001 — Create case

- **Actor:** Operator, Reviewer hoặc Tenant Admin.
- **Hành vi:** Tạo `SupplierCase` cho supplier cùng tenant ở state `DRAFT`.
- **Ranh giới quyền:** Server xác minh ownership supplier và membership.
- **Kết quả quan sát được:** Case mới có tenant/supplier đúng và state `DRAFT`.

### CASE-002 — State transition

- **Actor:** Deterministic workflow.
- **Hành vi:** Chuyển case theo `DRAFT → DOCUMENTS_UPLOADED → PROCESSING → VALIDATION_REQUIRED → READY_FOR_REVIEW → APPROVED` hoặc `REJECTED`.
- **Ranh giới quyền:** Client/Agent không tự chuyển state; transition cần state và validation phù hợp.
- **Kết quả quan sát được:** State hợp lệ được lưu/audit; technical failure không biến case thành `REJECTED`.

## DOC — Document lifecycle

### DOC-001 — Accept document type

- **Actor:** Operator, Reviewer hoặc Tenant Admin.
- **Hành vi:** Upload `company_profile`, `business_registration`, `tax_registration`, `bank_information_form` hoặc `quotation`.
- **Ranh giới quyền:** Cần quyền trên case cùng tenant; MVP chỉ nhận PDF tiếng Anh.
- **Kết quả quan sát được:** Metadata ghi document type hợp lệ; loại ngoài danh sách bị từ chối rõ ràng.

### DOC-002 — Validate và version document

- **Actor:** API.
- **Hành vi:** Kiểm tra PDF tối đa 20 MB, 50 pages; thay document tạo version mới thay vì ghi đè.
- **Ranh giới quyền:** Server tạo storage key, kiểm tra case ownership/state và không tin filename client.
- **Kết quả quan sát được:** Corrupted, password-protected hoặc invalid PDF bị từ chối; version trước giữ cho audit, chỉ active version được dùng.

### DOC-003 — View status/evidence

- **Actor:** Operator, Reviewer hoặc Tenant Admin.
- **Hành vi:** Xem document metadata, processing status, extracted fields và evidence.
- **Ranh giới quyền:** Document/evidence/signed access phải cùng tenant/case.
- **Kết quả quan sát được:** API/UI chỉ hiển thị evidence của active document được ủy quyền.

## ING — Ingestion và retry

### ING-001 — Acknowledge upload bất đồng bộ

- **Actor:** API.
- **Hành vi:** Stream PDF hợp lệ vào object storage, tính SHA-256, tạo `Document` và `DocumentUploaded` outbox event trong cùng transaction, trả `202 Accepted` với `document_id` và `status_url`.
- **Ranh giới quyền:** Authenticated user, active membership, role, tenant ownership và case state được kiểm tra trước write.
- **Kết quả quan sát được:** Client nhận acknowledgement có thể theo dõi; event chỉ chứa verified identifiers, không chứa PDF binary.

### ING-002 — Idempotent worker run

- **Actor:** Worker.
- **Hành vi:** Xử lý at-least-once theo `document_id + pipeline_version` với checkpoint `FILE_VALIDATED → PARSED → CLASSIFIED → EXTRACTED → VALIDATED → CHUNKED → INDEXED → COMPLETED`.
- **Ranh giới quyền:** Worker chỉ dùng identifiers API đã xác minh và giữ tenant scope.
- **Kết quả quan sát được:** Redelivery/retry không tạo extraction hoặc chunk duplicate; tiến trình run xem được.

### ING-003 — Retry/dead letter

- **Actor:** Worker.
- **Hành vi:** Retry tối đa ba lần với exponential backoff + jitter cho timeout, rate limit, service unavailable và lỗi kết nối.
- **Ranh giới quyền:** Chỉ transient technical failure được retry, không làm đổi business state tùy ý.
- **Kết quả quan sát được:** Run hiển thị `RETRY_PENDING` hoặc `FAILED`; permanent PDF/schema failure chuyển dead-letter queue, không retry vô hạn.

## EXT — Extraction

### EXT-001 — Extract có evidence

- **Actor:** Worker.
- **Hành vi:** Parse/OCR, classify, chọn versioned schema, structured extraction, schema validation, normalization và confidence check.
- **Ranh giới quyền:** Model chỉ chạy trong tenant-scoped workflow và không quyết định review state.
- **Kết quả quan sát được:** Field lưu `raw_value`, `normalized_value`, confidence và evidence truy vết được.

### EXT-002 — Critical fields

- **Actor:** Worker.
- **Hành vi:** Trích legal name, registration number, tax ID, account holder, account number, quotation total và quotation validity.
- **Ranh giới quyền:** Chỉ active document của tenant/case được dùng.
- **Kết quả quan sát được:** Critical field có value/evidence hoặc được báo thiếu/không tin cậy cho validation.

## VAL — Deterministic validation

### VAL-001 — Run rules

- **Actor:** Deterministic rule engine.
- **Hành vi:** Kiểm tra completeness, required fields, format, cross-document consistency, quotation validity/calculation, confidence và duplicate/version.
- **Ranh giới quyền:** LLM không quyết định severity, rule pass/fail hoặc case status.
- **Kết quả quan sát được:** `ValidationRun` tạo issues/evidence có kết quả tái lập bằng deterministic fixture.

### VAL-002 — Enforce severity

- **Actor:** Deterministic workflow.
- **Hành vi:** Gắn `ERROR`, `WARNING` hoặc `INFO` cho mỗi issue.
- **Ranh giới quyền:** Chỉ deterministic rule engine gán severity; Agent không thể hạ severity.
- **Kết quả quan sát được:** `ERROR` chặn `READY_FOR_REVIEW`; `WARNING` cần review nhưng không chặn; `INFO` chỉ ghi chú.

## RET — Retrieval và citation

### RET-001 — Retrieve filtered evidence

- **Actor:** Authenticated user qua API/Agent.
- **Hành vi:** Chạy keyword/BM25 + vector search, mandatory filter `tenant_id + case_id + active document`, top 20 candidates, rerank, top 5 context.
- **Ranh giới quyền:** Server áp filter; model/client không tự chọn tenant.
- **Kết quả quan sát được:** Retrieval chỉ trả chunk của case/document active đúng tenant; top-k là configuration.

### RET-002 — Validate citation

- **Actor:** Backend.
- **Hành vi:** Kiểm tra citation có document, page, evidence snippet và bounding region khi parser cung cấp.
- **Ranh giới quyền:** Citation phải thuộc tenant/case, active document, page/chunk đúng.
- **Kết quả quan sát được:** Citation hợp lệ được trả cùng answer; citation invalid bị loại.

## AGT — Read-only Agent

### AGT-001 — Provide registered tools

- **Actor:** Authenticated user.
- **Hành vi:** Agent chỉ gọi `get_case_summary`, `list_case_documents`, `get_document_metadata`, `get_extracted_fields`, `list_validation_issues`, `search_case_evidence`, `explain_validation_issue`.
- **Ranh giới quyền:** Server truyền auth context; model không truyền `tenant_id` tùy ý và không có write tool.
- **Kết quả quan sát được:** Tool trace cho read-only action trong case đúng; Agent không sửa record/state.

### AGT-002 — Grounded answer/refusal

- **Actor:** Agent.
- **Hành vi:** Trả lời theo evidence/citation đã xác minh hoặc refusal khi evidence không đủ/citation invalid.
- **Ranh giới quyền:** PDF là untrusted data; chỉ system instruction/registered tools điều khiển Agent; server áp step, timeout, token, cost limits.
- **Kết quả quan sát được:** Không suy đoán khi thiếu evidence; Agent failure không làm hỏng upload, extraction hoặc review.

## REV — Human review

### REV-001 — Review workspace

- **Actor:** Reviewer hoặc Tenant Admin.
- **Hành vi:** Xem case, document, extracted field, issue, PDF evidence và grounded chat trước quyết định.
- **Ranh giới quyền:** Cần review permission, active membership và case cùng tenant.
- **Kết quả quan sát được:** Reviewer có evidence cần thiết; Operator không thấy action approve/reject.

### REV-002 — Approve/reject safely

- **Actor:** Reviewer hoặc Tenant Admin.
- **Hành vi:** Approve/reject case bằng idempotency key và optimistic locking.
- **Ranh giới quyền:** Chỉ role review, đúng tenant, case `READY_FOR_REVIEW` không có `ERROR` được quyết định.
- **Kết quả quan sát được:** Case chuyển `APPROVED`/`REJECTED`; conflict trả `409` và không ghi đè decision khác.

## AUD — Audit

### AUD-001 — Record audit events

- **Actor:** System.
- **Hành vi:** Audit membership, document lifecycle, processing, validation và review decision.
- **Ranh giới quyền:** Audit scope theo tenant; Tenant Admin xem audit của tenant mình.
- **Kết quả quan sát được:** Timeline truy vết actor, action, resource và thời điểm.

### AUD-002 — Preserve audit lineage

- **Actor:** System.
- **Hành vi:** Giữ document version, `ProcessingRun`, `ValidationRun`, evidence và review decision.
- **Ranh giới quyền:** Chỉ resource cùng tenant được liên kết/hiển thị; sensitive values không dùng làm log thay thế audit.
- **Kết quả quan sát được:** Có thể lần từ decision về evidence, document version và run đã tạo nó.

## Liên quan

- [Yêu cầu sản phẩm](product-requirements.md)
- [Yêu cầu phi chức năng](non-functional-requirements.md)
