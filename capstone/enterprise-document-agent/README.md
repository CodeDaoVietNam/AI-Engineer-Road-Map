# Enterprise Document Agent

The Enterprise Document Agent is a local-first foundation for receiving,
inspecting, and reviewing supplier-onboarding records. A supplier case accepts
five English PDF document types: `company_profile`, `business_registration`,
`tax_registration`, `bank_information_form`, and `quotation`. Documents are
processed asynchronously, structured fields and evidence are extracted, and
deterministic validation rules surface issues for a human decision-maker.
Agents may classify, extract, retrieve, cite, and raise alerts, but they are
read-only. Only a Reviewer or Tenant Admin can approve or reject a case.

## Current status: M0 Foundation

M0 is the documentation and boundary foundation. The repository shape,
requirements, API and security contracts, architecture, operations, evaluation
plan, and accepted architecture decisions are in place. Application code,
React/FastAPI runtime wiring, migrations, Compose services, and build tooling
are intentionally not implemented yet; command snippets in the linked
operations documents are targets for later milestones, not a claim of a
runnable system today.

## Source and package map

All project-owned files stay below this directory so the capstone can be
extracted as a self-contained repository.

```text
enterprise-document-agent/
├── apps/                 # planned web, API, and asynchronous worker boundaries
├── packages/             # core domain, contracts, and infrastructure adapters
├── docs/                 # requirements, design, contracts, operations, ADRs
├── infrastructure/       # local and Azure deployment boundaries
├── evaluation/           # datasets, ground truth, metrics, and reports
├── migrations/           # database migration boundary
├── tests/                # integration, end-to-end, security, performance
├── .env.example          # safe configuration names only
└── AGENTS.md             # contributor and boundary constraints
```

The intended runtime is a modular monolith with an asynchronous worker. The
web app consumes published API contracts; the API and worker compose core,
contracts, and adapters; and `packages/core` never imports framework, queue,
storage-SDK, or Azure implementation details.

## M0 documentation index

### Requirements

- [Product requirements](docs/requirements/product-requirements.md)
- [Functional requirements](docs/requirements/functional-requirements.md)
- [Non-functional requirements](docs/requirements/non-functional-requirements.md)

### Architecture and data model

- [System overview](docs/architecture/system-overview.md)
- [Ingestion pipeline](docs/architecture/ingestion-pipeline.md)
- [Extraction and validation](docs/architecture/extraction-validation.md)
- [Retrieval and Agent](docs/architecture/retrieval-agent.md)
- [Domain model](docs/data-model/domain-model.md)
- [State machines](docs/data-model/state-machines.md)

### API and security

- [API contract](docs/api/api-contract.md)
- [Authorization matrix](docs/api/authorization-matrix.md)
- [Security model](docs/security/security-model.md)
- [Failure modes](docs/security/failure-modes.md)

### Evaluation and operations

- [Evaluation strategy](docs/evaluation/evaluation-strategy.md)
- [Local development](docs/operations/local-development.md)
- [Observability](docs/operations/observability.md)
- [Azure mapping](docs/operations/azure-mapping.md)

### Architecture decisions

- [ADR 0001: Modular monolith](docs/decisions/0001-modular-monolith.md)
- [ADR 0002: Local-first adapters](docs/decisions/0002-local-first-adapters.md)
- [ADR 0003: Human-in-the-loop](docs/decisions/0003-human-in-the-loop.md)
- [ADR 0004: Deterministic rule engine](docs/decisions/0004-deterministic-rule-engine.md)
- [ADR 0005: Read-only Agent](docs/decisions/0005-read-only-agent.md)

### Boundary guides

- [Contributor constraints](AGENTS.md)
- [API app boundary](apps/api/README.md)
- [Web app boundary](apps/web/README.md)
- [Worker boundary](apps/worker/README.md)
- [Core package boundary](packages/core/README.md)
- [Contracts package boundary](packages/contracts/README.md)
- [Adapters package boundary](packages/adapters/README.md)
- [Local infrastructure boundary](infrastructure/local/README.md)
- [Azure infrastructure boundary](infrastructure/azure/README.md)

## Milestone checklist

- [x] **M0 Foundation:** establish the repository boundary and document
  requirements, architecture, contracts, operations, evaluation, and ADRs.
- [ ] **M1 Core domain:** implement local authentication, tenancy/RBAC,
  suppliers, cases, documents, and migrations.
- [ ] **M2 Reliable ingestion:** implement storage, outbox, RabbitMQ,
  checkpoints, retries, and idempotency.
- [ ] **M3 Document intelligence:** implement classification, extraction,
  evidence, normalization, and deterministic validation.
- [ ] **M4 RAG and Agent:** implement hybrid retrieval, reranking, citation
  validation, and read-only tools.
- [ ] **M5 Human review:** implement the dashboard, PDF highlights, review
  decisions, optimistic locking, and audit views.
- [ ] **M6 Evaluation and hardening:** add synthetic evaluation, security,
  failure, load, and telemetry tests.
- [ ] **M7 Azure deployment:** add Azure adapters, infrastructure, identity,
  secrets, CI/CD, monitoring, and cost controls after local gates pass.

## Local first, Azure later

Local development is the baseline: PostgreSQL 16 with pgvector, MinIO,
RabbitMQ, local JWT, and deterministic fake or mock model adapters. It must not
require Azure or Microsoft Foundry credentials. Azure services are provider
implementations behind the same ports and are introduced only after the local
evaluation baseline passes; see the [Azure mapping](docs/operations/azure-mapping.md)
for the reversible migration gates and quota caveats.

## Next milestone

The next milestone is **M1 Core domain**: implement the domain entities and
use cases in `packages/core`, publish schemas and ports in `packages/contracts`,
and add the local authentication, tenant membership, supplier, case, document,
and migration behavior described by the M0 contracts.
