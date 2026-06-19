# ConfigEngine

ConfigEngine (CE) is a microservice that stores, versions, and resolves
per client configurations. It publishes full or partial config updates to a
Kafka topic for downstream delivery to client.

## Contents

- [Domain Glossary](#domain-glossary)
- [Functional Requirements](#functional-requirements)
- [Non Functional Requirements](#non-functional-requirements)
- [Engineering Guidelines](.github/copilot-instructions.md)
- [Testing Guidelines](.github/instructions/testing.instructions.md)

## Domain Glossary

- **Config**: An immutable, versioned configuration record scoped to one
  client.
- **Config Type / Subtype**: Classifies a config. A client may hold many types
  at once.
- **additionalIdentifier**: Optional field used when type and subtype cannot
  uniquely identify a config.
- **Resolved Config**: The single active config chosen for a client and type.
  The most recently created version is active.
- **Delta**: A proto FieldMask describing only the fields changed since the
  last published version.

## Functional Requirements

1. **Versioning**: Every push creates a new immutable version. Versions
   increase monotonically per client and config type.
2. **Config Types**: A config has a type and an optional subtype. Use
   `additionalIdentifier` only when type and subtype are insufficient.
3. **Client Scoping**: Every config is scoped to a specific client identifier.
4. **Resolution**: Among versions of the same type for a client, the most
   recently created version is active.
5. **Full Publish**: Publish the complete resolved config for a client and type
   to the Kafka delivery topic.
6. **Delta Publish**: Publish only changed fields between the previous published
   version and the current resolved config, as a proto FieldMask.

## Non Functional Requirements

1. **Consistency (CP)**: Prefer strong consistency under CAP. If CE cannot
   deliver, the client retries with exponential backoff.
2. **Concurrency**: All writes must be safe under concurrent access using
   optimistic locking.
3. **Delivery**: Kafka delivery is at least once. Updates must survive a CE
   crash mid publish.
4. **Scalability**: CE scales horizontally by pod count. Distributed locking
   keeps per client operations correct across replicas.
5. **Retention**: Purge versions older than 30 days. Never delete the active
   version, regardless of age.
