# Local development

## Mục tiêu

Local topology là đường chạy mặc định của MVP: React/Vite `web`, FastAPI `api`,
asynchronous `worker`, PostgreSQL 16 + pgvector, MinIO và RabbitMQ. Application
processes compose provider-neutral ports với local adapters; Agent vẫn là module
trong API, không phải service riêng.

Chạy local, seed synthetic data, tests deterministic và evaluation baseline
**không yêu cầu Azure credentials hoặc Microsoft Foundry credentials**. Azure/
Foundry provider chỉ được chọn bằng adapter configuration cho một run riêng; nó
không nằm trên startup path mặc định và thiếu Azure environment variables không
được làm local health check thất bại.

## Topology và intended local ports

Các host port dưới đây là contract dự kiến cho `compose.yml`/developer tooling
ở milestone triển khai. Nếu host đã chiếm port, override chỉ ở local ignored
configuration; service-to-service address vẫn dùng Compose service name.

| Component | Trách nhiệm | Intended host interface |
|---|---|---|
| `web` | React + TypeScript + Vite reviewer dashboard; chỉ gọi published HTTP API | `http://localhost:5173` |
| `api` | FastAPI stateless API, local JWT, retrieval/read-only Agent và composition root | `http://localhost:8000` |
| `worker` | Queue consumer, ingestion/checkpoint/retry; không expose business HTTP API | không publish host port |
| `postgres` | PostgreSQL 16 + pgvector, business data, outbox, audit và local search projection | `localhost:5432` |
| `minio` | Local `ObjectStorage` cho PDF/derivative synthetic | API `localhost:9000`; console `localhost:9001` |
| `rabbitmq` | Local `MessageQueue`, durable at-least-once delivery | AMQP `localhost:5672`; management `localhost:15672` |

Browser chỉ giao tiếp với `web`/`api`; nó không truy cập PostgreSQL, MinIO API
hoặc RabbitMQ. Signed access chỉ được API tạo sau authorization. Với
`bank_information_form`, browser chỉ nhận redacted derivative/proxy, không nhận
original object.

## Port và adapter composition

`packages/contracts` sở hữu interfaces; `packages/adapters` sở hữu local
implementations; `apps/api` và `apps/worker` chọn implementations tại composition
root. `packages/core` không import framework, database driver, RabbitMQ/storage
SDK hoặc adapter implementation.

| Port | Local adapter/dependency |
|---|---|
| `ObjectStorage` | MinIO |
| `MessageQueue` | RabbitMQ |
| `DocumentParser` | PyMuPDF hoặc deterministic fake parser |
| `ExtractionModel` | configured provider-neutral adapter hoặc deterministic fake |
| `EmbeddingModel` | configured provider-neutral adapter hoặc deterministic fake |
| `SearchIndex` | PostgreSQL 16 + pgvector adapter |
| `LanguageModel` | configured provider-neutral adapter hoặc deterministic fake |
| `IdentityProvider` | local JWT adapter |
| `AuditPublisher` | local tenant-scoped audit persistence/outbox adapter |

Fake adapters trả kết quả từ versioned fixtures theo stable scenario/document
ID, không gọi network và không dùng wall clock/randomness ngoài seed đã pin.
Chúng không được bypass schema validation, citation validation, tenant filter,
retry/idempotency hoặc deterministic rule engine.

## Configuration contract

Copy `.env.example` thành ignored `.env` khi tooling được triển khai. File
committed chỉ chứa names và safe defaults; secret/token/password/private key
không được commit, in vào log hoặc đưa vào image. Names hiện hành:

- database: `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_NAME`,
  `DATABASE_SSLMODE`;
- object storage: `OBJECT_STORAGE_ENDPOINT`, `OBJECT_STORAGE_REGION`,
  `OBJECT_STORAGE_BUCKET`, `OBJECT_STORAGE_USE_SSL`;
- RabbitMQ: `RABBITMQ_HOST`, `RABBITMQ_PORT`, `RABBITMQ_VHOST`;
- local identity metadata: `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_ALGORITHM`;
- provider selection: `EXTRACTION_MODEL_PROVIDER`,
  `EMBEDDING_MODEL_PROVIDER`, `LANGUAGE_MODEL_PROVIDER`,
  `DOCUMENT_PARSER_PROVIDER`;
- browser: `CORS_ALLOWED_ORIGINS`;
- telemetry: `TELEMETRY_ENABLED`, `TELEMETRY_SERVICE_NAME`, `LOG_LEVEL`.

Tài liệu này không định nghĩa credential values. Local-only credentials cần cho
container/tooling về sau phải nằm trong ignored `.env` hoặc runtime secret, với
least privilege. Provider variables chọn adapter bằng stable provider name;
deterministic test/evaluation profile phải chọn fake/mock adapters. Không thêm
`AZURE_*`, `FOUNDRY_*`, Entra hoặc Key Vault requirement vào local profile.

## Health và readiness contract

Health check không chạy model generation, không đọc full PDF và không trả
configuration/secret. Liveness chỉ cho biết process loop còn sống; readiness
chỉ pass khi dependency bắt buộc cho capability của process sẵn sàng.

| Component | Liveness | Readiness |
|---|---|---|
| PostgreSQL | server accepts connection | database/migrations hiện hành và `vector` extension usable |
| MinIO | MinIO live check | target bucket reachable với bounded read/write metadata check |
| RabbitMQ | broker diagnostics/process alive | target vhost, exchange/queue contract và authenticated channel usable |
| API | planned unauthenticated `/health/live` chỉ trả safe status | planned `/health/ready` kiểm tra PostgreSQL/configuration và báo riêng object/queue dependency status; upload capability chỉ ready khi MinIO usable, còn RabbitMQ outage không vô hiệu durable outbox commit |
| Worker | process heartbeat/consumer loop | PostgreSQL, object storage và queue reachable; model/parser adapter configuration valid nhưng readiness không gọi provider billable |
| Web | Vite/static root responds | API base URL/config đã load; browser business action vẫn dựa vào API authorization |

Tên HTTP health endpoints ở trên là M0 implementation contract, không mở rộng
catalog business `/api/v1`. RabbitMQ/MinIO/PostgreSQL checks dùng vendor health
mechanism qua tooling; dashboard/operator không coi container `running` là
readiness.

## Startup ordering

1. Validate non-secret configuration; fail nếu profile local vô tình chọn một
   Azure-only adapter mà không được yêu cầu rõ ràng.
2. Start PostgreSQL, MinIO và RabbitMQ; chờ/report readiness thay vì fixed sleep.
3. Apply versioned database migrations, verify pgvector extension, provision
   private MinIO bucket/prefix policy và declare versioned RabbitMQ exchange,
   queue, retry/dead-letter topology idempotently.
4. Start API/outbox publisher sau PostgreSQL và MinIO readiness. Durable
   `202 Accepted` cần object durability + database/outbox commit, không cần broker
   confirm. Nếu RabbitMQ chưa ready, publisher ở degraded state, outbox giữ event
   chưa publish và API không claim processing đã bắt đầu/hoàn tất.
5. Start worker; worker claim `ProcessingRun` đã tồn tại và không tự tạo initial
   run. Một worker chưa ready không được làm API báo processing đã hoàn tất.
6. Start web sau khi API address được cấu hình.
7. Seed hoặc chạy evaluation chỉ sau khi application/infrastructure readiness
   pass.

Shutdown ngược thứ tự: dừng nhận browser traffic/new uploads, drain bounded
worker/outbox work hoặc để broker redeliver, rồi dừng infrastructure. Không xóa
volume/object bằng shutdown mặc định; reset dữ liệu phải là explicit destructive
developer action và không dùng broad/unresolved path.

## Seed và evaluation workflow

Milestone M0 chưa hứa các lệnh `make` đã executable. Khi tooling được thêm, nó
phải hiện thực cùng một workflow có thể chạy tay/CI mà không cần cloud:

1. chọn deterministic local profile và fake parser/extraction/embedding/language
   adapters;
2. validate synthetic dataset manifest, SHA-256, ground-truth schema và supplier
   split disjointness;
3. reset/seed chỉ database, bucket và queue namespace dành riêng cho evaluation
   run; không chạm developer/customer namespace khác;
4. tạo local JWT identities/memberships cho Operator, Reviewer, Tenant Admin và
   ít nhất hai synthetic tenants để test isolation;
5. ingest synthetic PDFs qua published API contract, không insert trực tiếp để
   bypass authorization/outbox; theo dõi `202 Accepted` status đến terminal
   processing leaf/validation snapshot;
6. chạy deterministic rule, retrieval, Agent, tenant-isolation và retry/
   idempotency suites; failure injection dùng real containerized PostgreSQL,
   MinIO/RabbitMQ nhưng fake model adapters;
7. ghi report vào `evaluation/reports/<evaluation_run_id>/` với run manifest và
   không chứa secrets/full sensitive values;
8. chạy lại cùng seed/config để kiểm tra reproducibility, sau đó giữ volumes cho
   điều tra hoặc xóa đúng evaluation namespace bằng explicit command.

Provider benchmark tùy chọn chạy như một profile khác, pin provider/model và
cost metadata; nó không thay deterministic local acceptance suite. Local
baseline chỉ pass khi các offline gates trong evaluation strategy đạt.

## Local failure/degraded behavior

- PostgreSQL unavailable: business reads/writes fail closed; không authorize từ
  object/queue cache.
- MinIO unavailable trước durability: upload trả stable dependency error và
  không trả `202 Accepted`.
- RabbitMQ unavailable sau commit: outbox giữ unpublished event; status có thể
  ở `QUEUED`, không mất acknowledged document.
- Worker/model/parser unavailable: existing authorized reads vẫn hoạt động;
  processing giữ `QUEUED`/`RETRY_PENDING`/`FAILED`, không thành `REJECTED`.
- Search/Agent semantic evidence failure: chat refusal theo contract; technical
  dependency/tool/budget/timeout failure: `CHAT_GROUNDING_UNAVAILABLE`, không
  tính là natural-language refusal và không unfiltered fallback.

## Liên quan

- [Tổng quan kiến trúc](../architecture/system-overview.md)
- [Ingestion pipeline](../architecture/ingestion-pipeline.md)
- [Failure modes](../security/failure-modes.md)
- [Chiến lược evaluation](../evaluation/evaluation-strategy.md)
- [Azure mapping](azure-mapping.md)
