# Capability: Spec Domain

## Purpose

Provides CLI commands for managing the `spx/` specification tree following the CODE framework. Enables AI agents and developers to create, navigate, validate, and track specs with deterministic, sub-100ms responses.

## Success Metric

- **Baseline**: Manual directory creation, DONE.md boolean tracking, no staleness detection
- **Target**: CLI-managed specs with pass.csv ledger, blob-based staleness, deterministic status
- **Measurement**: All spec operations complete in <100ms; `spx spec test --stamp` validates pass.csv integrity

## Requirements

The spec domain implements the CODE framework for specification management:

### Structure Management

- Directory naming: `{BSP}-{slug}.{type}/` (e.g., `31-spec-domain.capability/`)
- Spec file naming: `{slug}.{type}.md` (e.g., `spec-domain.capability.md`)
- ADRs interleaved: `{BSP}-{slug}.adr.md` at any level
- Tests co-located: `<container>/tests/` with level-appropriate suffixes

### Status Determination

Status is derived from `pass.csv` ledger state, not boolean flags:

| Condition                             | State     | Required Action               |
| ------------------------------------- | --------- | ----------------------------- |
| Test Files links don't resolve        | Unknown   | Write tests                   |
| Tests exist, not all passing          | Pending   | Fix code or fix tests         |
| Spec or test blob changed since stamp | Stale     | Re-stamp with `spx spec test` |
| All tests pass, blobs unchanged       | Passing   | None—potential realized       |
| Was passing, now fails, blobs same    | Regressed | Investigate and fix           |

### pass.csv Ledger

Machine-verifiable proof linking specs to test evidence:

```csv
# spec_blob,a3f2b7c...
# run,2026-01-28T14:15:00Z
test_file,test_blob,pass_time
parsing.unit.test.ts,1f2e...,2026-01-27T10:30:00Z
```

- `spec_blob`: Git blob SHA of spec file when ledger was stamped
- `test_blob`: Git blob SHA of test file when it last passed
- Staleness detected by blob comparison, not timestamps

### BSP Numbering

Two-digit (10-99) encoding dependency order:

- Lower number = must complete first
- First item: start at 21
- Insert between X and Y: `floor((X + Y) / 2)`
- Append after X: `floor((X + 99) / 2)`
- Same BSP = parallel work allowed

### Validation

Precommit is primary feedback loop:

1. **Phantom check**: Every test_file in pass.csv must exist
2. **Regression check**: Listed tests must pass; unchanged blob + failure = regression
3. **Staleness check**: spec_blob mismatch = re-stamp required
4. **Progress tests**: Tests not in pass.csv are in-progress (not an error)

## Test Strategy

| Component          | Level | Harness     | Rationale                          |
| ------------------ | ----- | ----------- | ---------------------------------- |
| BSP calculation    | 1     | -           | Pure arithmetic, no I/O            |
| Directory parsing  | 1     | -           | Pure string parsing                |
| pass.csv parsing   | 1     | -           | Pure CSV/blob parsing              |
| Status derivation  | 1     | -           | Pure logic from parsed state       |
| Spec creation      | 2     | cli-harness | Needs real filesystem + git        |
| Validation         | 2     | cli-harness | Needs real spx binary + test files |
| Full spec workflow | 3     | e2e-harness | Needs real project with git repo   |

### Escalation Rationale

- **1 → 2**: Level 1 tests pure logic; Level 2 confirms filesystem operations work correctly with real directories and git blob computation
- **2 → 3**: Level 2 tests individual commands; Level 3 confirms full agent workflow (create → implement → stamp → validate) works end-to-end

## Outcomes

### 1. Agent finds next incomplete spec

```gherkin
GIVEN a spx/ tree with mixed status specs
WHEN agent runs `spx spec next`
THEN exactly one fully-qualified path is returned
AND it is the lowest-BSP item not Passing
AND response time is <100ms
```

| File                                         | Level | Harness                                               |
| -------------------------------------------- | ----- | ----------------------------------------------------- |
| [spec-next.e2e](tests/spec-next.e2e.test.ts) | 3     | [e2e-harness](spx/13-test-infrastructure.capability/) |

---

### 2. Agent creates spec with correct BSP

```gherkin
GIVEN a capability with features up to 54-foo.feature/
WHEN agent runs `spx spec create feature` in that capability
THEN a new feature directory is created with BSP 76
AND the directory contains template feature.md
AND response time is <100ms
```

| File                                             | Level | Harness                                               |
| ------------------------------------------------ | ----- | ----------------------------------------------------- |
| [spec-create.e2e](tests/spec-create.e2e.test.ts) | 3     | [e2e-harness](spx/13-test-infrastructure.capability/) |

---

### 3. Validation detects regression

```gherkin
GIVEN a pass.csv with test entry (unchanged spec_blob, unchanged test_blob)
WHEN that test fails
THEN `spx spec test` reports regression
AND exit code is non-zero
AND output identifies the regressed test
```

| File                                           | Level | Harness                                               |
| ---------------------------------------------- | ----- | ----------------------------------------------------- |
| [regression.e2e](tests/regression.e2e.test.ts) | 3     | [e2e-harness](spx/13-test-infrastructure.capability/) |

---

### 4. Validation detects staleness

```gherkin
GIVEN a pass.csv with spec_blob from previous commit
WHEN spec file is modified (different blob)
THEN `spx spec test` reports stale
AND suggests re-stamp with `spx spec test --stamp`
```

| File                                         | Level | Harness                                               |
| -------------------------------------------- | ----- | ----------------------------------------------------- |
| [staleness.e2e](tests/staleness.e2e.test.ts) | 3     | [e2e-harness](spx/13-test-infrastructure.capability/) |

---

### 5. Stamp records passing tests

```gherkin
GIVEN a container with tests/ directory containing passing tests
WHEN agent runs `spx spec test --stamp <container>`
THEN pass.csv is created/updated
AND spec_blob matches current spec file
AND each passing test has test_blob and pass_time recorded
```

| File                                 | Level | Harness                                               |
| ------------------------------------ | ----- | ----------------------------------------------------- |
| [stamp.e2e](tests/stamp.e2e.test.ts) | 3     | [e2e-harness](spx/13-test-infrastructure.capability/) |

---

## Architectural Constraints

| ADR       | Constraint                                                 |
| --------- | ---------------------------------------------------------- |
| (pending) | Blob-based staleness detection over timestamp comparison   |
| (pending) | pass.csv as sole source of truth for verification state    |
| (pending) | BSP numbering with hyphen separator (`21-foo.capability/`) |
