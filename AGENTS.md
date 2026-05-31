# AGENTS.md

Guidance for AI agents working in this repository.

## Project Overview

Refer [README.md](README.md)

## Architecture

- **Language**: Java 21
- **Framework**: Spring boot 4.x
- **Build tools**: Maven + Make
- **Serialization and De-Serialization**: Protocol Buffers
- **Message Broker**: Apache Kafka
- **Database**: PostgreSQL
- **Caching**: Redis
- **Containerisation**: Docker
- **Orchestration**: Kubernetes (multi-pod)
- Favors **Consistency + Partition Tolerance**
- **HLD**: CQRS + Transactional Outbox

### CQRS Pipeline

- **Write Pipeline**: Topic A (`ce.config.ingest`) → Write Service (`ce-ingestor`) → PostgreSQL + Transactional Outbox → Topic B
- **Read Pipeline**: Topic B (`ce.config.requests`) → Read Service (`ce-distributor`) → Resolve + Delta → Topic C (`ce.config.delivery`)
- **Downstream**: Topic C → RelayEngine (Different side project, not included in ConfigEngine) → Devices

## Kafka Topics

| Topic | Name | Purpose |
|---|---|---|
| A | `ce.config.ingest` | Upstream services publish config bundles for storage |
| B | `ce.config.requests` | Requests to CE to generate and publish resolved configs |
| C | `ce.config.delivery` | Resolved configs (full/partial) per device for RelayEngine |

## Build Commands

```bash
# Build the project
make build

# Run tests
make test

# Package Docker image
make docker-build

# Run locally
make run
```

## Project Structure (Multi-Module)

```
ConfigEngine/
├── AGENTS.md
├── README.md
├── SETUP.md
├── knowledge-bank.md
├── Makefile
├── pom.xml                    # Parent POM (inherits spring-boot-starter-parent)
├── .editorconfig
├── .github/
│   ├── instructions/          # Copilot instruction files
│   └── skills/                # Copilot skill definitions
├── docs/
    ├── architecture/
    │   └── HLD.md            # High-level design
    └── adr/                  # Architectural Decision Records

```

## Code Style

- Class names: `UpperCamelCase` (nouns: `ConfigVersion`, `DeviceRegistry`)
- Methods: `lowerCamelCase` (verbs: `resolveConfig`, `publishUpdate`)
- Constants: `UPPER_SNAKE_CASE`
- Packages: all lowercase, no underscores
- Use Java Records for DTOs
- Adhere to OOP and design patterns

## Testing

- Unit tests: JUnit 6 + AssertJ + Mockito
- Integration tests: Testcontainers (Kafka, DB)
- Test class naming: `*Test.java` for unit, `*IT.java` for integration
- Run all: `make test`

## Concurrency

CE runs across multiple pods. Code must be:

- Thread-safe: no shared mutable state across requests
- Stateless at the application tier
- Using optimistic locking or distributed locks for write conflicts

## Formatting and linting

Formatting and linting are handled by:

1. `.editorconfig`: enforced by IDE, handles indentation, line endings, and charset.
2. `fmt-maven-plugin`: auto formats during `mvn compile` (`process-sources` phase)

