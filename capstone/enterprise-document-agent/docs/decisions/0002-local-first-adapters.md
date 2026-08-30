# ADR-0002: Local-first ports and adapters

## Status

Accepted — approved foundation decision dated 2026-08-30.

## Context

The MVP needs a repeatable local environment without Azure credentials while retaining a credible deployment path. Storage, queue, parser, model, identity, and search integrations vary by environment, but business rules must not import provider SDKs. Tenant isolation, audit lineage, evidence/citation validity, and deterministic write ownership must be identical across deployments.

## Decision

Define provider-neutral ports in contracts and implement local adapters first (MinIO, RabbitMQ, PostgreSQL/pgvector, local JWT, and configured model/parser providers). Keep Azure adapters behind the same ports and make Azure deployment an option after the local evaluation baseline. The application derives tenant context and owns authorization, audit, evidence scope, and all writes regardless of adapter.

## Alternatives

- **Azure-first:** Not selected now because it would make local startup depend on cloud credentials and could couple business logic to Azure SDK behavior before the MVP baseline exists.
- **Local-only:** Not selected because it would discard the required production deployment path and make later provider migration a rewrite rather than an adapter substitution.

## Consequences

- Benefits: fast offline/local development, portable tests with fakes, explicit dependency direction, and a migration path that preserves tenant, audit, evidence, and deterministic workflow boundaries.
- Costs: every supported provider needs contract-compatible adapters, capability differences need explicit handling, and adapter parity/evaluation adds maintenance work.

## Revisit conditions

Revisit when the local evaluation baseline is complete and an Azure deployment has passed adapter contract, tenant-isolation, audit/evidence, and performance gates, or when an adapter capability gap blocks a measured production requirement.
