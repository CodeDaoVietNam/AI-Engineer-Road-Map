# Security model

## Mục tiêu và trust boundaries

Security model bảo vệ tenant isolation, document/bank data, integrity của human
review và khả năng truy vết. Các trust boundary chính là browser → API, API →
PostgreSQL/object storage/outbox, outbox → queue → worker, worker/API → parser và
model providers, API → search index, cùng API → read-only Agent.

Nguyên tắc nền:

- deny by default và authorize ở application layer trước mọi side effect;
- authenticated active membership là nguồn duy nhất của tenant/role;
- deterministic workflows sở hữu writes/state transitions; rule engine sở hữu
  rule outcome/severity; Agent chỉ đọc;
- PDF gốc trong object storage và PostgreSQL metadata là recovery source of
  truth; queue/search index là projection/delivery mechanism;
- technical failure không được diễn giải thành business `REJECTED`.

## Identity, session và authorization

`IdentityProvider` xác minh credential/token; application đọc lại active
`Membership` cho request hiện hành và derive `user_id`, `tenant_id`, role. Token
hợp lệ không đủ nếu membership inactive. Token có expiry ngắn phù hợp, issuer,
audience và signature được validate; refresh/revocation policy thuộc identity
adapter và không được bỏ qua membership check.

Mỗi action kiểm tra authenticated identity, active membership, permission,
resource tenant và resource state/version server-side. Không tin `tenant_id`,
role hoặc user identifier từ body/query/header/model/tool. Cross-tenant lookup
trả not found đồng nhất để chống enumeration. Authorization xảy ra trước object
read, signed URL, model/tool invocation, search hoặc queue publish.

## Tenant isolation theo lớp

### PostgreSQL

- Mọi business record, join và audit record có `tenant_id`; child tenant phải
  khớp parent tenant.
- Query/update/delete luôn có tenant predicate từ server auth context. Composite
  foreign key/unique constraint ngăn cross-tenant relationship khi khả thi.
- Transaction compare-and-set bảo vệ case version, active document/result và
  validation snapshot; không dùng last-write-wins cho review/replacement.
- Connection role có least privilege; migration/backup role tách runtime role.
  Database connection dùng TLS ngoài local-only network và credentials được
  rotate qua secret store.
- Backup/restore giữ encryption và tenant isolation; restore test không dùng dữ
  liệu khách hàng trong fixture/report.

### Object storage

- Server tạo object key chứa opaque tenant/case/document/version identifiers;
  không dùng filename, path hoặc PDF metadata client cung cấp.
- Bucket/container private, public listing/access tắt. API/worker dùng service
  identity least privilege; tenant-scoped prefix được kiểm tra với database
  metadata trước read/write/delete.
- Object write phải có durability confirmation trước `202 Accepted`; object
  encryption at rest và TLS in transit là bắt buộc.
- Browser chỉ nhận signed URL có TTL ngắn, verb/object cố định và không cho
  listing. Application re-authorize active membership, tenant, case/document
  và state trước mỗi lần phát URL; URL không phải quyền lâu dài.
- Original versions immutable theo retention/audit policy. Orphan object sau DB
  failure được cleanup bằng job tenant-safe, không bằng broad prefix delete.

### Transactional outbox và queue

- `Document`, initial `ProcessingRun(QUEUED)` và `DocumentUploaded` outbox row
  commit cùng database transaction. Outbox là source of truth cho event chưa
  publish; publisher claim row và cần broker confirm trước mark published.
- Message là persistent, schema-versioned và chỉ chứa verified opaque
  identifiers: tenant/case/document/run/event/pipeline/trace; không chứa PDF
  binary, full text, prompt, credential hoặc bank value.
- Broker dùng authenticated service identities, TLS, least-privilege vhost/topic
  và durable queue. Producer/consumer không nhận routing key từ PDF/client.
- Worker đối chiếu toàn relationship với database trước object read/write; chữ
  ký/format message không thay tenant authorization/invariant check.
- At-least-once delivery được chặn duplicate theo
  `document_id + pipeline_version`, run/checkpoint và unique/upsert constraints.
  Dead-letter payload vẫn redacted và tenant-scoped; replay chỉ qua authorized
  reprocess/operational runbook.

### Search index và cache

- Mọi chunk/vector/cache key chứa tenant, case, document version và active result
  lineage. Index write chỉ từ validated worker context.
- Keyword và vector search bắt buộc áp filter
  `tenant_id + case_id + active document` trước candidate selection. Không có
  unfiltered fallback khi dependency lỗi.
- Search result được re-check ownership/active markers trước surface/citation.
  Query/cursor/cache entry bind tenant để không reuse cross-tenant.
- Search index là rebuildable projection, không phải source of truth; stale
  result bị loại ngay khi replacement/reprocess clear active pointer.

### Logs, metrics và traces

- Structured telemetry có request/trace/tenant/case/document/run identifiers,
  event, duration và stable error code. Identifier dùng opaque value theo quyền
  truy cập telemetry; không dùng document content làm label.
- Cấm secrets, tokens, credentials, full document text, full bank account,
  sensitive prompt/tool payload, signed URL query, raw model response và PDF
  binary trong log/trace/metric/report.
- Snippet/tool output được mask, truncate hoặc chỉ lưu content hash. Telemetry
  access là privileged, least privilege, retention giới hạn và được audit.
- Redaction xảy ra trước exporter; provider-side sampling không được xem như cơ
  chế redaction. Unauthorized/cross-tenant attempts có metric/alert riêng.

## Secrets và cryptography

Committed configuration chỉ chứa variable names và safe defaults. Local secret
nằm trong ignored `.env`; deployed secret ở managed secret store (Azure Key
Vault khi dùng Azure). Không đưa secret vào image, source, fixture, CI artifact,
URL, queue hoặc model prompt.

Service credential dùng scope tối thiểu, rotate định kỳ/khẩn cấp và tách theo
environment/service. TLS bảo vệ external/service connections. Encryption keys
không đồng vị trí với ciphertext; key access và rotation là privileged action
có audit. Backup, snapshot và exported diagnostics dùng cùng hoặc mạnh hơn
encryption/access policy của primary data.

## Bank-data protection

`account_number` canonical được mã hóa ở application/storage boundary bằng
authenticated encryption và managed key; database chỉ lưu ciphertext, key
version và tenant-scoped fingerprint cần cho deterministic comparison/duplicate
rule. Plaintext chỉ tồn tại trong memory tối thiểu của authorized workflow và
không đi vào search index, queue, cache, log, trace, fixture, generated report
hoặc Agent context.

API/UI/Agent/evidence hiển thị masked value (ví dụ chỉ bốn chữ số cuối). Raw
evidence snippet và bounding region phải được redact/mask trước khi rời trusted
processing boundary. Reviewer/Tenant Admin không mặc định nhận plaintext qua
API; future unmask cần permission/step-up/audit contract riêng, ngoài MVP.

## Upload security

Upload validation là streaming và fail closed:

1. enforce request/file byte limit **20 MB** không dựa vào `Content-Length`;
2. chỉ chấp nhận năm approved document types và PDF tiếng Anh;
3. kiểm tra magic/signature + parse structure, không tin filename/MIME/extension;
4. từ chối corrupted, password-protected, malformed PDF và file quá **50 pages**;
5. giới hạn parser CPU, memory, recursion, decompression/embedded-object budget
   và timeout để chống PDF bomb/DoS;
6. không thi hành JavaScript, macro, attachment, external URI hoặc embedded file;
7. tính SHA-256 khi stream, ghi vào server-generated key và chỉ acknowledge sau
   durability + database/outbox commit.

Upload không hợp lệ không tạo processing job. Parser/OCR chạy trong isolated,
least-privilege worker/container, không shell, không arbitrary network, không
host filesystem write ngoài scratch/object adapter. Scratch data được bounded,
tenant-scoped và xóa an toàn sau run.

Malware scanning có thể là defense-in-depth adapter khi triển khai, nhưng không
thay structural validation, sandbox hoặc authorization. Filename chỉ là
sanitized display metadata.

## Untrusted PDF và prompt injection

PDF, OCR text, chunk, user question và model output đều là untrusted data plane.
Chuỗi trong PDF như “ignore previous instructions”, tool name, system-looking
prompt, URL hoặc instruction không được trở thành control plane.

- Chỉ system instruction và registered tool schemas do server cấu hình điều
  khiển Agent; prompt injection trong PDF chỉ có thể được trích như evidence.
- Model không tự đăng ký tool, chọn tenant/filter, tạo signed URL, SQL/HTTP/shell
  call hoặc thay budget.
- Tool input/output schema-validated, size-limited, encoded và bound server auth
  context; arbitrary HTML/script không được render executable.
- Model output không trực tiếp tạo database/index/audit write, state transition,
  severity hoặc review decision.
- Citation validator kiểm tra tồn tại, tenant/case, active document/result,
  page/chunk/hash/snippet; invalid/insufficient evidence gây refusal.

## Agent restrictions và model-provider controls

Agent chỉ có bảy read-only tools: `get_case_summary`, `list_case_documents`,
`get_document_metadata`, `get_extracted_fields`, `list_validation_issues`,
`search_case_evidence`, `explain_validation_issue`. Không có generic database,
object read, filesystem, shell, HTTP fetch, state mutation hoặc decision tool.

Server enforce allowlist, active membership/resource checks, mandatory search
filter, maximum steps, timeout, token và cost budget. Tool trace chỉ lưu safe
identifiers/masked summary. Model/provider request được data-minimize; provider
adapter không nhận storage credentials hoặc broad tenant access. Provider
timeout/rate limit/failure chỉ tạo refusal/technical error và bị cô lập khỏi
upload, extraction state đã commit và review API.

## URLs, browser và output safety

- Signed URL dùng HTTPS, TTL ngắn, fixed object/method và response headers an
  toàn; không log toàn URL hoặc đưa vào long-lived cache/audit payload.
- API đặt content disposition/type an toàn và browser hardening headers. PDF
  viewer không execute active content; snippet/field/chat output được escape.
- Cursor, redirect và callback URL là allowlisted/signed; không open redirect
  hoặc SSRF từ document/user URL.
- CORS dùng explicit trusted origins; credentialed wildcard bị cấm. CSRF
  protection áp dụng nếu auth dùng cookie; local JWT storage/transport phải theo
  threat model đã chọn.

## Privileged actions và audit

Append-only, tenant-scoped `AuditEvent` bao phủ tối thiểu:

- membership role/active-status change;
- document upload, replacement, signed-access issuance và reprocess;
- processing retry/dead-letter/replay/active-result selection;
- validation snapshot activation/invalidation và state transition;
- Reviewer/Tenant Admin `APPROVED`/`REJECTED` decision;
- denied privileged/cross-tenant attempt và secret/key/config administrative
  change khi platform cung cấp.

Event lưu actor/service identity, action, target type/opaque ID, tenant, outcome,
time, request/trace, case/document/run/validation lineage, previous/new version
và safe reason code khi phù hợp. Không lưu full bank value, full document,
credential, prompt hoặc signed URL. Tenant Admin xem audit tenant mình; platform
operations access là separate least-privilege path và cũng được audit.

## Security verification gates

- Tenant-isolation security tests phải pass 100% cho direct ID, parent/child
  mismatch, list/count/search/cursor/cache, signed URL, queue message, Agent tool
  và citation paths.
- Contract tests xác minh mọi endpoint chạy đủ năm authorization checks và
  cross-tenant trả indistinguishable not found.
- Upload tests gồm MIME spoof, truncation, encrypted/corrupted PDF, oversized
  stream, page bomb, embedded active content và resource-budget exhaustion.
- Prompt-injection/red-team fixtures xác minh tool allowlist, tenant filter,
  read-only/write refusal, output encoding và citation refusal.
- Review concurrency tests xác minh stale version/snapshot nhận `409 Conflict`
  và không tạo duplicate/overwrite decision.
- Log-scrubbing tests tìm token, secret, full bank value, document text và signed
  URL trong log/trace/report.

## Liên quan

- [API contract](../api/api-contract.md)
- [Authorization matrix](../api/authorization-matrix.md)
- [Failure modes](failure-modes.md)
- [Retrieval và read-only Agent](../architecture/retrieval-agent.md)
