# ADR-0005: Read-only Agent tools

## Status

Accepted — approved foundation decision dated 2026-08-30.

## Context

PDFs, OCR text, user questions, and model output are untrusted. The Agent must answer grounded case questions without bypassing authorization, exposing another tenant, mutating workflow state, changing rule severity, or making an approval decision. Evidence must be active, citation-valid, masked where sensitive, and auditable.

## Decision

The Agent is an API module with exactly seven registered read-only tools: `get_case_summary`, `list_case_documents`, `get_document_metadata`, `get_extracted_fields`, `list_validation_issues`, `search_case_evidence`, and `explain_validation_issue`. The server binds authenticated tenant/case context, enforces resource and active-document filters, validates schemas/citations, limits execution, and refuses write/state/approval requests. Deterministic workflows alone own writes and transitions. Any future write tool requires explicit approval and security work before registration: threat model, least-privilege authorization, confirmation/idempotency/concurrency design, audit lineage, tenant-isolation and prompt-injection testing, and a reviewed rollout gate.

## Alternatives

- **Agent with write or review tools now:** Not selected because untrusted model inputs and prompt injection must not reach business mutations before the explicit security and approval work is complete.
- **Generic database/object/HTTP tools:** Not selected because they would bypass bounded evidence, tenant filters, citation validation, and controlled audit paths.

## Consequences

- Benefits: bounded blast radius, grounded answers with verifiable citations, predictable refusal behavior, and preserved tenant, audit, evidence, and deterministic write boundaries.
- Costs: users cannot complete mutations through chat, and maintaining the allowlist, context binding, citation validator, redaction, and refusal tests adds API/security work.

## Revisit conditions

Revisit when a documented business use case passes the required threat model and security review, explicit role/confirmation and idempotency design, tenant-isolation/prompt-injection regression suite, immutable audit design, and staged evaluation with no unresolved high-severity findings.
