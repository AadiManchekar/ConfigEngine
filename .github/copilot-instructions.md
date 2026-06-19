# ConfigEngine (CE)

ConfigEngine stores, versions, and resolves per device configurations. It
publishes full or partial config updates to a Kafka topic for downstream
delivery to devices.

The product spec lives in the [root README](../README.md): domain glossary,
functional requirements, and non functional requirements. Read it before
changing behavior.

When working in this repo, follow the principles below. They are mandatory, not
advisory. If a request conflicts with them, flag the conflict before
proceeding.

## Engineering Principles

Apply these at the stage noted. They override personal preference.

### Always

1. **Murphy's Law**: Anything that can go wrong will. Design for failure, add
   retries and fallbacks, and test failure paths.
2. **KISS**: Choose the simplest design that works. Avoid needless complexity
   and keep code readable and maintainable.
3. **Write like a developer**: Use plain, natural language in code, comments,
   and docs. Avoid AI tell formatting such as em dashes, semicolons as
   separators, and curly quotes. Prefer short sentences and proper grammar.

### Architectural Design

1. **CAP trade off**: A distributed system guarantees only two of Consistency,
   Availability, and Partition Tolerance. A healthy network gives all three,
   but a partition forces a choice. CE chooses consistency.
2. **Fallacies of Distributed Computing**: Never assume the network is
   reliable, latency is zero, bandwidth is infinite, the network is secure,
   topology is stable, there is one administrator, transport is free, or the
   network is homogeneous.

### Implementation

1. **SOLID**:
   - **Single Responsibility**: one reason to change per class.
   - **Open/Closed**: open for extension, closed for modification.
   - **Liskov Substitution**: subtypes replace base types without breaking
     correctness.
   - **Interface Segregation**: clients depend only on what they use.
   - **Dependency Inversion**: depend on abstractions, not concretions.
2. **Least surprise**: Code and interfaces should behave the way other
   developers expect.
3. **Loose coupling**: Each class knows as little as possible about others.
4. **No regressions**: A fix in one module must not break another.
5. Deletion over addition. Boring over clever. Fewest files possible.
6. Before writing any code, stop at the first rung that holds:
    - **Rung 1**: Does the standard library already do this? Use it.
    - **Rung 2**: Does a native platform feature cover it? Use it.
    - **Rung 3**: Does an already-installed dependency solve it? Use it.
    - **Rung 4**: Write the minimum code that works, only when rungs 1 to 3 do not apply.


### Testing

Testing conventions live in their own path scoped file:
[testing.instructions.md](instructions/testing.instructions.md).




