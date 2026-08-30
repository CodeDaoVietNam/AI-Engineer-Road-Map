# ADR-0003: Human review and approval boundaries

## Status

Accepted — approved foundation decision dated 2026-08-30.

## Context

Extraction and retrieval can assist supplier onboarding, but evidence may be incomplete or uncertain and a business approval is consequential. Review must be attributable, tenant-scoped, concurrency-safe, and reproducible from the active validation snapshot. Technical failures and Agent output must never become a business rejection.

## Decision

Use AI for recommendations, extraction, grounded explanations, and evidence navigation only. A Reviewer or Tenant Admin must inspect the active evidence/issues and explicitly approve or reject through the authorized API. The API enforces active membership, role, tenant/resource ownership, case state, active validation snapshot, zero blocking errors, optimistic version, and idempotency; it records the decision and audit lineage atomically. Deterministic workflows remain the sole owners of writes and state transitions.

## Alternatives

- **Fully automated approval:** Not selected because model uncertainty, missing evidence, and business accountability require an authorized human decision.
- **Human review without server guards:** Not selected because UI-only boundaries would permit stale, cross-tenant, or unaudited decisions.

## Consequences

- Benefits: accountable approvals, explicit evidence/citation review, safer handling of uncertainty, strong tenant isolation, and auditable/replay-safe decisions.
- Costs: reviewer time and queueing delay remain part of the workflow; the API must maintain snapshot/version/idempotency controls and a review workspace.

## Revisit conditions

Revisit only after measured decision accuracy and reviewer outcomes support a narrower automation proposal, with an approved risk assessment, role/security controls, evidence requirements, audit design, and an independent evaluation gate.
