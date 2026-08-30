# ADR-0004: AI-assisted extraction with deterministic validation

## Status

Accepted — approved foundation decision dated 2026-08-30.

## Context

Models are useful for extracting structured fields from untrusted PDFs, but validation outcomes must be repeatable and explainable. Required fields, formats, cross-document consistency, quotation calculations, confidence policy, severity, and active-version selection affect review and must retain evidence lineage and tenant scope.

## Decision

Use an AI extraction adapter to produce versioned structured values with confidence and evidence references. Normalize and validate those values with versioned deterministic code and a rule catalog. Only the deterministic rule engine decides pass/fail and fixed severity; it stores operands, rule/pipeline versions, outcomes, and evidence lineage. AI and Agent outputs cannot create, alter, resolve, or downgrade issues, writes, or state transitions.

## Alternatives

- **LLM-only decisions:** Not selected because outputs are non-deterministic, harder to reproduce, and unsuitable as the owner of severity or business state.
- **Rules-only extraction:** Not selected because varied document layouts and OCR conditions need model-assisted structure extraction while preserving deterministic validation.

## Consequences

- Benefits: reproducible rule outcomes, explainable evidence, testable severity and calculations, safer review gates, and clear ownership of writes/state.
- Costs: schema, normalizer, rule-catalog, and pipeline versions require migration/evaluation discipline; extraction errors still need reviewer handling and reprocessing.

## Revisit conditions

Revisit when evaluation shows a validated extraction/schema limitation that materially affects coverage, or when a proposed rule change cannot meet deterministic replay, evidence lineage, 100% expected rule-case coverage, and tenant/audit regression gates.
