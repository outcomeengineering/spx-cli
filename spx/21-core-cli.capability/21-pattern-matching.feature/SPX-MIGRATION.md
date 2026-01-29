# SPX Migration Log: 21-pattern-matching.feature

## Migration Date

2026-01-29

## Tests Migrated

The legacy tests combined multiple stories into single files. The SPX structure separates tests by story:

| Legacy Location                                    | SPX Location                                                             | Story                           |
| -------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------- |
| `tests/unit/scanner/patterns.test.ts` (21 tests)   | Split across stories 21, 32, 43 (18 tests)                               | Capabilities, Features, Stories |
| `tests/unit/scanner/validation.test.ts` (10 tests) | `54-validate-bsp-numbers.story/tests/validation.unit.test.ts` (10 tests) | BSP Validation                  |
| `tests/unit/fixtures/factories.test.ts` (9 tests)  | `65-test-factories.story/tests/factories.unit.test.ts` (9 tests)         | Test Factories                  |

### Test Distribution in SPX

| Story                     | File                            | Tests |
| ------------------------- | ------------------------------- | ----- |
| 21-parse-capability-names | `tests/patterns.unit.test.ts`   | 5     |
| 32-parse-feature-names    | `tests/patterns.unit.test.ts`   | 5     |
| 43-parse-story-names      | `tests/patterns.unit.test.ts`   | 8     |
| 54-validate-bsp-numbers   | `tests/validation.unit.test.ts` | 10    |
| 65-test-factories         | `tests/factories.unit.test.ts`  | 9     |

## Verification

- Legacy tests: 40 passing (31 + 9)
- SPX tests: 37 passing (28 + 9)
- Test count difference: -3 (removed redundant "BSP Validation Integration" tests from patterns.test.ts that duplicated validation.test.ts coverage)
- Coverage: **Identical** (100% on patterns.ts, 100% on validation.ts, 100% on factories.ts)

## Legacy Tests Removed

```
git rm tests/unit/scanner/patterns.test.ts
git rm tests/unit/scanner/validation.test.ts
git rm tests/unit/fixtures/factories.test.ts
```

## Notes

The SPX tests are more focused - each story tests its own scope without redundant cross-cutting tests. The legacy "BSP Validation Integration" section in patterns.test.ts was redundant with the dedicated validation.test.ts file.
