# API application boundary

`api` owns the stateless FastAPI HTTP boundary and authentication context only.
It validates request shape, derives tenant and role context from authenticated
membership, authorizes resource access, and maps use-case results to HTTP
responses.

It calls shared application use cases in `packages/core`; it does not duplicate
domain logic, persistence logic, queue handlers, or adapter implementations.
The API composes `core`, `contracts`, and `adapters` for request processing.
