# Enterprise Document Agent

Enterprise Document Agent helps an enterprise receive, inspect, and review
supplier-onboarding records. It processes documents asynchronously, extracts
structured data and evidence, applies deterministic validation rules, and
supports grounded questions with citations. The final decision always belongs
to a human reviewer.

## MVP boundary

Each supplier case accepts these five English PDF document types:

1. `company_profile`
2. `business_registration`
3. `tax_registration`
4. `bank_information_form`
5. `quotation`

Each file is limited to 20 MB and 50 pages. Corrupted, password-protected, or
invalid files are rejected clearly.

The product has three user roles:

- **Operator** creates suppliers and cases, uploads or replaces documents, and
  views processing results.
- **Reviewer** has Operator permissions and may approve or reject a case.
- **Tenant Admin** manages memberships, views audit records, and has all review
  permissions.

AI may classify documents, extract fields, find evidence, and raise alerts.
Deterministic workflows are the sole owners of writes and state transitions.
The deterministic rule engine owns pass/fail evaluation. Only a Reviewer or
Tenant Admin can approve or reject a case; the agent is read-only and cannot
change case data or state.

## Local-first architecture

This self-contained capstone is a modular monolith with an asynchronous worker:
React + TypeScript + Vite for `web`, FastAPI for `api`, and a shared-core
ingestion `worker`. Local development uses PostgreSQL 16 with pgvector, MinIO,
RabbitMQ, local JWT, and model-provider adapters. Azure and Microsoft Foundry
credentials are never required for local work; Azure support is provided later
through adapters.

The capstone must not import code from `learning-paths/`, `resources/`, or any
directory outside this root.

## Repository map

The following structure is introduced progressively during M0. All application
source, dependencies, tests, infrastructure, and project documentation remain
inside this directory so it can later become its own repository.

```text
enterprise-document-agent/
├── apps/                 # React web, FastAPI API, and ingestion worker
├── packages/             # core domain, contracts, and infrastructure adapters
├── migrations/           # database migrations
├── infrastructure/       # local and Azure deployment assets
├── evaluation/           # datasets, ground truth, metrics, and generated reports
├── tests/                # integration, end-to-end, security, and performance tests
├── docs/                 # product, architecture, API, security, and operations docs
├── compose.yml           # local service topology (added in M0)
├── Makefile              # developer commands (added in M0)
├── .env.example          # non-secret local configuration names
└── AGENTS.md             # contributor constraints
```

## Milestones

1. **M0 Foundation** — repository boundary, documentation, Compose, CI, and health checks.
2. **M1 Core domain** — local auth, tenancy/RBAC, suppliers, cases, documents, and migrations.
3. **M2 Reliable ingestion** — storage, outbox, RabbitMQ, worker, retries, checkpoints, and idempotency.
4. **M3 Document intelligence** — classification, extraction, evidence, normalization, and validation.
5. **M4 RAG and Agent** — chunking, hybrid retrieval, reranking, citation validation, and read-only tools.
6. **M5 Human review** — dashboard, PDF highlights, review decisions, optimistic locking, and audit.
7. **M6 Evaluation and hardening** — synthetic data, evaluation, security/failure/load tests, and telemetry.
8. **M7 Azure deployment** — infrastructure as code, Azure adapters, identity, secrets, CI/CD, monitoring, and cost controls.

## Milestone-target commands

The commands below are targets for **after M1**. They are intentionally not
executable yet because this M0 task creates documentation and scaffolding only,
not application code or build tooling.

```bash
cp .env.example .env
docker compose up -d
make api-dev
make worker-dev
make web-dev
make test
```

When these commands are implemented, local configuration must remain
non-secret, and validation must include tenant-isolation tests.
