# Command: arch-check

Run architecture rule checks against the current codebase.

## Usage

```
/arch-check                    Run full architecture check
/arch-check --fix              Check and offer auto-fixes for violations
/arch-check --module <path>    Check only a specific module
/arch-check --rule <id>        Check only a specific rule
/arch-check --severity error   Show only errors (skip warnings)
/arch-check --json             Output results as JSON
/arch-check --diff             Check only files changed since last commit
/arch-check --baseline         Save current violations as baseline (for legacy projects)
/arch-check --compare          Compare current violations against saved baseline
```

## Execution Steps

1. **Validate prerequisites**
   - Verify `.arch-rules.json` exists in the project root
   - If not found, output: "No architecture rules found. Run `/arch-init` to set up rules."
   - Validate the JSON syntax and schema of `.arch-rules.json`

2. **Load and parse rules**
   - Read `.arch-rules.json`
   - Resolve all path patterns relative to the project root
   - Identify which rules are active (not disabled)
   - If `--rule` flag is provided, filter to only that rule
   - If `--module` flag is provided, filter to only rules affecting that module

3. **Discover source files**
   - Scan the `rootDir` (default: `src/`) for source files
   - Apply the `ignore` patterns to exclude files
   - If `--diff` flag is provided, only include files changed since the last commit (use `git diff --name-only HEAD`)
   - Classify each file into its module and layer

4. **Delegate to boundary-checker agent**
   - Pass the loaded rules and discovered files to the boundary-checker agent
   - The agent performs all violation checks (see SKILL.md Workflow 2)

5. **Process results**
   - Collect all violations from the agent
   - If `--severity` flag is provided, filter by severity
   - Sort violations: errors first, then warnings, grouped by file
   - If `--json` flag is provided, format as JSON instead of human-readable output

6. **Display report**
   - Show the summary header with counts
   - Show each violation with file path, line number, rule ID, description, and suggested fix
   - At the end, show actionable next steps:
     - "Run `/arch-check --fix` to auto-fix N fixable violations"
     - "Run `/arch audit` for a deep architecture health analysis"

7. **Handle `--fix` mode**
   - For each fixable violation, show the proposed change and ask for confirmation
   - Apply approved fixes one at a time
   - After all fixes, re-run the check to verify the fixes resolved the violations

8. **Handle `--baseline` mode**
   - Save the current violation count and details to `.arch-baseline.json`
   - This is useful for legacy projects where you want to prevent NEW violations without fixing all existing ones
   - Future `--compare` runs will only flag violations that are NOT in the baseline

9. **Exit code behavior** (for CI/hook integration)
   - Exit 0: No errors (warnings are OK)
   - Exit 1: One or more error-severity violations found
   - Exit 2: Configuration error (invalid `.arch-rules.json`)

## Output Format

### Human-readable (default)

```
Architecture Check - project-name
══════════════════════════════════

Checking 142 files against 10 rules...

[Layer Dependencies]
  ❌ ERROR src/components/Dashboard.tsx:5
     Imports 'src/repositories/analyticsRepo' — presentation cannot access infrastructure.
     Rule: no-business-in-presentation
     Fix: Use 'src/services/analyticsService' as an intermediary.

  ❌ ERROR src/pages/Settings.tsx:12
     Imports 'src/db/connection' — presentation cannot access infrastructure.
     Rule: no-business-in-presentation
     Fix: Move database logic to a repository and expose via a service.

[Module Boundaries]
  ❌ ERROR src/services/orderService.ts:8
     Imports 'src/api/routes' — 'src/services' cannot depend on 'src/api'.
     Rule: module-boundary (src/services)
     Fix: Extract shared types to 'src/types' or invert the dependency.

[Naming Conventions]
  ⚠️  WARN src/components/userAvatar.tsx
     File name 'userAvatar' does not match PascalCase.
     Rule: component-naming
     Fix: Rename to 'UserAvatar.tsx' and update all imports.

[Circular Dependencies]
  ❌ ERROR Cycle detected:
     src/services/authService.ts → src/services/userService.ts → src/services/authService.ts
     Rule: no-circular-deps
     Fix: Extract shared logic into a new module or use dependency injection.

══════════════════════════════════
Summary: 4 errors, 1 warning across 4 files
  3 rules violated out of 10 total
  2 violations are auto-fixable

Run `/arch-check --fix` to fix naming violations.
Run `/arch audit` for detailed architecture health analysis.
```

### JSON format (`--json`)

```json
{
  "timestamp": "2026-03-13T10:00:00Z",
  "project": "project-name",
  "filesChecked": 142,
  "rulesChecked": 10,
  "summary": {
    "errors": 4,
    "warnings": 1,
    "filesAffected": 4,
    "rulesViolated": 3
  },
  "violations": [
    {
      "id": "v001",
      "ruleId": "no-business-in-presentation",
      "severity": "error",
      "type": "layer-dependency",
      "file": "src/components/Dashboard.tsx",
      "line": 5,
      "message": "Imports 'src/repositories/analyticsRepo' — presentation cannot access infrastructure.",
      "fix": {
        "description": "Use 'src/services/analyticsService' as an intermediary.",
        "autoFixable": false
      }
    }
  ]
}
```
