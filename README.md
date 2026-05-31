# ConfigEngine

ConfigEngine (CE) stores, versions, and resolves configurations per device. CE publishes full or partial config updates to a Kafka topic for downstream delivery to the device.

## Functional Requirements

1. **Config Versioning**: Each config push creates a new immutable version.
   Versions are monotonically increasing per device and config type.
2. **Config Types**: Each config belongs to a type and optional subtype.
   A device may hold multiple config types simultaneously.
   An optional `additionalIdentifier` field exists for cases where
   type + subtype is insufficient to uniquely identify a config.
3. **Device Scoping**: Every config is scoped to a specific device identifier.
4. **Priority Resolution**: When multiple configs of the same type exist for
   a device, only the highest priority config is considered active.
   On a priority tie, the most recently created version wins.
5. **Full Config Publish**: Publish the complete resolved config for a device
   and config type to the Kafka delivery topic.
6. **Partial Config Publish (Delta)**: Publish only the fields that changed
   between the previous published version and the current resolved config,
   expressed as a proto FieldMask delta.

## Non-Functional Requirements

1. **CP Consistency Model**: Prioritize Strong consistency (CP under CAP theorem).
   If CE fails to deliver config, a device can have exponential backoff to request config.
2. **Concurrency & Optimistic Locking**: All write operations must be safe
   under concurrent access.
3. **At-Least-Once Kafka Delivery**: Config updates must reach Kafka even
   if CE crashes mid-publish.
4. **Horizontal Scalability**: CE scales by increasing pod count.
   Distributed locking ensures correctness across replicas for operations
   on the same device.
5. **Retention**: Config versions older than 30 days are purged. The
   currently active version is never deleted regardless of age.

## Setup

See [SETUP.md](SETUP.md) for prerequisites and local development instructions.

## Documentation

- [High-Level Design](docs/architecture/HLD.md)
- [Architectural Decision Records](docs/adr/README.md)
- [Knowledge Bank](knowledge-bank.md)
- [Config Proto Guide](config/README.md)
- [Agent Guidance](AGENTS.md)
