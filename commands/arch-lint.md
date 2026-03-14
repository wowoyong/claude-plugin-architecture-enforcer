# Command: arch-lint

Run agent-friendly architecture linting with structured, actionable output. Every violation includes machine-executable fix instructions.

## Usage

```
/arch-lint                                   Run harness linting with full output
/arch-lint --module <path>                   Lint a specific module only
/arch-lint --format full                     Complete violation objects (default)
/arch-lint --format compact                  Abbreviated one-line violations for CI/CD
/arch-lint --format agent-only               Fix instructions only, no explanations
/arch-lint --fix                             Auto-fix naming and file placement violations
/arch-lint --diff                            Check only files changed since last commit
```

## Description

에이전트 친화적 아키텍처 린팅. 위반 사항에 수정 방법을 포함한 구조화된 출력 제공.

This command delegates to the **harness-linter** agent, which wraps the boundary-checker analysis with enriched fix instructions. The output is designed for LLMs and automated coding agents that need to resolve architecture violations autonomously.

## Execution Steps

### Step 1: Load Configuration

- Read `.arch-rules.json` from the project root
- If not found, suggest running `/arch-init` first
- Detect if the 6-layer model is configured; if not, suggest `/arch-init --preset harness`
- Validate the configuration schema before proceeding

### Step 2: Determine Scope

- If `--module <path>` is provided, restrict analysis to files within that module
- If `--diff` is provided, identify files changed since the last commit and restrict to those
- Otherwise, scan all source files (respecting the `ignore` list in `.arch-rules.json`)

### Step 3: Run Analysis

Delegate to the harness-linter agent, which:

1. Invokes the boundary-checker agent for structural analysis
2. Enriches each violation with structured fix objects
3. Resolves documentation references to `docs/layers.md`
4. Generates a dependency-aware fix plan

### Step 4: Output Results

Format the output based on `--format`:

- **full** (default): Complete violation objects with all fields including multi-step fix plans
- **compact**: One-line violations for CI/CD integration
- **agent-only**: Only fix instructions, optimized for direct agent execution

Output follows the harness-linter JSON schema:

```json
{
  "violation": "<violation-type>",
  "severity": "error | warn",
  "file": "<file-path>",
  "line": "<line-number>",
  "what": "<description>",
  "why": "<architectural rule explanation>",
  "fix": {
    "action": "<fix-action-type>",
    "current": "<current code or state>",
    "replacement": "<corrected code or state>",
    "steps": ["<step-by-step instructions>"]
  },
  "docs": "<docs reference>",
  "context": {
    "fromLayer": "<source layer>",
    "toLayer": "<target layer>",
    "ruleId": "<rule identifier>"
  }
}
```

### Step 5: Auto-Fix (if `--fix` flag is provided)

For auto-fixable violations only:

- **Naming convention violations**: Rename files to match the required convention (PascalCase, camelCase, etc.) and update all import references
- **File placement violations**: Move files to the correct directory and update all import references

Non-auto-fixable violations (layer dependencies, circular dependencies, module boundaries) are reported but NOT auto-fixed, as they require design decisions.

Always ask for user confirmation before executing fixes.

### Step 6: Summary

Display an aggregate summary:

```
Harness Lint Report
═══════════════════
Files checked:  142
Rules checked:  12
Violations:     7 (4 errors, 3 warnings)
Auto-fixable:   5

Fix Plan:
  Phase 1: Create missing type definitions (2 violations)
  Phase 2: Extract shared utilities (1 violation)
  Phase 3: Rename and move files (2 violations)
```

## Options

| Option | Description |
|--------|-------------|
| `--module <path>` | Restrict linting to a specific module directory |
| `--format full` | Complete violation output with fix plans (default) |
| `--format compact` | One-line violations for CI/CD |
| `--format agent-only` | Fix instructions only |
| `--fix` | Auto-fix naming and file placement violations |
| `--diff` | Only check files changed since last commit |

## Related

- See `agents/harness-linter.md` for the full violation handler specification
- See `docs/layers.md` for the 6-layer architecture model reference
- See `mcp/server-spec.md` for real-time linting via MCP `lint_file` tool
