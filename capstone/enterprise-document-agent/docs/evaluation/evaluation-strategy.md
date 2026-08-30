# Chiến lược evaluation

## Mục đích và ranh giới

Evaluation phải chỉ ra chất lượng của từng tầng thay vì chỉ chấm câu trả lời
cuối. Mọi report là tái lập được từ synthetic dataset, ground truth và
configuration đã version; không dùng tài liệu khách hàng, secret, full bank
account hoặc sensitive prompt trong dataset/report đã commit.

Hai loại mục tiêu được quản lý riêng:

- **offline quality gate** quyết định một classifier, extraction pipeline,
  retrieval version hoặc Agent configuration có đủ chất lượng để phát hành;
- **operational SLO** đo latency, availability và durability của hệ thống đang
  chạy. SLO không được dùng để bù cho quality gate thất bại và quality score
  không chứng minh SLO đạt.

Deterministic workflows vẫn là chủ sở hữu duy nhất của writes/state transitions;
deterministic rule engine vẫn là chủ sở hữu rule outcome/severity; Agent chỉ có
bảy tools read-only. Evaluation không tạo một state machine hay security policy
thay thế các contract hiện hành.

## Dataset synthetic và split theo supplier

Baseline MVP gồm khoảng **20 suppliers × 5 document types**: `company_profile`,
`business_registration`, `tax_registration`, `bank_information_form` và
`quotation`. Mỗi PDF là tiếng Anh, không quá 20 MB hoặc 50 pages. Dataset phải
bao phủ clean cases, missing documents/fields, low-confidence fields, format và
cross-document mismatch, quotation calculation/validity, corrupted/password-
protected inputs, unanswerable questions, prompt injection, retry/redelivery và
cross-tenant attempts.

Supplier là đơn vị split. Mọi document version, case, question, paraphrase và
failure variant của một supplier chỉ thuộc đúng một trong `development`,
`validation` hoặc `final-test`; không split theo page, document hay question.
`final-test` được khóa trước tuning và chỉ chạy cho release candidate. Số lượng
cụ thể của từng split, supplier IDs, random seed và thuật toán phân bổ nằm trong
manifest versioned; không tự reshuffle khi đổi model.

Layout chuẩn:

```text
evaluation/
├── datasets/
│   ├── manifests/
│   │   ├── dataset-manifest.json
│   │   └── supplier-splits.json
│   └── synthetic/
│       └── <supplier_id>/<case_id>/<document_type>/<document_version>.pdf
├── ground-truth/
│   ├── documents.jsonl
│   ├── cases.jsonl
│   ├── questions.jsonl
│   ├── security-scenarios.jsonl
│   └── resilience-scenarios.jsonl
├── metrics/
│   └── metric-definitions.json
└── reports/
    └── <evaluation_run_id>/
        ├── run-manifest.json
        ├── metrics.json
        ├── failures.jsonl
        └── summary.md
```

`dataset-manifest.json` ghi dataset/generator version, generation seed, file
SHA-256, synthetic-only assertion và schema version. `supplier-splits.json`
map mỗi supplier vào đúng một split. `run-manifest.json` pin Git commit,
dataset/split/ground-truth versions, pipeline/schema/rule/retrieval/prompt
versions, fake/provider adapter, provider/model identifier khi có, decoding
configuration, evaluator version, environment và run timestamp. Report không
được chứa full PDF text, full account number, token, credential, signed URL hoặc
raw sensitive model response.

## Ground-truth contract

Mỗi JSONL record có `ground_truth_version`, stable scenario ID, `supplier_id`,
`case_id`, split và provenance tới synthetic generator/seed. References dùng
opaque IDs và one-based page numbers như API contract.

### Document record

`documents.jsonl` chứa tối thiểu:

- `document_id`, `document_type`, version, file SHA-256 và expected
  classification;
- expected fields với `field_name`, `raw_value` khi an toàn,
  `normalized_value`, `required`, `critical` và acceptable comparison policy;
- evidence references gồm page, short masked `evidence_snippet`, source block/
  chunk hash và `bounding_region` khi generator/parser cung cấp;
- expected parse/input outcome cho valid, corrupted, password-protected,
  oversized hoặc page-limit scenarios;
- expected schema/pipeline version. Account number chỉ dùng masked synthetic
  display value và tenant-scoped fingerprint trong committed truth/report.

### Case và rule record

`cases.jsonl` pin active document versions, evaluation date, expected required
document/field presence, normalized operands, stable `rule_id`/rule version,
expected outcome, severity, expected issues và evidence references. Nó cũng ghi
expected case state ở deterministic validation boundary; technical `FAILED`
không bao giờ được gán thành business `REJECTED`.

### Question, citation và tool record

`questions.jsonl` chứa question, answerable flag, expected answer facts,
acceptable evidence/citation IDs, required/forbidden tools, expected refusal
reason khi unanswerable và category như retrieval, cross-document, write request
hoặc prompt injection. Tool truth chỉ dùng đúng bảy registered read-only tools;
không có generic SQL, HTTP, object-read, state-update hay review tool.

### Security và resilience records

`security-scenarios.jsonl` định nghĩa actor/membership, source tenant, target
resource relationship, path (direct ID, parent/child, list/count, cursor, cache,
search, signed access, queue, Agent tool hoặc citation) và expected denial/no
leak result. `resilience-scenarios.jsonl` định nghĩa checkpoint, injected fault,
retry class/budget, redelivery count, expected `ProcessingRun` lineage, output
uniqueness, acknowledged-document durability và expected telemetry. Payload chỉ
chứa identifiers/synthetic masked values.

Ground truth thay đổi phải qua review độc lập với candidate output. Không sửa
truth chỉ để làm một model version vượt gate; ambiguity được sửa bằng version
mới, có changelog và chạy lại baseline lẫn candidate trên cùng version.

## Các tầng evaluation

1. **Dataset/schema checks:** manifest, hash, split disjointness, ground-truth
   references, page/chunk/evidence consistency và sensitive-data scan.
2. **Deterministic unit/component:** normalizers, schema validation, rule catalog,
   state guards và expected severity trên fixtures cố định.
3. **Classification/extraction:** chạy per-document qua versioned parser,
   classifier và extraction adapter; report theo document type và field.
4. **Retrieval/citation:** build index chỉ từ split đang chạy, áp mandatory
   `tenant_id + case_id + active document`, lấy top 20 candidates, rerank và
   đánh giá top 5 context/citations.
5. **Agent:** dùng tool/evidence fixtures hoặc pinned provider configuration để
   đo tool selection, faithfulness và refusal; citation validator luôn chạy ở
   backend.
6. **Security:** chạy cross-tenant, signed-access, prompt-injection, tool allowlist
   và citation-isolation cases; không cho unfiltered fallback.
7. **Resilience/idempotency:** fault injection với containerized PostgreSQL,
   MinIO và RabbitMQ, duplicate delivery, worker crash/checkpoint resume,
   transient retries, DLQ và recovery/rebuild.
8. **Performance/operations:** chạy workload đại diện và báo SLI/SLO riêng; kết
   quả không được gộp vào offline quality score.

Deterministic tests dùng fake/mock `DocumentParser`, `ExtractionModel`,
`EmbeddingModel` và `LanguageModel`. Integration/resilience dùng services thật
trong containers và model adapters deterministic; provider benchmark chỉ được
so sánh khi provider/model/config đã pin trong manifest.

## Metric definitions và offline release gates

Mọi denominator, excluded record và failure phải xuất hiện trong report. Không
macro-average qua tenant hoặc document type để che một slice rỗng/thất bại.

| Layer | Definition | Release gate |
|---|---|---|
| Classification | Tính precision/recall/F1 cho từng trong năm document types, rồi trung bình không trọng số các F1. Invalid-input rejection báo riêng. | Document classification macro F1 ≥ **0.90** |
| Field extraction | So khớp `(document_id, field_name)` trên normalized value theo comparison policy versioned; missing là FN, spurious là FP, wrong value tính một FP + một FN. Report micro và per-field; gate dùng F1 đã khai báo trong metric manifest, không đổi giữa runs. | Field extraction F1 ≥ **0.85** |
| Critical fields | Recall = số expected critical-field instances có normalized value đúng và evidence hợp lệ / tổng expected instances trên bảy concepts: legal name, registration number, tax ID, account holder, account number, quotation total và quotation validity. | Critical-field recall ≥ **0.95** |
| Deterministic rules | Exact match stable rule ID, outcome, fixed severity, machine-readable operands và expected blocking behavior trên fixtures. | **100%** expected deterministic cases pass |
| Retrieval | Một question đạt hit khi ít nhất một relevant active evidence/chunk xuất hiện trong top 5 sau mandatory filter; trung bình trên answerable queries. Zero-result là miss. | Retrieval Recall@5 ≥ **0.90** |
| Citations | Tỷ lệ citation phát ra tồn tại, đúng tenant/case, active document/result, page/chunk/hash/snippet và thực sự support claim. Unsupported claim không citation cũng được báo riêng. | Citation correctness ≥ **0.90** |
| Faithfulness | Tỷ lệ atomic answer claims được support bởi validated citations/ground truth; evaluator rubric/version và human adjudication sample được pin. | Answer faithfulness ≥ **0.90** |
| Refusal | Denominator chỉ gồm refusal-required scenarios: evidence thiếu/invalid/stale, unanswerable, cross-tenant, write/state/severity/approval request hoặc grounding không hoàn tất do tool/budget/timeout. Correct-refusal recall = số scenario trả safe refusal đúng / tổng refusal-required scenarios. False-refusal rate trên answerable scenarios là diagnostic riêng, không đi vào gate này. | Correct refusal ≥ **0.85** |
| Tool selection | Exact required tool intent/sequence theo fixture; tool thừa, forbidden, wrong case hoặc tenant override là fail. | Agent tool-selection accuracy ≥ **0.90** |
| Tenant isolation | Mỗi scenario phải deny/no-leak ở database, object, queue, search, cache, URL, telemetry, Agent tool và citation paths. | **100%** tenant-isolation security tests pass |
| Retry/idempotency | Mỗi required fault scenario phải giữ `document_id + pipeline_version` uniqueness, checkpoint/run lineage, tối đa ba transient retries, không duplicate field/evidence/chunk và không mất acknowledged document. | Release bị chặn nếu bất kỳ invariant bắt buộc nào fail |

Một candidate chỉ pass khi mọi gate liên quan cùng đạt trên `final-test`, không
có security/resilience invariant failure và không có slice bắt buộc thiếu dữ
liệu. Trung bình tốt ở metric khác không bù được một gate fail. Regression report
so baseline/candidate trên cùng run manifest và liệt kê failure IDs để tái lập.

## Operational SLOs — báo riêng khỏi quality gates

Các mục sau là design/operational targets, không phải offline model metrics:

- upload acknowledgement P95 dưới **500 ms**, đo từ byte cuối server nhận đến
  response `202 Accepted`, bao gồm validation, storage durability, SHA-256 và
  database/outbox commit nhưng không tính thời gian client truyền file;
- successful document processing P95 dưới **hai phút**, từ lúc
  `DocumentUploaded` được published đến `ProcessingRun` đạt `SUCCEEDED`; failure-
  detection latency và failed/retry counts báo riêng;
- retrieval P95 dưới **một giây**;
- chat không streaming P95 dưới **10 giây**, gồm retrieval, tool execution và
  citation validation;
- availability target **99,5%** theo rolling 30-day window cho published API
  flows với denominator/error semantics trong non-functional requirements;
- không mất document đã acknowledge và không tạo extraction/chunk trùng khi
  redelivery/retry.

Performance report ghi workload, 10-tenant design profile, concurrency, data
volume, warm-up, sample count, environment và dependency/provider versions.
Không loại failed runs khỏi failure report; chỉ successful-processing latency
SLI dùng successful runs như contract đã định nghĩa.

## Quy trình baseline và phát hành

1. Validate manifests, hashes, split disjointness, schemas và sensitive-data
   scan; fail closed trước khi gọi model.
2. Chạy deterministic fake-adapter suite nhiều lần để phát hiện nondeterminism.
3. Chạy candidate trên `development`, tune trên `validation`, đóng configuration,
   rồi chạy `final-test` một lần cho release candidate.
4. Sinh report immutable với run manifest, overall + per-type/per-field/per-
   scenario metrics, confidence intervals khi phù hợp và complete failure list.
5. Review riêng mọi security, citation, critical-field, retry/idempotency
   failure; không waive bằng aggregate score.
6. Chỉ đánh dấu local evaluation baseline pass khi mọi offline gate bắt buộc đạt.
   Baseline này là điều kiện trước khi bắt đầu Azure migration; Azure không được
   dùng để né một local gate fail.

## Liên quan

- [Yêu cầu phi chức năng](../requirements/non-functional-requirements.md)
- [Extraction và deterministic validation](../architecture/extraction-validation.md)
- [Retrieval và read-only Agent](../architecture/retrieval-agent.md)
- [Security model](../security/security-model.md)
- [Observability](../operations/observability.md)
