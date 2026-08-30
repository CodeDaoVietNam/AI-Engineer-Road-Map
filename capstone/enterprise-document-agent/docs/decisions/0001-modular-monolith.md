# ADR-0001: Modular monolith with asynchronous worker

## Status

Accepted — approved foundation decision dated 2026-08-30.

## Context

The MVP must run the API, web application, and ingestion worker with shared domain rules. Tenant authorization, audit lineage, evidence/citation scope, and deterministic write ownership must remain consistent across upload, processing, validation, and review. The queue is a delivery mechanism; PostgreSQL and object storage remain recovery sources of truth.

## Decision

Use a modular monolith with a stateless API and asynchronous worker. Keep domain and application use cases in shared core packages, ports/contracts independent of infrastructure, and the Agent as a read-only API module. Deterministic workflows alone own writes and state transitions; every record, message, query, cache key, and tool call is tenant-scoped and auditable.

## Alternatives

- **Microservices:** Not selected for the MVP because independently deployed domains would duplicate or distribute invariants, authorization, audit, and write ownership before operational evidence justifies that complexity.
- **Serverless-first:** Not selected because queue-driven ingestion, long-running parsing/model work, local reproducibility, and explicit checkpoint/idempotency controls need a stable worker boundary.

## Consequences

- Benefits: one shared rule implementation, clear module boundaries, simpler local operation, durable asynchronous processing, and consistent tenant/audit/evidence controls.
- Costs: the monolith requires disciplined dependency direction and module ownership; API and worker releases remain coupled, and a worker/API fault domain is broader than separately deployed services.

## Revisit conditions

Revisit when measured worker or API scaling requires independent deployment, when a module has a separately owned release cadence and stable contract, or when operational telemetry shows sustained resource contention that cannot be addressed within the worker/API split.
