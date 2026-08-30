# Yêu cầu sản phẩm

## Mục đích và vấn đề

`enterprise-document-agent` hỗ trợ doanh nghiệp tiếp nhận và review hồ sơ onboarding nhà cung cấp. Quy trình thủ công thường chậm, khó đối chiếu dữ liệu giữa nhiều PDF, khó truy vết bằng chứng và có nguy cơ bỏ sót quy tắc. MVP rút ngắn phần kiểm tra lặp lại bằng xử lý bất đồng bộ, extraction có cấu trúc, deterministic validation và hỏi đáp có citation; quyết định nghiệp vụ cuối cùng vẫn do con người đưa ra.

## Personas

- **Operator** tạo supplier và case, upload hoặc thay document, theo dõi kết quả xử lý.
- **Reviewer** có toàn bộ quyền của Operator và là người được quyền approve/reject case.
- **Tenant Admin** quản lý membership, xem audit và có toàn bộ quyền review.

## Bộ tài liệu MVP

Mỗi `SupplierCase` gồm đúng năm loại PDF tiếng Anh sau:

1. `company_profile`
2. `business_registration`
3. `tax_registration`
4. `bank_information_form`
5. `quotation`

MVP chỉ nhận PDF, tối đa **20 MB** và **50 pages** cho mỗi file. PDF corrupted, password-protected hoặc sai định dạng phải bị từ chối với lý do rõ ràng.

## Hành trình người dùng

1. Operator đăng nhập trong tenant, tạo supplier và `SupplierCase`.
2. Operator upload hoặc thay từng PDF; hệ thống xác nhận nhận file và xử lý nền.
3. Hệ thống phân loại, trích xuất fields/evidence, chạy deterministic rules và hiển thị trạng thái, issues cùng tài liệu nguồn.
4. Reviewer mở case, xem extracted fields, PDF evidence, validation issues và grounded chat có citation.
5. Khi không còn issue `ERROR`, Reviewer hoặc Tenant Admin approve/reject; quyết định và ngữ cảnh được ghi audit. Case của tenant khác luôn không thể truy cập.

## Ranh giới human review

AI chỉ được classify, extract, retrieve evidence và tạo cảnh báo. Deterministic rule engine là nơi duy nhất quyết định rule pass/fail. Deterministic workflows là nơi duy nhất được ghi dữ liệu và chuyển state. Agent chỉ có read-only tools; không được sửa dữ liệu, chuyển state hoặc approve/reject. Chỉ Reviewer hoặc Tenant Admin được approve/reject.

## Phạm vi MVP

- Một modular monolith với asynchronous worker, chạy local-first qua adapters.
- Onboarding supplier theo tenant với membership và RBAC.
- Upload/versioning PDF, reliable ingestion, extraction, evidence và deterministic cross-document validation.
- Hybrid retrieval, grounded chat và citation được backend xác minh.
- Human review, audit, evaluation dataset synthetic và observability nền tảng.
- Quy mô demo/acceptance độc lập với capacity: một tenant, 3–5 users, 20 suppliers và khoảng 100 PDF synthetic.
- Quy mô thiết kế: 10 tenants, 100 users mỗi tenant, 1.000 cases mỗi tenant, trung bình năm documents mỗi case, khoảng 1.000 uploads/ngày, peak 20 uploads/phút và 50 chat requests đồng thời.

## Ngoài phạm vi MVP

- Tự động approve/reject hoặc Agent write tools.
- Multi-agent workflow và MCP server.
- Tài liệu song ngữ.
- Kubernetes, multi-region, billing, subscription hoặc quản trị SaaS đầy đủ.
- Microservices tách theo từng domain.

## Tín hiệu thành công

- Upload acknowledgement P95 dưới 500 ms, không tính thời gian truyền file; document processing P95 dưới hai phút; retrieval P95 dưới một giây; chat không streaming P95 dưới 10 giây.
- Availability target 99,5%; không mất document đã được xác nhận upload; redelivery/retry không tạo extraction hoặc chunk trùng.
- Document classification macro F1 ≥ 0,90; field extraction F1 ≥ 0,85; critical-field recall ≥ 0,95; Retrieval Recall@5 ≥ 0,90; citation correctness và answer faithfulness mỗi chỉ số ≥ 0,90; correct refusal ≥ 0,85; Agent tool-selection accuracy ≥ 0,90.
- Tenant-isolation security tests đạt 100%; deterministic rule-engine fixtures đạt 100% expected result.

## Kịch bản nghiệm thu cuối

Một Operator thuộc tenant A đăng nhập, tạo supplier/case và upload đủ năm PDF trong giới hạn. API trả `202 Accepted` cho từng upload trong mục tiêu latency; worker xử lý lại an toàn khi redelivery mà không tạo duplicate, tạo fields và evidence, chạy deterministic validation rồi cập nhật case. Reviewer thuộc tenant A xem document, evidence, issues và grounded chat có citation hợp lệ, sau đó approve hoặc reject với audit record. Người dùng tenant B không đọc, sửa, search hay review được bất kỳ resource nào của tenant A. Evaluation report tách và báo cáo chất lượng của từng tầng, không chỉ câu trả lời cuối.

## Liên quan

- [Yêu cầu chức năng](functional-requirements.md)
- [Yêu cầu phi chức năng](non-functional-requirements.md)
