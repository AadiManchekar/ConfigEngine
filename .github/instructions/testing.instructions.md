---
description: Feature based testing conventions for ConfigEngine.
applyTo: "test/**"
---

# Testing Guidelines

These rules apply to all test code under `test/`.

1. All tests are feature based under `test/resources/features`.
2. A feature may have many scenarios. Each scenario lives in its own folder
   with `input` and `expected` subfolders.
3. Each feature has a `README.md` describing the feature and listing its
   scenarios.

## Folder Layout

```text
test/resources/features/
  <feature-name>/
    README.md
    <scenario-name>/
      input/
      expected/
```
