# Capability: Core CLI

## Success Metric

**Quantitative Target:**

- **Baseline**: Manual status checks take 1-2 minutes with LLM calls (~$0.01-0.05 per check)
- **Target**: Deterministic status analysis in <100ms with zero token cost
- **Measurement**: Performance benchmarks in E2E tests; cost = $0 per invocation

## Testing Strategy

> Use `/testing-typescript` skill to understand testing strategy.

### Level Assignment

| Component            | Level | Justification                                       |
| -------------------- | ----- | --------------------------------------------------- |
| Pattern matching     | 1     | Pure functions, regex operations                    |
| Directory scanning   | 2     | Needs real filesystem operations                    |
| Status determination | 2     | Needs real file structure and DONE.md detection     |
| Tree building        | 1     | Pure data structure assembly                        |
| Output formatting    | 1     | Pure rendering functions                            |
| CLI integration      | 2     | Needs Commander.js framework                        |
| Full workflow        | 3     | Needs complete environment + performance validation |

### Escalation Rationale

- **1 -> 2**: Unit tests prove our parsing and state logic, but Level 2 verifies real filesystem operations work correctly (directory walking, file detection, cross-platform paths)
- **2 -> 3**: Integration tests prove components work together, but Level 3 verifies the complete CLI workflow delivers value within performance targets (<100ms for 50 work items)

## Capability E2E Tests (Level 3)

These tests verify the **complete user journey** delivers value.

### E2E1: Fast Status Analysis

```typescript
// test/e2e/core-cli.e2e.test.ts
describe("Capability: Core CLI", () => {
  it("GIVEN fixture repo with 50 work items WHEN running spx status --json THEN completes in <100ms", async () => {
    // Given: Fixture repo with realistic work item structure
    const fixtureRoot = "test/fixtures/repos/sample-50-items";

    // When: Execute full CLI workflow
    const startTime = Date.now();
    const { stdout, exitCode } = await execa("node", [
      "dist/bin/spx.js",
      "status",
      "--json",
    ], {
      cwd: fixtureRoot,
    });
    const elapsed = Date.now() - startTime;

    // Then: Performance target met and valid output delivered
    expect(exitCode).toBe(0);
    expect(elapsed).toBeLessThan(100);

    const result = JSON.parse(stdout);
    expect(result).toHaveProperty("summary");
    expect(result.summary).toMatchObject({
      done: expect.any(Number),
      inProgress: expect.any(Number),
      open: expect.any(Number),
    });
  });
});
```

### E2E2: Multiple Output Formats

```typescript
describe("Capability: Core CLI - Output Formats", () => {
  it("GIVEN fixture repo WHEN requesting different formats THEN all formats render correctly", async () => {
    const fixtureRoot = "test/fixtures/repos/sample-10-items";

    // JSON format
    const jsonResult = await execa("node", [
      "dist/bin/spx.js",
      "status",
      "--json",
    ], {
      cwd: fixtureRoot,
    });
    expect(() => JSON.parse(jsonResult.stdout)).not.toThrow();

    // Text format (default)
    const textResult = await execa("node", ["dist/bin/spx.js", "status"], {
      cwd: fixtureRoot,
    });
    expect(textResult.stdout).toContain("specs/doing/");

    // Table format
    const tableResult = await execa("node", [
      "dist/bin/spx.js",
      "status",
      "--format",
      "table",
    ], {
      cwd: fixtureRoot,
    });
    expect(tableResult.stdout).toMatch(/\|.*\|/); // Contains table borders
  });
});
```

## System Integration

This capability is foundational - it provides the core functionality that future capabilities will build upon:

- **Capability 30 (MCP Server)**: Will expose these CLI operations as MCP tools
- **Future capabilities**: Will use the scanner and status components as building blocks

## Completion Criteria

- [ ] All Level 1 tests pass (via story completion)
- [ ] All Level 2 tests pass (via feature completion)
- [ ] All Level 3 E2E tests pass
- [ ] Success metric achieved: <100ms for 50 work items
- [ ] All 7 features completed (21, 32, 43, 54, 65, 76, 87)

**Note**: To see current features in this capability, use `ls` or `find` to list feature directories (e.g., `feature-*`) within this capability's folder.

---

## PRD Content (Merged)

> The following content was merged from the original capability-scoped PRD.

### Problem Statement

#### Customer Problem

```
As an AI coding agent user, I am frustrated by slow and expensive spec status analysis
because the current LLM skill uses probabilistic inference for deterministic file operations,
which prevents me from getting instant project status and wastes thousands of tokens per query.
```

#### Current Customer Pain

- **Symptom**: Spec workflow skill takes 1-2 minutes and consumes excessive tokens
- **Root Cause**: Using LLM for deterministic tasks (directory walking, pattern matching, file existence checks)
- **Customer Impact**: Slow feedback loops, context window pollution, delayed decision-making
- **Business Impact**: Unnecessary API costs, reduced productivity, friction in AI-assisted development

### Solution Design

#### Customer Solution

```
Implement a lightweight CLI tool (spx) that performs spec status analysis in <100ms
through native file system operations, resulting in instant project visibility
and clean MCP tool integration for AI agents.
```

#### Customer Journey Context

- **Before**: Invoke LLM skill -> Wait 1-2 min -> Receive status report -> Resume work
- **During**: `spx status --json` provides instant, deterministic results; MCP server exposes same capability to agents
- **After**: Instant status checks, zero token cost for facts, LLM reserved for interpretation and decisions

### Expected Outcome

#### Measurable Outcome

```
Users will receive spec status in <100ms with zero token cost,
leading to faster iteration cycles and reduced API spend,
proven by 1000x speed improvement and 100% token elimination for status queries.
```

#### Evidence of Success (BDD Tests)

- [ ] `Response Time: Current 60-120s -> Target <100ms (1000x improvement)`
- [ ] `Token Cost: Current ~2000 tokens -> Target 0 tokens (100% reduction)`
- [ ] `Accuracy: Current ~99% -> Target 100% (deterministic correctness)`
- [ ] `Integration: MCP tool callable by Claude Code and other agents`

### Scope

#### In Scope (MVP)

- Deterministic scan of a specs tree rooted at `specs/` (or `--root` override)
- Identify work items by directory naming patterns:
  - `capability-NN_slug/`
  - `feature-NN_slug/`
  - `story-NN_slug/`
- Determine status using the "tests directory state" rule:
  - Missing/empty `tests/` -> **OPEN**
  - `tests/` has files but no `DONE.md` -> **IN PROGRESS**
  - `tests/DONE.md` exists -> **DONE**
- Render hierarchical report from capabilities -> features -> stories
- Output formats:
  - `text` (human-friendly tree)
  - `json` (agents/automation)
  - `md` (copy/paste into specs/docs)
  - `table` (compact overview)
- Context-aware discovery: read `specs/templates/structure.yaml` for custom patterns if present
- Stable sorting by `(kindOrder, number, slug)` for deterministic output

#### Explicit Non-Goals (MVP)

- Modifying project structure ("analyze and report", not "fix")
- Editing artifacts (requirements/decisions/work items)
- Deep parsing of markdown contents beyond basic metadata
- Enforcing or generating work items/templates (optional later)

### Key CLI Commands

#### Core Commands

- **`spx status [item] [--format text|json|md|table] [--root <path>]`**
  - Primary command: scan + report
  - `[item]` - Optional work item filter (e.g., `capability-32`)
  - Returns hierarchical status tree

- **`spx next [--scope capability|feature] [--root <path>]`**
  - Return the next OPEN/IN PROGRESS item by ordering (lowest-number first)
  - Respects implicit work item ordering

- **`spx tree [--root <path>]`**
  - Visual tree without status computation (fast navigation)

- **`spx done <item> [--root <path>]`**
  - Create `tests/DONE.md` for specified work item

#### Quality & Initialization

- **`spx validate [--root <path>]`**
  - Validate naming patterns, ordering, required folders/files
  - Exit non-zero on violations (CI/CD integration)

- **`spx init [--root <path>]`**
  - Initialize specs directory structure with conventions

- **`spx explain <path-to-item>`**
  - Explain why an item is OPEN/IN PROGRESS/DONE (rule-based, minimal)

### Data Model

#### Work Item

- `kind`: `"capability" | "feature" | "story"`
- `number`: integer parsed from directory name
- `slug`: full directory name (e.g., `story-32_lighthouse-runner`)
- `path`: absolute or repo-relative path
- `status`: `"OPEN" | "IN_PROGRESS" | "DONE"`
- `children`: nested work items (capability -> features -> stories)

#### JSON Output (Stable Contract)

```typescript
{
  "project": {
    "name"?: string,
    "root": string,
    "specsRoot": string
  },
  "summary": {
    "done": number,
    "inProgress": number,
    "open": number
  },
  "capabilities": WorkItem[]  // capabilities with nested children
}
```

### Architecture

#### Layering (for CLI now, MCP later)

1. **Core library** (`spx-core`)
   - Pure functions for scanning, parsing names, computing statuses, ordering, rendering models
2. **CLI adapter**
   - Argument parsing, formatting, exit codes, printing
3. **(Future) MCP server adapter**
   - Exposes `status/next/validate/explain` as tools using the same core library

#### Proposed Architecture

```
spx/
├── src/
│   ├── cli.ts              # Command definitions (Commander.js)
│   ├── scanner/
│   │   ├── index.ts        # Scanner public API
│   │   ├── walk.ts         # Directory walking
│   │   └── patterns.ts     # Work item pattern matching
│   ├── status/
│   │   ├── index.ts        # Status public API
│   │   ├── state.ts        # DONE/IN PROGRESS/OPEN state machine
│   │   └── context.ts      # Context discovery (specs/templates/structure.yaml)
│   ├── reporter/
│   │   ├── index.ts        # Reporter public API
│   │   ├── text.ts         # Text tree output
│   │   ├── json.ts         # JSON output
│   │   ├── markdown.ts     # Markdown output
│   │   └── table.ts        # Table output
│   └── mcp/
│       └── server.ts       # MCP tool exposure
├── package.json
└── tsconfig.json
```

#### Determinism Rules

- Stable sorting by `(kindOrder, number, slug)`
- No filesystem-order dependence
- Normalize path separators internally for cross-platform compatibility

#### Performance Considerations

- **Target**: <100ms for typical project (~50 work items)
- **Strategy**: Avoid reading file contents unless required (status derivable from presence/absence only)
- **Optional caching** (Future): Cache keyed by directory mtimes and/or git diff to avoid rescanning unchanged subtrees
- **Incremental mode** (Future): Only rescan modified work items since last scan

### Pre-Mortem Analysis

#### Risk: Name collision with Microsoft Azure Speech CLI (spx)

- **Likelihood**: Low - different domain, different audience
- **Impact**: Low - SEO differentiation via `spx.sh`, npm scoping options
- **Mitigation**: Own `spx.sh` domain, consider `@spx/cli` npm scope, clear positioning as spec workflow tool

#### Risk: Spec convention drift across projects

- **Likelihood**: Medium - users may customize directory patterns
- **Impact**: Medium - CLI returns incorrect status for non-standard layouts
- **Mitigation**: Support `spx.config.json` for custom patterns, read `specs/templates/structure.yaml` if present, provide clear error messages

#### Risk: MCP ecosystem immaturity

- **Likelihood**: Medium - MCP is evolving rapidly
- **Impact**: Low - CLI works standalone, MCP is additive
- **Mitigation**: Decouple CLI core from MCP layer, version MCP integration separately

#### Risk: Cross-platform path and glob quirks

- **Likelihood**: Medium
- **Impact**: High - breaks matching on Windows
- **Mitigation**: Use `path` utilities; avoid brittle globbing; add fixtures for Windows-style separators

#### Risk: Large repos make scans slow

- **Likelihood**: Medium
- **Impact**: Medium-High
- **Mitigation**: Avoid file reads; add incremental cache; provide `--scope` to limit traversal

### Deployment Plan

#### Structured around descendant work items

1. **Capability: Core CLI** - Scanner, status logic, reporter with multiple output formats
2. **Feature: Status Command** - `spx status [item] [--format json|tree|table]`
3. **Feature: Navigation Commands** - `spx next`, `spx tree`
4. **Feature: Mutation Commands** - `spx done <item>`, `spx init`
5. **Feature: Validation Command** - `spx validate`, `spx explain`
6. **(Future) Feature: Incremental Performance** - Cache by mtime / git diff
7. **(Future) Capability: MCP Server** - Expose status and next as MCP tools

### Success Criteria

- [ ] `spx status` completes in <100ms for typical project (~50 work items)
- [ ] JSON output parseable by MCP clients without transformation
- [ ] Zero false positives/negatives on status detection
- [ ] Deterministic output across runs (no nondeterministic ordering)
- [ ] Cross-platform compatibility (Windows/macOS/Linux)
- [ ] Commands return sensible exit codes for CI/automation (`0` ok, `1` validation failed, etc.)
- [ ] Published to npm
- [ ] `spx.sh` documentation live at v1.0 release
