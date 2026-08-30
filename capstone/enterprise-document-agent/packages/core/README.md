# Core package boundary

`core` owns framework-independent domain entities, deterministic business
rules, and application use cases. It may depend only on the standard library
and the contracts package.

Do not import FastAPI, SQLAlchemy, React, Vite, RabbitMQ clients, storage SDKs,
Azure SDKs, or adapter implementations here. Application processes invoke core
use cases; core never owns HTTP or queue handling.
