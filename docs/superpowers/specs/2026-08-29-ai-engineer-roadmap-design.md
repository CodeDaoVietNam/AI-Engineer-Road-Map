# AI Engineer Roadmap Repository Design

## Goal

Create a GitHub-ready learning repository for an AI Engineer intern building RAG, chatbot, and AI Agent systems. It must turn two Vietnamese roadmaps into one practical progression: first build a production-ready Document RAG Assistant through System Design, then extend it with advanced AI engineering, Azure Cloud, and LLMOps capabilities.

## Learning Sequence

The two roadmaps have different roles and must not be studied in parallel as two full-time curricula.

1. **Primary path — System Design for AI Engineer (8 weeks):** Learn requirements, APIs, data design, ingestion, retrieval, reliability, security, and deployment by building an end-to-end Document RAG Assistant.
2. **Expansion path — Advanced AI Engineer and Cloud (20 weeks):** After the initial system runs end to end, expand it with model foundations, LLM application engineering, agents, MCP, evaluation, multimodal processing, fine-tuning, inference serving, and Azure operations.

Each study session has one main topic. The expansion roadmap may supply a small, directly relevant supplement during the first eight weeks—for example, Azure Blob Storage while learning object storage—but not a separate full lesson.

## Repository Layout

```text
.
├── README.md
├── roadmap/
│   ├── AI_ENGINEER_SYSTEM_DESIGN_ROADMAP.md
│   └── ADVANCED_AI_ENGINEER_AND_CLOUD_ROADMAP.md
├── learning-paths/
│   ├── 00-setup/
│   ├── 01-system-design-rag/                 # primary 8-week path
│   │   ├── week-00-environment/
│   │   ├── week-01-requirements-estimation/
│   │   ├── week-02-api-scalability/
│   │   ├── week-03-data-storage/
│   │   ├── week-04-ingestion-queues/
│   │   ├── week-05-retrieval-design/
│   │   ├── week-06-performance-cost/
│   │   ├── week-07-reliability-security-observability/
│   │   └── week-08-deployment-portfolio/
│   └── 02-advanced-ai-cloud/                 # follow-on 20-week path
│       ├── 01-model-foundations/
│       ├── 02-llm-application-engineering/
│       ├── 03-agents-mcp-security/
│       ├── 04-evaluation-observability/
│       ├── 05-multimodal-retrieval/
│       ├── 06-fine-tuning-serving/
│       ├── 07-cloud-azure-foundations/
│       └── 08-iac-cicd-reliability-finops/
├── capstone/
│   └── enterprise-document-rag-assistant/
│       ├── app/
│       ├── docs/
│       ├── evaluation/
│       ├── experiments/
│       └── infrastructure/
├── docs/
│   ├── decisions/
│   ├── diagrams/
│   ├── system-design/
│   ├── portfolio/
│   └── superpowers/
├── resources/
│   ├── system-design-primer.md
│   ├── ai-engineering.md
│   └── cloud-azure.md
├── templates/
│   ├── learning-session.md
│   ├── weekly-module.md
│   ├── architecture-decision-record.md
│   ├── experiment.md
│   └── project-readme.md
├── .github/
├── .gitignore
├── CONTRIBUTING.md
└── LICENSE
```

## Module Contract

Every weekly or advanced module has the following layout, using `.gitkeep` to make empty directories visible in GitHub:

```text
module-name/
├── README.md
├── notes.md
├── labs/
├── artifacts/
├── measurements.md
├── decisions.md
├── reflection.md
└── resources.md
```

Completion requires four outputs:

1. **Knowledge note:** Explain the concept in personal words.
2. **Artifact:** Add code, a diagram, a dataset sample, an API design, or infrastructure configuration.
3. **Measurement:** Record a meaningful quality, latency, throughput, error, or cost observation.
4. **Decision:** State the selected approach, alternatives, trade-offs, and the condition that would prompt reassessment.

## Capstone Boundaries

The single capstone evolves throughout both paths. The first stage is a modular monolith with an API application, ingestion worker, PostgreSQL, object storage, vector search, and a managed LLM/embedding provider. It supports authenticated document upload, asynchronous ingestion, access-controlled retrieval, citations, and evaluation.

Later modules may extend it with agents and MCP tools, multimodal document processing, self-hosted inference, Azure services, IaC, CI/CD, observability, reliability testing, and FinOps. The repository deliberately avoids premature microservices and large-scale infrastructure; new components require a documented reason.

## Study Workflow

`templates/learning-session.md` records the topic, available time, learning mode, unanswered questions, practice, and end-of-session understanding check. The weekly README defines prerequisites, outcomes, labs, expected artifacts, and definition of done.

The root README provides a progress checklist, navigation to the two learning paths, capstone milestones, and contribution conventions. `resources/system-design-primer.md` explains how to use System Design Primer as supporting material: read a concept, apply it to the capstone, draw or implement the relevant flow, measure it, and document the trade-off.

## GitHub Readiness

`.gitignore` excludes Python bytecode, virtual environments, Jupyter checkpoints, environment files and secrets, editor metadata, and operating-system artifacts. `CONTRIBUTING.md` documents the module contract and naming convention. `.github/` is reserved for issue and pull-request templates.

The workspace will be initialized as a local Git repository and receive an initial commit only after structure and documentation links are validated. Connecting or pushing to GitHub requires a destination repository or the user's authorization to create one.

## Validation

Validate the scaffold by checking all required paths, confirming links in Markdown resolve locally, verifying the module template exposes all four required outputs, checking ignore behavior for representative secret and Python artifacts, and inspecting the initial Git status.
