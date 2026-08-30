# Worker application boundary

`worker` owns asynchronous queue handlers only. A handler receives a verified
identifier message, establishes execution context, invokes shared application
use cases, and acknowledges or retries the message according to the use-case
outcome.

It calls shared application use cases in `packages/core`; it does not duplicate
domain logic, expose HTTP endpoints, or contain API authentication behavior.
The worker composes `core`, `contracts`, and `adapters` for asynchronous work.
