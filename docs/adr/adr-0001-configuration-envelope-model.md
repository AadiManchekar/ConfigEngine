---
title: "ADR-0001: Configuration Model"
status: "Proposed"
date: "19-06-2026"
authors: "Aadi Manchekar"
tags: ["architecture", "decision", "proto", "domain-model"]
---

# ADR-0001: Configuration Model

## Status

**Proposed**

## Context

ConfigEngine resolves per client configs and publishes them to a Kafka topic
for downstream delivery. The published message is the `Configuration` proto
defined in [config/Configuration.proto](../../config/Configuration.proto).

A config is classified by a client class, a config type, and a sub type.
Keep `config_payload` opaque so CE stays agnostic to concrete config shapes,
and stay simple to extend with new client classes.

## Decision

The `Configuration` message holds a `oneof target` whose chosen branch 
identifies the client class. Each branch is a per client message 
(`AccessPointConfig`, `GatewayConfig`, `MobileConfig`,`VmConfig`) 
that carries its own scoped `Type` and `SubType` enums. Each client
message lives in its own folder under `config/`.

Key points:

- The selected `oneof` branch is the client type, so a standalone `ClientType`
  enum is not needed.
- `Type` and `SubType` are scoped inside each client message, so values from
  one client class cannot leak into another.
- The triple `(branch, type, sub_type)` is the routing key the consumer uses to
  deserialize the opaque `config_payload` bytes.
- `update_mask` stays a single `google.protobuf.FieldMask`, never repeated. One
  mask already lists every changed field path through its internal `paths`
  array.
- The Configuration carries `version` for dedup and ordering under at least once
  delivery.
- The Configuration carries `published_at`, the send time, so a client can detect
  and drop a config older than its freshness window.


## Consequences

### Positive

- **POS-001**: Each client class owns its own enums.
- **POS-003**: Adding a new client class is an additive `oneof` branch and a new
  message, aligning with the Open/Closed principle. Existing branches are
  untouched.
- **POS-004**: `config_payload` stays opaque, so CE remains agnostic to concrete
  config shapes while still giving the consumer a precise routing key.

### Negative

- **NEG-001**: The schema enforces the client to specify type and subType. 
  So there can be mismatch between type and subType. If this was to be 
  handled then that would result into another `oneof` inside the exisiting `oneof`
  defeating readability and simplicity.


## Alternatives Considered

### Flat enums with a client prefix

- **ALT-001**: **Description**: Keep three flat enums but prefix every value
  with its client class, for example `CONFIG_SUB_TYPE_AP_RADIO`.
- **ALT-002**: **Rejection Reason**: Still a single shared value space, still
  allows illegal triples, and grows unbounded. Cosmetic fix only.

### Embed client type inside the subtype enum

- **ALT-003**: **Description**: Add the client class as a field or value within
  `ConfigSubType` so a reader can tell which client it belongs to.
- **ALT-004**: **Rejection Reason**: Duplicates data already implied by the
  client class and does not prevent constructing an invalid combination. The
  team explicitly rejected this.

### Fully nested oneof for both levels

- **ALT-005**: **Description**: Use a `oneof` for client class and a second
  nested `oneof` per type to scope sub_types, enforcing both tree levels in the
  schema.
- **ALT-006**: **Rejection Reason**: Heavy and hard to read for marginal gain.
  A small boundary validator covers the type to sub_type pairing with far less
  ceremony (KISS).

## Implementation Notes

- **IMP-001**: Consumers must switch on the `target` branch first, then on
  `type` and `sub_type`, to select the payload decoder.
- **IMP-002**: Add a boundary validator that rejects sub_types not belonging to
  their `type` within a client message.
- **IMP-003**: Reserve and document `oneof` field numbers (20 onward) so future
  client classes do not collide with envelope fields (1 to 8).
- **IMP-004**: Each client config message lives in its own folder under
  `config/` (for example `config/access_point/access_point.proto`) and is
  imported by `config/Configuration.proto`.

## References

- **REF-001**: [config/Configuration.proto](../../config/Configuration.proto)
- **REF-002**: [README.md](../../README.md) Domain Glossary and Functional
  Requirements
- **REF-003**: protobuf FieldMask, `google.protobuf.FieldMask`
