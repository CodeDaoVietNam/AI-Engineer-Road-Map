# Using System Design Primer

Use [System Design Primer](https://github.com/donnemartin/system-design-primer) as a foundation for distributed-systems concepts. Do not try to read it from start to finish.

For each relevant concept, follow this loop:

1. Read the concept, such as caching, queues, replication, or consistency.
2. Ask which problem it solves in the Enterprise Document RAG Assistant.
3. Create a diagram, a small implementation, or a design note.
4. Measure the relevant effect where possible.
5. Record the trade-off in a decision record.

Prioritize latency and throughput, storage, queues, caching, reliability, security, and access-controlled retrieval. Begin with a modular monolith and an ingestion worker; add complex infrastructure only when there is a documented reason.
