# SPX Migration Log: 43-status-determination.feature

## Migration Date

2026-01-29

## Migration Status: INCOMPLETE

**Cannot remove legacy tests - coverage would drop from 86.3% to 39.72%**

## Tests Comparison

| Legacy Location                                      | Tests | SPX Location                                      | Tests | Gap           |
| ---------------------------------------------------- | ----- | ------------------------------------------------- | ----- | ------------- |
| `tests/unit/status/state.test.ts`                    | 5     | `21-state-machine.story/tests/state.unit.test.ts` | 5     | None          |
| `tests/integration/status/state.integration.test.ts` | 19    | Split across stories 32, 43, 54                   | 11    | **8 missing** |

### Missing Tests in SPX

The following tests from legacy are NOT present in SPX:

1. `getWorkItemStatus` - 7 tests:
   - "GIVEN work item with no tests dir WHEN getting status THEN returns OPEN"
   - "GIVEN work item with tests but no DONE.md WHEN getting status THEN returns IN_PROGRESS"
   - "GIVEN work item with DONE.md WHEN getting status THEN returns DONE"
   - "GIVEN work item with empty tests dir WHEN getting status THEN returns OPEN"
   - "GIVEN work item with only DONE.md WHEN getting status THEN returns DONE"
   - "GIVEN work item with DONE.md as directory WHEN getting status THEN returns IN_PROGRESS"
   - "GIVEN non-existent work item WHEN getting status THEN throws StatusDeterminationError"

2. Performance test - 1 test:
   - "GIVEN work item WHEN getting status multiple times THEN completes quickly"

## Action Required

Before legacy tests can be removed:

1. Add missing tests to `54-status-edge-cases.story/tests/`
2. Verify coverage matches legacy (86.3% on state.ts)

## Legacy Tests NOT Removed

```
# DO NOT REMOVE - incomplete migration
tests/unit/status/state.test.ts
tests/integration/status/state.integration.test.ts
```
