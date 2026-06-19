# ConfigEngine

ConfigEngine (CE) is a microservice that stores, versions, and resolves
per device configurations. It publishes full or partial config updates to a
Kafka topic for downstream delivery to devices.

## Contents

- [Domain Glossary](#domain-glossary)
- [Functional Requirements](#functional-requirements)
- [Non Functional Requirements](#non-functional-requirements)
- [Engineering Guidelines](.github/copilot-instructions.md)
- [Testing Guidelines](.github/instructions/testing.instructions.md)

## Domain Glossary

- **Config**: An immutable, versioned configuration record scoped to one
  device.
- **Config Type / Subtype**: Classifies a config. A device may hold many types
  at once.
- **additionalIdentifier**: Optional field used when type and subtype cannot
  uniquely identify a config.
- **Resolved Config**: The single active config chosen for a device and type
  after priority resolution.
- **Delta**: A proto FieldMask describing only the fields changed since the
  last published version.

## Functional Requirements

1. **Versioning**: Every push creates a new immutable version. Versions
   increase monotonically per device and config type.
2. **Config Types**: A config has a type and an optional subtype. Use
   `additionalIdentifier` only when type and subtype are insufficient.
3. **Device Scoping**: Every config is scoped to a specific device identifier.
4. **Priority Resolution**: Among configs of the same type for a device, the
   highest priority one is active. On a tie, the most recently created version
   wins.
5. **Full Publish**: Publish the complete resolved config for a device and type
   to the Kafka delivery topic.
6. **Delta Publish**: Publish only changed fields between the previous published
   version and the current resolved config, as a proto FieldMask.

## Non Functional Requirements

1. **Consistency (CP)**: Prefer strong consistency under CAP. If CE cannot
   deliver, the device retries with exponential backoff.
2. **Concurrency**: All writes must be safe under concurrent access using
   optimistic locking.
3. **Delivery**: Kafka delivery is at least once. Updates must survive a CE
   crash mid publish.
4. **Scalability**: CE scales horizontally by pod count. Distributed locking
   keeps per device operations correct across replicas.
5. **Retention**: Purge versions older than 30 days. Never delete the active
   version, regardless of age.
