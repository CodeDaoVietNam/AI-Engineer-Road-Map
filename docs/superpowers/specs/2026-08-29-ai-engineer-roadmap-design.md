# Thiết Kế Repository Lộ Trình AI Engineer

## Mục tiêu

Xây dựng repository học tập bằng tiếng Việt, có thể mở rộng trên GitHub, cho AI Engineer đang phát triển RAG, chatbot và AI Agent. Repository chuyển hai roadmap hiện có thành các module có thể học, thực hành, đo lường và đưa vào portfolio.

## Cấu trúc chuẩn

```text
ai-engineer-roadmap/
├── README.md
├── roadmap/
│   ├── advanced-ai-cloud-roadmap.md
│   └── ai-system-design-roadmap.md
├── learning-paths/
│   ├── 00-setup/
│   ├── 01-model-foundations/
│   ├── 02-llm-application-engineering/
│   ├── 03-agents-mcp-security/
│   ├── 04-evaluation-observability/
│   ├── 05-multimodal-retrieval/
│   ├── 06-fine-tuning-serving/
│   ├── 07-cloud-azure-foundations/
│   ├── 08-iac-cicd-reliability/
│   └── 09-system-design-rag-agent/
├── capstone/enterprise-document-agent/
├── docs/{decisions,diagrams,system-design,portfolio}/
├── resources/
├── templates/
└── .github/
```

Hai file gốc được giữ nguyên nội dung trong `roadmap/`, với tên ngắn, dễ đọc trên GitHub.

## Nội dung lộ trình

`00-setup` chuẩn bị Git, Python, HTTP, SQL, Docker, testing và môi trường phát triển. Các module `01` đến `08` tạo thành lộ trình 20 tuần: nền tảng model, LLM application, Agent/MCP/security, evaluation/observability, multimodal/retrieval, fine-tuning/serving, Azure, và IaC/CI-CD/reliability/FinOps.

`09-system-design-rag-agent` là lộ trình 8 tuần chuyên sâu về requirements, API/scalability, storage, ingestion queue, retrieval, performance/cost, reliability/security/observability, và deployment/portfolio. Nó là companion path: dùng để thiết kế và nâng cấp capstone, không phải một project trùng lặp.

## Chuẩn nội dung module

Mỗi module có một `README.md` tiếng Việt với:

1. Mục tiêu và kiến thức trọng tâm.
2. Thời lượng và prerequisite.
3. Kế hoạch tuần hoặc bài học.
4. Bài thực hành gắn với capstone.
5. Đầu ra bắt buộc: knowledge note, artifact, measurement, decision record.
6. Tiêu chí hoàn thành và bước tiếp theo.

Các thư mục module có `notes/`, `labs/`, `artifacts/`, `exercises/` và `.gitkeep` để giữ cấu trúc trên GitHub. Không tạo code ứng dụng giả; code chỉ được thêm khi bắt đầu từng lab hoặc capstone milestone.

## Capstone

`enterprise-document-agent` là Enterprise Document RAG/Agent. Nó tiến triển theo roadmap: document upload và ingestion, retrieval có citation/evaluation, agent có tool permission, multimodal document processing, sau đó Azure deployment, observability, security và chi phí.

Capstone tách `app/`, `docs/`, `infrastructure/`, `evaluation/`, và `experiments/`. Kiến trúc khởi đầu là modular monolith + worker, không tạo microservice hoặc Kubernetes khi chưa có nhu cầu được đo lường.

## Tài liệu và GitHub

README gốc bằng tiếng Việt là điểm bắt đầu, mô tả thứ tự học, milestone capstone và progress checklist. `templates/` cung cấp mẫu session, weekly module, ADR, experiment và project README. `docs/decisions/` lưu trade-off quan trọng; `resources/` lưu nguồn học có chọn lọc.

`.gitignore`, CONTRIBUTING, LICENSE và `.github/` được giữ. Repository đã có local Git history; việc đổi cấu trúc và tạo đầy đủ nội dung sẽ là commit tiếp theo. Không push hoặc tạo GitHub remote nếu chưa có sự cho phép riêng.

## Xác minh

Sau khi scaffold, kiểm tra tất cả module có README, các liên kết root README tồn tại, hai roadmap gốc giữ nguyên nội dung sau khi đổi tên, Git working tree sạch, và `git diff --check` không có lỗi whitespace.
