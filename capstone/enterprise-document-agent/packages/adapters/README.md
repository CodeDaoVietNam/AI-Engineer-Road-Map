# Adapters package boundary

`adapters` owns implementations of contracts ports for local and Azure
integrations. It contains infrastructure-specific behavior and translates it
to the interfaces defined in `packages/contracts`.

Adapters do not own domain rules or application use cases. Application
processes compose adapters with `core` and `contracts` at their boundaries.
