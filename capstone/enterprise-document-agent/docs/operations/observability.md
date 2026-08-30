# Observability contract

## Mục tiêu và nguyên tắc

Observability phải nối được một request hoặc ingestion job qua API → database/
outbox → queue → worker → parser/model/search adapter mà không cần full document
hay sensitive value. Telemetry hỗ trợ vận hành/điều tra; nó không là business
source of truth và không thay thế append-only `AuditEvent`.

Các nguyên tắc:

- structured logs, metrics và traces dùng cùng stable event/error vocabulary;
- tenant/case/document/run scope đến từ server-verified context, không từ PDF,
  model hay client-supplied `tenant_id`;
- redaction xảy ra trước logger/exporter; sampling/exporter không phải biện pháp
  redaction;
- offline quality gates nằm trong evaluation report; latency/availability/
  durability SLOs nằm trong operational telemetry và không được trộn điểm.

## Structured log contract

Mỗi record là một event, UTC timestamp, JSON/schema versioned. Required trên mọi
record: `timestamp`, `severity`, `service`, `environment`, `event`, `outcome`,
`telemetry_schema_version`. Các field dưới đây required khi context tồn tại:

- correlation: `request_id`, `trace_id`, `span_id`, `correlation_id`,
  `event_id`;
- authorization/lineage: `tenant_id`, `actor_id` hoặc service identity,
  `case_id`, `document_id`, `document_version`, `run_id`, `validation_run_id`,
  `pipeline_version`;
- operation: safe `route_template`/HTTP method/status class hoặc queue/worker
  `stage`, checkpoint, `duration_ms`, retry count/class và idempotent replay/
  duplicate-suppression outcome;
- dependency: stable adapter/operation, safe provider/model/deployment identifier,
  dependency status và circuit state;
- failure: stable `error_code`/error class, retryable flag; không log stack trace
  vào user-visible response.

IDs là opaque và chỉ xuất hiện khi telemetry access policy cho phép. Không dùng
raw URL/path chứa resource ID làm `route_template`; không dùng document text,
question hoặc account value làm event/metric label. Missing optional context là
`null`/omitted theo schema, không điền giá trị giả.

### Giá trị cấm

Logs, traces, metric labels, generated reports và exception payload cấm:

- secret, token, credential, password, private/encryption key;
- PDF binary, full document/OCR/chunk text hoặc raw model response;
- full bank-account value, unmasked bank evidence hoặc tenant-wide plaintext
  fingerprint;
- full user question/sensitive prompt/tool payload;
- signed URL/query string, authorization header, cookie hoặc idempotency key;
- arbitrary filename/path, stack/local variables có thể chứa payload nhạy cảm.

Snippet/tool summary chỉ lưu masked/truncated value hoặc content hash khi thực sự
cần. Provider prompt/completion chỉ có token counts, safe template/version và
result/error metadata. Log-scrubbing tests scan mọi exporter/report artifact.

## Trace propagation

API nhận `X-Request-ID` đã validate hoặc sinh mới và tham gia standard trace
context. Root span tạo server-derived tenant/actor context sau authentication;
authorization failure vẫn trace bằng safe reason mà không lookup/leak resource.

Upload/outbox flow:

1. API spans bao phủ validation, object durability, SHA-256 metadata và database/
   outbox transaction; không span attribute nào chứa bytes/full hash khi không
   cần cho correlation.
2. `DocumentUploaded` outbox row giữ trace/correlation identifiers cùng verified
   tenant/case/document/run IDs. Outbox publisher tạo producer span và inject
   trace context vào versioned message headers/body contract.
3. RabbitMQ redelivery hoặc Service Bus delivery về sau tiếp tục/links producer
   trace theo tracing standard; không tạo một unrelated root làm mất lineage.
4. Worker consumer span gắn `run_id`, delivery attempt và checkpoint; child spans
   bao phủ object read, parse, classify, extract, validate, chunk và index.
5. Provider adapters tạo client spans với provider/model/deployment, latency,
   status, tokens/cost an toàn; không ghi prompt/completion.

Explicit reprocess/retry có `ProcessingRun` mới và predecessor/root lineage;
trace có link tới originating run thay vì giả như duplicate delivery. Trace của
Agent nối API orchestration, từng registered read-only tool, retrieval/rerank và
citation validation; tool payload không được capture.

## Metrics contract

Metric name/unit/label set được version trong dashboard-as-code. Labels phải có
bounded cardinality như service, environment, route template, method, status
class, stage, error class, document type, provider/model alias và business/
technical state. Raw `request_id`, `case_id`, `document_id`, `run_id` không là
time-series labels; chúng ở logs/traces. Tenant/case allocation dùng protected
cost events/records mô tả bên dưới, không tạo unbounded public metrics.

### API và SLO metrics

- request count, latency histogram, timeout/`5xx`, intended business `4xx` và
  `409 Conflict` theo published route template;
- upload acknowledgement latency từ byte cuối server nhận đến `202 Accepted`,
  cùng durability/DB/outbox outcome;
- availability numerator/denominator cho đúng published flows: intended
  authorization/business `4xx` và `409` trong deadline là valid response;
  timeout/`5xx` unavailable; planned maintenance/dependency outage vẫn nằm trong
  denominator khi gây unavailable;
- retrieval và non-streaming chat latency, chat refusal/technical failure reason.

Operational SLOs: upload acknowledgement P95 dưới **500 ms**, successful
document processing P95 dưới **hai phút**, retrieval P95 dưới **một giây**, chat
không streaming P95 dưới **10 giây**, availability **99,5%** theo rolling 30-day
window. Đây là SLOs, không phải offline quality gates.

### Ingestion và dependency metrics

- outbox unpublished count/oldest age, publish attempts/confirm failures;
- queue depth, oldest job age, publish/consume/redelivery rate và dead-letter
  count/age;
- worker heartbeat/lease age, current technical status, checkpoint/stage latency,
  processing success latency, failure-detection latency, retry count/class và
  duplicate suppression;
- object/DB/parser/model/index operation latency/error, schema repair/budget
  exhaustion, circuit state và provider rate limit;
- active-vs-index lineage mismatch và search rebuild backlog.

Successful processing latency chỉ đo từ `DocumentUploaded` published đến
`ProcessingRun(SUCCEEDED)` trên successful runs. Failed/retried counts và
publish-to-`FAILED`/DLQ failure-detection latency luôn báo riêng, không cho
failure nhanh làm đẹp P95 success.

### Quality/security/product signals

- retrieval candidate counts/zero-result/filtered-result/reranker fallback;
- citation valid/invalid/stale reason, answer/refusal reason, Agent steps/tool
  error/timeout/budget;
- case count theo exact business state và processing count theo exact technical
  status; không trộn `FAILED` với human `REJECTED`;
- unauthorized/cross-tenant attempt, signed-access denial, invalid cursor/cache/
  citation/tool attempt theo safe route/resource class;
- tenant-isolation và deterministic-rule pass rates đến từ evaluation/test
  reports, không được suy từ production absence-of-alerts.

## Cost dimensions và budget telemetry

Mỗi parser/model/embedding/rerank/Agent call phát một protected cost-usage event
với request/trace, server-derived tenant, case và document/run khi phù hợp,
adapter operation, provider, model/deployment alias, input/output/cached tokens
hoặc provider usage unit, Agent step, configured budget, currency/rate-card
version và estimated/actual cost khi provider trả về. Không lưu prompt,
completion hoặc sensitive tool output.

Cost views hỗ trợ các dimensions `tenant`, `case`, `provider`, `model`, service,
operation và environment khi safe. Tenant/case-level records có access/retention
giới hạn, không export thành high-cardinality shared metric labels. Dashboard
aggregate dùng provider/model/service/environment; điều tra drill-down join qua
protected trace/cost store. Budget exceed fail closed theo server policy và tạo
safe alert/event; telemetry không tự tăng budget.

## Dashboards

1. **API/SLO:** availability, error-budget consumption, route latency, upload
   acknowledgement, timeout/`5xx`, business `4xx`/`409` breakdown.
2. **Ingestion:** outbox age, queue depth/job age, worker leases, checkpoint
   throughput/latency, successful processing P95, failure detection, retries và
   DLQ.
3. **Retrieval/Agent:** retrieval P95/zero results, reranker fallback, chat P95,
   citations/refusals, tool steps/errors, token/cost budgets.
4. **Security/isolation:** auth denials, cross-tenant/signed-access/search/tool/
   citation attempts, log-scrubbing/security-test status. Access là privileged.
5. **Business workflow:** exact case states, active processing statuses,
   validation `ERROR`/`WARNING`/`INFO` counts; technical `FAILED` tách khỏi human
   `REJECTED`.
6. **Cost/capacity:** usage/cost theo safe provider/model/service/environment,
   protected tenant/case drill-down, uploads/day, peak upload rate và concurrent
   chat load so với design profile.

Mỗi panel nêu source metric/query, unit, aggregation window, freshness và link
tới runbook. Dashboard không hiển thị full tenant/customer names, document text
hay account values.

## Alerts và runbooks

Alert threshold được versioned và calibrate từ local baseline/SLO error budget;
không tạo một quality target mới trong tài liệu vận hành. Alert tối thiểu:

- SLO burn hoặc P95 vượt approved target trên sustained window;
- PostgreSQL/object storage unavailable, uncertain commit hoặc checksum mismatch;
- outbox/queue oldest age tăng, publisher confirm failure, worker heartbeat/lease
  stale hoặc dead-letter có work chưa triage;
- retry/error class spike, parser/model/index circuit open, schema repair budget
  exhausted hoặc active-index lineage mismatch;
- retrieval zero-result/citation-invalid/refusal tăng bất thường, Agent token/
  cost budget exceeded;
- unauthorized/cross-tenant/signed-access attempts vượt configured security
  threshold hoặc tenant-isolation/log-scrubbing test fail;
- telemetry pipeline/exporter mất dữ liệu; thiếu telemetry không được coi là hệ
  thống khỏe.

Mỗi alert có owner, severity, safe identifiers, dashboard/trace link, runbook,
ack/escalation và rollback/degraded-mode step. Runbook không yêu cầu operator
copy full PDF/prompt/account value vào ticket. Alerts của technical failure
không tự chuyển case thành `REJECTED`.

## Audit separation

`AuditEvent` là append-only, tenant-scoped business/security record cho
membership, document lifecycle, processing/reprocess, validation snapshot,
signed access và human review decision. Nó có retention/access/integrity policy
riêng và là lineage cho privileged actions.

Operational logs/traces có thể sampled hoặc hết retention; chúng không chứng
minh một review decision. Audit không chứa full document/bank/prompt/secret và
không được dùng như verbose debug log. Azure Monitor/Application Insights về sau
có thể nhận redacted telemetry/audit notification, nhưng PostgreSQL durable audit
record vẫn là authoritative audit lineage theo contract.

## Liên quan

- [Yêu cầu phi chức năng](../requirements/non-functional-requirements.md)
- [Security model](../security/security-model.md)
- [Failure modes](../security/failure-modes.md)
- [Chiến lược evaluation](../evaluation/evaluation-strategy.md)
- [Azure mapping](azure-mapping.md)
