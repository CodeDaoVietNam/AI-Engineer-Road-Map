# Enterprise Document Agent Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the self-contained `enterprise-document-agent` project structure and its complete M0 documentation set inside the learning repository.

**Architecture:** The capstone is a path-isolated project under `capstone/enterprise-document-agent/` with React web, FastAPI API, asynchronous worker, shared Python core/contracts/adapters, tests, evaluation assets, local/Azure infrastructure, and focused design documents. This plan creates documentation and directory contracts only; application behavior starts in separate milestone plans M1–M7.

**Tech Stack:** Markdown, Git, React + TypeScript + Vite, FastAPI, PostgreSQL 16 + pgvector, MinIO, RabbitMQ, Docker Compose, pytest, Vitest, React Testing Library, Playwright.

**Spec:** `docs/superpowers/specs/2026-08-30-enterprise-document-agent-design.md`

## Global Constraints

- All project-owned files live below `capstone/enterprise-document-agent/`.
- The capstone must not import from `learning-paths/`, `resources/`, or any parent directory.
- Do not create a nested `.git` directory.
- MVP accepts five English PDF types, with limits of 20 MB and 50 pages per file.
- Architecture is modular monolith + asynchronous worker, not microservices.
- Agent is read-only; deterministic workflows own writes and state changes.
- Local development must not require Azure or Microsoft Foundry credentials.
- Documentation and empty-directory scaffolding are validated structurally; no application code is introduced in M0.
- Preserve unrelated and previously uncommitted roadmap changes.

---

### Task 1: Normalize the capstone boundary

**Files:**
- Inspect: `capstone/enterprise-document-agent/`
- Inspect: `capstone/enterprise-document-rag-assistant/`
- Create: `capstone/enterprise-document-agent/README.md`
- Create: `capstone/enterprise-document-agent/AGENTS.md`
- Create: `capstone/enterprise-document-agent/.gitignore`
- Create: `capstone/enterprise-document-agent/.env.example`

**Interfaces:**
- Consumes: repository design and global constraints from the approved specification.
- Produces: one canonical capstone root with documented commands, ownership boundaries, naming rules, and safe configuration conventions.

- [ ] **Step 1: Inspect both capstone directories and confirm the legacy directory contains no user-authored source before consolidating it.**

Run: `find capstone/enterprise-document-agent capstone/enterprise-document-rag-assistant -maxdepth 3 -type f -print | sort`

Expected: the legacy directory contains only scaffold markers; if it contains authored files, stop and preserve them instead of removing it.

- [ ] **Step 2: Write the capstone README.**

Include the product statement, five document types, users, human-in-the-loop boundary, local-first strategy, repository map, milestone list, and commands that will exist after M1. Clearly label application commands as milestone targets rather than currently executable commands.

- [ ] **Step 3: Write project-local contributor instructions.**

`AGENTS.md` must define Python/TypeScript naming, domain dependency direction, test placement, tenant-isolation rule, no-secret policy, and verification expectations.

- [ ] **Step 4: Add safe local configuration contracts.**

`.env.example` lists non-secret variable names for database, object storage, RabbitMQ, JWT, model adapters, CORS, and telemetry. `.gitignore` excludes `.env`, virtual environments, node modules, build output, caches, coverage output, generated evaluation reports, and local uploaded documents.

- [ ] **Step 5: Verify boundary files and secret rules.**

Run: `test -f capstone/enterprise-document-agent/README.md && test -f capstone/enterprise-document-agent/AGENTS.md && git check-ignore -q capstone/enterprise-document-agent/.env`

Expected: all checks exit 0.

### Task 2: Scaffold source, infrastructure, evaluation, and test boundaries

**Files:**
- Create: `capstone/enterprise-document-agent/apps/web/README.md`
- Create: `capstone/enterprise-document-agent/apps/api/README.md`
- Create: `capstone/enterprise-document-agent/apps/worker/README.md`
- Create: `capstone/enterprise-document-agent/packages/core/README.md`
- Create: `capstone/enterprise-document-agent/packages/contracts/README.md`
- Create: `capstone/enterprise-document-agent/packages/adapters/README.md`
- Create: `capstone/enterprise-document-agent/infrastructure/local/README.md`
- Create: `capstone/enterprise-document-agent/infrastructure/azure/README.md`
- Create: `capstone/enterprise-document-agent/evaluation/{datasets,ground-truth,metrics,reports}/.gitkeep`
- Create: `capstone/enterprise-document-agent/tests/{integration,end-to-end,security,performance}/.gitkeep`
- Create: `capstone/enterprise-document-agent/migrations/.gitkeep`

**Interfaces:**
- Consumes: code-boundary design from specification sections 4–5.
- Produces: explicit ownership contracts for future M1–M7 implementation plans.

- [ ] **Step 1: Create application boundary READMEs.**

Document that `web` owns UI only, `api` owns HTTP/authentication context only, and `worker` owns queue handlers only. Both Python apps call shared application use cases rather than duplicating domain logic.

- [ ] **Step 2: Create package boundary READMEs.**

Document that `core` is framework-independent domain/application logic, `contracts` owns schemas/events/ports, and `adapters` owns local/Azure integrations. List the nine approved ports verbatim.

- [ ] **Step 3: Create infrastructure, evaluation, migration, and test directories with focused README or `.gitkeep` markers.**

- [ ] **Step 4: Verify the complete boundary list.**

Run: `find capstone/enterprise-document-agent/apps capstone/enterprise-document-agent/packages capstone/enterprise-document-agent/infrastructure capstone/enterprise-document-agent/evaluation capstone/enterprise-document-agent/tests -maxdepth 2 -type d | sort`

Expected: all directories named in the approved specification are present and no `agent` application service exists.

### Task 3: Write product and system requirements

**Files:**
- Create: `capstone/enterprise-document-agent/docs/requirements/product-requirements.md`
- Create: `capstone/enterprise-document-agent/docs/requirements/functional-requirements.md`
- Create: `capstone/enterprise-document-agent/docs/requirements/non-functional-requirements.md`

**Interfaces:**
- Consumes: specification sections 1–3 and 16.
- Produces: stable scope and measurable acceptance criteria used by all later milestone plans.

- [ ] **Step 1: Write product requirements.**

Cover problem statement, personas, five document types, user journeys, human-review boundary, MVP scope, exclusions, success signals, and final acceptance scenario.

- [ ] **Step 2: Write numbered functional requirements.**

Use stable IDs grouped as `AUTH`, `TEN`, `SUP`, `CASE`, `DOC`, `ING`, `EXT`, `VAL`, `RET`, `AGT`, `REV`, and `AUD`. Each requirement states actor, behavior, authorization boundary, and observable result.

- [ ] **Step 3: Write numbered non-functional requirements.**

Use stable IDs for performance, availability, consistency, security, privacy, auditability, recoverability, operability, maintainability, portability, and cost. Include all approved P95 and design-target values.

- [ ] **Step 4: Verify critical scope and targets.**

Run: `rg -n "company_profile|business_registration|tax_registration|bank_information_form|quotation|20 MB|50 pages|P95|tenant" capstone/enterprise-document-agent/docs/requirements`

Expected: every approved type, limit, latency class, and tenant boundary is represented.

### Task 4: Write architecture, domain, and workflow documents

**Files:**
- Create: `capstone/enterprise-document-agent/docs/architecture/system-overview.md`
- Create: `capstone/enterprise-document-agent/docs/architecture/ingestion-pipeline.md`
- Create: `capstone/enterprise-document-agent/docs/architecture/extraction-validation.md`
- Create: `capstone/enterprise-document-agent/docs/architecture/retrieval-agent.md`
- Create: `capstone/enterprise-document-agent/docs/data-model/domain-model.md`
- Create: `capstone/enterprise-document-agent/docs/data-model/state-machines.md`

**Interfaces:**
- Consumes: requirements from Task 3 and specification sections 4–9.
- Produces: component responsibilities, data flows, domain vocabulary, invariants, retry behavior, and AI boundaries.

- [ ] **Step 1: Write system overview with Mermaid context/container diagrams and dependency rules.**
- [ ] **Step 2: Write ingestion pipeline with upload validation, 202 response, transactional outbox, RabbitMQ delivery, checkpoints, retry classification, dead-letter behavior, API idempotency, and run/checkpoint delivery dedupe.**
- [ ] **Step 3: Write extraction/validation design with versioned schemas, standard fields for all five document types, normalization, confidence thresholds, severity model, rule catalog, and evidence contract.**
- [ ] **Step 4: Write retrieval/Agent design with structure-aware chunks, hybrid retrieval, configuration-based top-k, mandatory filters, seven read-only tools, citation validation, refusal, and prompt-injection boundary.**
- [ ] **Step 5: Write domain model and state machines with Mermaid diagrams, entity definitions, tenant invariants, document versioning, processing runs, business state and technical processing state.**
- [ ] **Step 6: Verify architecture vocabulary is internally consistent.**

Run: `rg -n "DOCUMENTS_UPLOADED|VALIDATION_REQUIRED|READY_FOR_REVIEW|RETRY_PENDING|document_id \+ pipeline_version|tenant_id \+ case_id|read-only" capstone/enterprise-document-agent/docs/architecture capstone/enterprise-document-agent/docs/data-model`

Expected: approved state names, idempotency key, retrieval filter, and Agent boundary appear exactly as specified.

### Task 5: Write API, authorization, security, and failure contracts

**Files:**
- Create: `capstone/enterprise-document-agent/docs/api/api-contract.md`
- Create: `capstone/enterprise-document-agent/docs/api/authorization-matrix.md`
- Create: `capstone/enterprise-document-agent/docs/security/security-model.md`
- Create: `capstone/enterprise-document-agent/docs/security/failure-modes.md`

**Interfaces:**
- Consumes: domain states from Task 4.
- Produces: `/api/v1` endpoint catalog, request/response contracts, permission rules, threat boundaries, and degraded-operation behavior.

- [ ] **Step 1: Write the endpoint catalog and representative request/response/error bodies.**

Include auth/me, suppliers, cases, document upload/status/evidence/reprocess, extracted fields, issues, chat, review decisions, and audit events. Specify `202`, `409`, stable error codes, idempotency headers, pagination, and request IDs.

- [ ] **Step 2: Write the authorization matrix.**

Map Operator, Reviewer, and Tenant Admin to every action. State that authenticated user, active membership, permission, resource tenant, and resource state are all checked server-side.

- [ ] **Step 3: Write the security model.**

Cover tenant isolation across every persistence/transport layer, bank-data encryption/masking, upload validation, signed URLs, secret handling, untrusted document content, Agent limits, and audit events.

- [ ] **Step 4: Write failure modes and degraded operation.**

Cover storage, database, outbox, queue, worker, parser, model, schema, index, chat, citation, concurrent review, and cross-tenant failures. State retryability, user-visible result, recovery source, and telemetry signal for each.

- [ ] **Step 5: Verify authorization and failure isolation.**

Run: `rg -n "Operator|Reviewer|Tenant Admin|409 Conflict|signed URL|prompt injection|source of truth|REJECTED" capstone/enterprise-document-agent/docs/api capstone/enterprise-document-agent/docs/security`

Expected: roles, concurrency, access, recovery, and business/technical failure separation are explicit.

### Task 6: Write evaluation, observability, local, and Azure operations documents

**Files:**
- Create: `capstone/enterprise-document-agent/docs/evaluation/evaluation-strategy.md`
- Create: `capstone/enterprise-document-agent/docs/operations/local-development.md`
- Create: `capstone/enterprise-document-agent/docs/operations/observability.md`
- Create: `capstone/enterprise-document-agent/docs/operations/azure-mapping.md`

**Interfaces:**
- Consumes: quality targets from Task 3 and component boundaries from Task 4.
- Produces: reproducible evaluation design, local service topology, telemetry contract, and provider-neutral Azure migration map.

- [ ] **Step 1: Write layered evaluation strategy.**

Define synthetic dataset layout, supplier-level split, ground-truth schema, metrics and target thresholds for classification, extraction, critical fields, rules, retrieval, citations, faithfulness, refusal, tool selection, tenant isolation, and retry/idempotency.

- [ ] **Step 2: Write local-development architecture.**

Document React/Vite, FastAPI, worker, PostgreSQL 16 + pgvector, MinIO and RabbitMQ roles; environment variables; intended ports; health checks; startup ordering; and the rule that deterministic tests use mock/fake model adapters.

- [ ] **Step 3: Write observability contract.**

Define structured-log fields, forbidden log content, metrics, ingestion trace propagation, dashboards, alerts, and cost attribution dimensions.

- [ ] **Step 4: Write Azure mapping and migration gates.**

Map each local port to Container Apps, PostgreSQL, Blob, Service Bus, Document Intelligence, AI Search, Foundry/Azure OpenAI, Entra ID, Key Vault, Monitor and ACR. State that Azure begins only after local evaluation baseline and business logic cannot import Azure SDK.

- [ ] **Step 5: Verify every quality target and Azure service mapping.**

Run: `rg -n "0.90|0.85|0.95|100%|Container Apps|Service Bus|AI Search|Foundry|Entra ID|Key Vault" capstone/enterprise-document-agent/docs/evaluation capstone/enterprise-document-agent/docs/operations`

Expected: approved metrics and every local-to-Azure mapping are present.

### Task 7: Record architecture decisions

**Files:**
- Create: `capstone/enterprise-document-agent/docs/decisions/0001-modular-monolith.md`
- Create: `capstone/enterprise-document-agent/docs/decisions/0002-local-first-adapters.md`
- Create: `capstone/enterprise-document-agent/docs/decisions/0003-human-in-the-loop.md`
- Create: `capstone/enterprise-document-agent/docs/decisions/0004-deterministic-rule-engine.md`
- Create: `capstone/enterprise-document-agent/docs/decisions/0005-read-only-agent.md`

**Interfaces:**
- Consumes: approved decisions in specification sections 2, 4, 8, 9, and 13.
- Produces: immutable decision context, alternatives, consequences, and revisit conditions for later implementation.

- [ ] **Step 1: Write ADR 0001 comparing modular monolith + worker against microservices and serverless.**
- [ ] **Step 2: Write ADR 0002 comparing local-first adapters against Azure-first and local-only.**
- [ ] **Step 3: Write ADR 0003 documenting AI recommendation and human approval boundaries.**
- [ ] **Step 4: Write ADR 0004 documenting AI extraction + deterministic rules versus LLM-only decisions.**
- [ ] **Step 5: Write ADR 0005 documenting read-only Agent tools and future write-tool approval requirements.**
- [ ] **Step 6: Verify every ADR includes Status, Context, Decision, Alternatives, Consequences, and Revisit conditions.**

Run: `for f in capstone/enterprise-document-agent/docs/decisions/*.md; do rg -q "## Status" "$f" && rg -q "## Context" "$f" && rg -q "## Decision" "$f" && rg -q "## Alternatives" "$f" && rg -q "## Consequences" "$f" && rg -q "## Revisit" "$f"; done`

Expected: loop exits 0.

### Task 8: Connect navigation, validate M0, and commit

**Files:**
- Modify: `capstone/enterprise-document-agent/README.md`
- Modify: root `README.md`
- Validate: every file created in Tasks 1–7

**Interfaces:**
- Consumes: complete M0 documentation set.
- Produces: navigable GitHub documentation, verified project boundary, and one focused local commit.

- [ ] **Step 1: Add a documentation index and milestone checklist to the capstone README.**
- [ ] **Step 2: Update the learning-repository README to link to the canonical capstone without making the capstone depend on parent files.**
- [ ] **Step 3: Check internal Markdown links and required files.**

Run: `rg -o '\]\([^)#]+\.md\)' capstone/enterprise-document-agent -g '*.md' | sed -E 's/.*\]\(([^)]+)\)/\1/' | sort -u`

Resolve each relative link from its source file and confirm the target exists.

- [ ] **Step 4: Run full M0 validation.**

Run: `find capstone/enterprise-document-agent -type f | sort && git diff --check && git status --short`

Expected: all planned files exist, no whitespace errors occur, and status contains only intended M0 files plus previously known roadmap changes.

- [ ] **Step 5: Commit only M0 files and the root README link.**

Run: `git add capstone/enterprise-document-agent README.md docs/superpowers/plans/2026-08-30-enterprise-document-agent-foundation.md && git commit -m "docs: establish enterprise document agent foundation"`

- [ ] **Step 6: Verify the commit and preserve unrelated changes.**

Run: `git show --stat --oneline HEAD && git status --short`

Expected: HEAD contains only M0 Foundation files; pre-existing roadmap changes remain unstaged and intact.
