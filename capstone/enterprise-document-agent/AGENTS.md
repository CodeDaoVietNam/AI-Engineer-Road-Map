# Contributor instructions

## Scope and ownership

Keep every project-owned file under `capstone/enterprise-document-agent/`.
Do not create a nested Git repository or import code from `learning-paths/`,
`resources/`, or any parent directory. This is a modular monolith with an
asynchronous worker, not a collection of domain microservices.

## Naming

- Python files, functions, variables, and module names use `snake_case`.
- Python classes, protocols, and typed domain entities use `PascalCase`.
- TypeScript variables, functions, hooks, and object properties use `camelCase`.
- TypeScript components, types, interfaces, and enums use `PascalCase`.
- React component files use `PascalCase.tsx`; other TypeScript files use
  `kebab-case.ts` unless a framework convention requires another form.
- API paths use lowercase, hyphen-free plural resource names under `/api/v1`;
  event names use `PascalCase` (for example, `DocumentUploaded`).

## Dependency direction

`packages/core` owns domain entities and application use cases. It may depend
only on the standard library and `packages/contracts`; it must not import
FastAPI, SQLAlchemy, Vite/React, RabbitMQ clients, storage SDKs, Azure SDKs, or
adapter implementations. `packages/contracts` owns schemas, events, and port
interfaces. `packages/adapters` implements those ports. `apps/api` and
`apps/worker` compose core, contracts, and adapters; `apps/web` consumes only
published API contracts and must not import Python packages.

## Tests

Place focused unit tests beside the package or feature they exercise. Place
cross-boundary tests in `tests/integration`, browser workflows in
`tests/end-to-end`, authorization and isolation tests in `tests/security`, and
load/latency tests in `tests/performance`. Use deterministic fixtures and fake
model adapters for unit tests; tests that need infrastructure run against real
containerized services.

## Tenant isolation and review safety

Every business record, object path, queue message, search query, cache key,
log context, and agent tool call must be scoped by the authenticated tenant.
Never accept `tenant_id` from a request body as an authorization source; derive
it from authenticated membership and verify resource ownership. Agent tools are
read-only.
Deterministic workflows are the sole owners of writes and state transitions.
Only a Reviewer or Tenant Admin may approve or reject a case.

## Secrets and local configuration

Never commit secrets, tokens, credentials, connection passwords, private keys,
or customer documents. Keep committed configuration to non-secret names and
safe defaults in `.env.example`; put local values in the ignored `.env` file.
Do not put full document text, bank-account values, sensitive prompts, or
tokens in logs, fixtures, traces, or generated reports.

## Verification

Before committing, run the narrowest relevant formatter, linter, type check,
and tests, then run the affected integration or end-to-end checks when a
boundary changes. Confirm `git diff --check` is clean and inspect `git status
--short`. Security-sensitive changes must include tenant-isolation coverage;
deterministic rule-engine cases must remain fully covered.
