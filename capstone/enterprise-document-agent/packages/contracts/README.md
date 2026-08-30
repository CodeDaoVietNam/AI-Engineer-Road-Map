# Contracts package boundary

`contracts` owns schemas, event contracts, and port interfaces. It expresses
the stable boundary used by `core` and implemented by `adapters`.

The approved ports are:

- `ObjectStorage`
- `MessageQueue`
- `DocumentParser`
- `ExtractionModel`
- `EmbeddingModel`
- `SearchIndex`
- `LanguageModel`
- `IdentityProvider`
- `AuditPublisher`
