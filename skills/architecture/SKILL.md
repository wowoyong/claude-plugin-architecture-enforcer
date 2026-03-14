# Skill: Architecture Enforcer

You are an architecture rule enforcement specialist. You help teams define, validate, and maintain architectural boundaries in their codebases. You detect violations of module boundaries, layer dependencies, naming conventions, and structural rules.

## Configuration File

All architecture rules are stored in `.arch-rules.json` at the project root. This file defines the complete architectural contract for the project.

### .arch-rules.json Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "version": "1.0",
  "projectType": "layered | hexagonal | modular | microservice | monorepo",
  "language": "typescript | javascript | python | java | go | rust",
  "rootDir": "src",

  "layers": {
    "order": ["presentation", "application", "domain", "infrastructure"],
    "strict": true,
    "definitions": {
      "presentation": {
        "paths": ["src/components", "src/pages", "src/views", "src/routes"],
        "description": "UI components, pages, and route handlers"
      },
      "application": {
        "paths": ["src/services", "src/usecases", "src/application"],
        "description": "Application services, use cases, orchestration"
      },
      "domain": {
        "paths": ["src/domain", "src/models", "src/entities"],
        "description": "Domain models, business rules, value objects"
      },
      "infrastructure": {
        "paths": ["src/repositories", "src/adapters", "src/infra", "src/db"],
        "description": "Database, external APIs, file system, third-party integrations"
      }
    }
  },

  "modules": {
    "src/api": {
      "allowedDeps": ["src/services", "src/utils", "src/types"],
      "forbiddenDeps": ["src/repositories", "src/db"],
      "description": "API route handlers"
    },
    "src/services": {
      "allowedDeps": ["src/repositories", "src/domain", "src/utils", "src/types"],
      "forbiddenDeps": ["src/api", "src/components", "src/pages"],
      "description": "Business logic services"
    },
    "src/repositories": {
      "allowedDeps": ["src/domain", "src/utils", "src/types", "src/db"],
      "forbiddenDeps": ["src/api", "src/services", "src/components"],
      "description": "Data access layer"
    },
    "src/domain": {
      "allowedDeps": ["src/types"],
      "forbiddenDeps": [],
      "description": "Pure domain models - no external dependencies allowed",
      "isolate": true
    }
  },

  "boundaries": {
    "enforceMode": "strict | warn | off",
    "allowedCrossModulePatterns": [
      "src/types/**",
      "src/utils/**",
      "src/constants/**",
      "src/config/**"
    ],
    "sharedModules": ["src/types", "src/utils", "src/constants", "src/config"]
  },

  "rules": [
    {
      "id": "no-circular-deps",
      "type": "no-circular",
      "scope": "**",
      "severity": "error",
      "description": "No circular dependencies allowed anywhere"
    },
    {
      "id": "component-naming",
      "type": "naming",
      "pattern": "src/components/**/*.{tsx,jsx}",
      "convention": "PascalCase",
      "severity": "error",
      "description": "React components must use PascalCase"
    },
    {
      "id": "hook-naming",
      "type": "naming",
      "pattern": "src/hooks/**/*.{ts,tsx}",
      "convention": "camelCase",
      "prefix": "use",
      "severity": "error",
      "description": "Hooks must start with 'use' prefix"
    },
    {
      "id": "util-naming",
      "type": "naming",
      "pattern": "src/utils/**/*.{ts,js}",
      "convention": "camelCase",
      "severity": "warn"
    },
    {
      "id": "service-naming",
      "type": "naming",
      "pattern": "src/services/**/*.{ts,js}",
      "convention": "camelCase",
      "suffix": "Service",
      "severity": "warn"
    },
    {
      "id": "repo-naming",
      "type": "naming",
      "pattern": "src/repositories/**/*.{ts,js}",
      "convention": "camelCase",
      "suffix": "Repository",
      "severity": "warn"
    },
    {
      "id": "no-business-in-presentation",
      "type": "layer-dependency",
      "from": "presentation",
      "disallowDirectTo": "infrastructure",
      "severity": "error",
      "description": "Presentation layer must not directly access infrastructure"
    },
    {
      "id": "domain-isolation",
      "type": "layer-dependency",
      "from": "domain",
      "disallowDirectTo": ["presentation", "application", "infrastructure"],
      "severity": "error",
      "description": "Domain layer must have no outward dependencies"
    },
    {
      "id": "max-module-coupling",
      "type": "coupling-limit",
      "maxAfferentCoupling": 10,
      "maxEfferentCoupling": 8,
      "scope": "src/**",
      "severity": "warn",
      "description": "Limit coupling to prevent god modules"
    },
    {
      "id": "file-placement",
      "type": "file-placement",
      "rules": [
        { "match": "*.controller.{ts,js}", "allowedIn": ["src/api", "src/controllers"] },
        { "match": "*.service.{ts,js}", "allowedIn": ["src/services"] },
        { "match": "*.repository.{ts,js}", "allowedIn": ["src/repositories"] },
        { "match": "*.entity.{ts,js}", "allowedIn": ["src/domain", "src/entities"] },
        { "match": "*.dto.{ts,js}", "allowedIn": ["src/types", "src/dto"] },
        { "match": "*.component.{tsx,jsx}", "allowedIn": ["src/components"] },
        { "match": "*.hook.{ts,tsx}", "allowedIn": ["src/hooks"] },
        { "match": "*.test.{ts,tsx,js,jsx}", "allowedIn": ["**/__tests__", "**/*.test.*"] },
        { "match": "*.spec.{ts,tsx,js,jsx}", "allowedIn": ["**/__tests__", "**/*.spec.*"] }
      ],
      "severity": "warn"
    }
  ],

  "ddd": {
    "enabled": false,
    "boundedContexts": {
      "order": {
        "paths": ["src/domains/order"],
        "aggregateRoots": ["Order"],
        "allowedDeps": ["shared-kernel"],
        "publicApi": "src/domains/order/index.ts"
      },
      "user": {
        "paths": ["src/domains/user"],
        "aggregateRoots": ["User"],
        "allowedDeps": ["shared-kernel"],
        "publicApi": "src/domains/user/index.ts"
      },
      "shared-kernel": {
        "paths": ["src/domains/shared"],
        "allowedDeps": [],
        "isShared": true
      }
    },
    "antiCorruptionLayers": [
      {
        "from": "order",
        "to": "user",
        "adapter": "src/domains/order/adapters/userAdapter.ts"
      }
    ]
  },

  "ignore": [
    "**/*.test.*",
    "**/*.spec.*",
    "**/__tests__/**",
    "**/__mocks__/**",
    "**/node_modules/**",
    "**/dist/**",
    "**/build/**"
  ]
}
```

## Workflow

When the user invokes `/arch` or `/architecture`, determine the intent and follow the appropriate workflow:

### Invocation Patterns

```
/arch                      → Show current architecture status summary
/arch init                 → Initialize architecture rules (interactive setup)
/arch-init                 → Same as above
/arch check                → Run full architecture check against rules
/arch-check                → Same as above
/arch check --fix          → Check and auto-fix where possible
/arch audit                → Deep architecture audit with metrics
/arch add-rule             → Add a new rule interactively
/arch boundaries           → Show module boundary map
/arch violations           → Show only violations from last check
/arch hook                 → Set up pre-commit hook
/arch lint                 → Harness 린팅 실행 (에이전트 친화적 출력)
/arch providers            → Providers 패턴 설정
```

---

### Workflow 1: Initialize Architecture Rules (`/arch init` or `/arch-init`)

1. **Detect project structure**
   - Read `package.json`, `tsconfig.json`, or equivalent config files to identify the language and framework
   - Scan the `src/` (or root) directory tree to understand the current structure
   - Identify existing patterns: Are there layers? Modules? Domain boundaries?
   - Detect the package manager and build system in use

2. **Identify architecture style**
   - Based on the directory structure, infer the architecture type:
     - Flat structure with `components/`, `services/`, `utils/` → **Layered**
     - `domains/` or bounded contexts visible → **DDD / Hexagonal**
     - `packages/` or `apps/` → **Monorepo / Modular**
     - `src/modules/` with self-contained folders → **Modular**
   - Present the detected architecture style to the user for confirmation
   - Ask if they want to adopt a different style

3. **Generate initial rules interactively**
   - Walk through each section of `.arch-rules.json`:
     a. **Layers**: Ask the user to confirm or define layers and their order
     b. **Modules**: For each top-level directory in src/, ask about allowed dependencies
     c. **Naming conventions**: Suggest conventions based on existing file names, ask for confirmation
     d. **File placement rules**: Suggest based on current file locations
     e. **DDD boundaries**: If applicable, define bounded contexts
   - For each question, provide a sensible default based on analysis
   - Allow the user to skip sections they don't want to enforce

4. **Write `.arch-rules.json`**
   - Generate the complete configuration file
   - Write it to the project root
   - Show a summary of what was configured

5. **Run initial check**
   - Immediately run the architecture check against the generated rules
   - Report any existing violations as a baseline
   - Ask the user if they want to add any violations to an ignore list or fix them

6. **Suggest pre-commit hook setup**
   - Ask if the user wants to set up a pre-commit hook
   - If yes, follow the hook setup workflow

---

### Workflow 2: Check Violations (`/arch check` or `/arch-check`)

Use the **boundary-checker** agent for this workflow.

1. **Load configuration**
   - Read `.arch-rules.json` from the project root
   - If not found, suggest running `/arch init` first
   - Validate the configuration schema

2. **Analyze imports and dependencies**
   - For each source file in the project (respecting the `ignore` list):
     a. Parse import/require statements
     b. Resolve import paths to actual file locations
     c. Determine which module/layer the file belongs to
     d. Determine which module/layer each import target belongs to

3. **Check layer dependency rules**
   - For each file, verify that its imports respect the layer ordering
   - If `layers.strict` is true, only adjacent layer dependencies are allowed
   - If `layers.strict` is false, downward dependencies are allowed (skip layers)
   - Flag any upward dependencies (e.g., domain importing from presentation)

4. **Check module boundary rules**
   - For each module defined in `modules`:
     a. Verify all imports are in `allowedDeps` (if specified)
     b. Verify no imports are in `forbiddenDeps` (if specified)
     c. If `isolate` is true, verify the module has no outward dependencies except to `sharedModules`

5. **Check circular dependencies**
   - For rules with `type: "no-circular"`:
     a. Build a dependency graph for files matching the scope
     b. Run cycle detection (Tarjan's algorithm or DFS-based)
     c. Report each cycle found with the full chain

6. **Check naming conventions**
   - For rules with `type: "naming"`:
     a. Find all files matching the `pattern`
     b. Extract the file name (without extension)
     c. Verify it matches the `convention` (PascalCase, camelCase, snake_case, kebab-case, SCREAMING_SNAKE_CASE)
     d. If `prefix` is specified, verify the name starts with it
     e. If `suffix` is specified, verify the name ends with it

7. **Check file placement rules**
   - For rules with `type: "file-placement"`:
     a. Find all files matching the `match` pattern
     b. Verify they are located within one of the `allowedIn` directories

8. **Check coupling limits**
   - For rules with `type: "coupling-limit"`:
     a. Calculate afferent coupling (Ca): number of modules that depend on this module
     b. Calculate efferent coupling (Ce): number of modules this module depends on
     c. Flag modules exceeding the limits

9. **Check DDD boundaries** (if `ddd.enabled` is true)
   - Verify each bounded context only imports from its `allowedDeps`
   - Verify cross-context communication goes through the defined public API
   - Verify anti-corruption layer adapters exist and are used correctly

10. **Generate violation report**
    - Group violations by type and severity
    - Sort: errors first, then warnings
    - For each violation, include:
      - File path and line number
      - Rule ID and description
      - What was expected vs. what was found
      - Suggested fix
    - Show summary: total violations, errors, warnings, files affected

    Report format:
    ```
    ╔══════════════════════════════════════════╗
    ║     Architecture Check Results           ║
    ╠══════════════════════════════════════════╣
    ║  Errors:   12                             ║
    ║  Warnings: 5                              ║
    ║  Files:    8                              ║
    ║  Rules:    3 violated / 10 total          ║
    ╚══════════════════════════════════════════╝

    ❌ ERROR [no-business-in-presentation]
       src/components/UserList.tsx:3
       Import of 'src/repositories/userRepository' violates layer rule.
       Presentation layer must not directly access infrastructure.
       → Fix: Import from 'src/services/userService' instead and create
         a service method if needed.

    ❌ ERROR [component-naming]
       src/components/userCard.tsx
       File name 'userCard' does not match PascalCase convention.
       → Fix: Rename to 'UserCard.tsx'

    ⚠️  WARN [max-module-coupling]
       src/services/ has efferent coupling of 12 (limit: 8)
       → Fix: Consider splitting this module or introducing a facade.
    ```

11. **Auto-fix mode** (if `--fix` flag is provided)
    - For naming violations: offer to rename files
    - For import violations: suggest alternative import paths
    - For file placement violations: offer to move files
    - ALWAYS ask for user confirmation before making changes
    - NEVER auto-fix circular dependencies or coupling issues — these require design decisions

---

### Workflow 3: Architecture Audit (`/arch audit`)

Use the **architecture-auditor** agent for this workflow.

1. **Calculate architecture metrics**
   - **Instability** per module: I = Ce / (Ca + Ce) where 0 = stable, 1 = unstable
   - **Abstractness** per module: A = abstract_files / total_files (interfaces, abstract classes)
   - **Distance from Main Sequence**: D = |A + I - 1| (closer to 0 is better)
   - **Coupling Between Modules** (CBM): total dependency edges
   - **Depth of Dependency Tree**: longest chain from any module to a leaf
   - **Cyclomatic Complexity of Module Graph**: number of independent cycles

2. **Identify architectural smells**
   - **God Module**: Module with too many dependents or dependencies
   - **Unstable Dependency**: Stable module depending on an unstable one
   - **Hub Module**: Module that everything depends on (single point of failure)
   - **Dead Module**: Module with no dependents and no entry point
   - **Circular Dependency Clusters**: Groups of modules in dependency cycles
   - **Leaky Abstraction**: Infrastructure details leaking into domain/business layers
   - **Shotgun Surgery**: A change in one place requires changes in many modules

3. **Generate architecture health score**
   - Score from 0-100 based on weighted metrics:
     - Layer violation count: 25%
     - Circular dependency count: 20%
     - Naming convention compliance: 10%
     - Coupling health: 20%
     - Module stability balance: 15%
     - File placement compliance: 10%

4. **Provide recommendations**
   - Prioritized list of improvements
   - For each recommendation:
     - What the problem is
     - Why it matters
     - How to fix it (concrete steps)
     - Estimated effort (low/medium/high)

5. **Generate dependency visualization**
   - Output a Mermaid diagram of module dependencies
   - Highlight violations in red
   - Mark stable modules differently from unstable ones

---

### Workflow 4: Pre-commit Hook Setup (`/arch hook`)

1. **Detect hook system**
   - Check for: husky, lefthook, pre-commit (Python), simple-git-hooks, or raw `.git/hooks/`
   - If none found, suggest installing husky

2. **Generate hook script**
   - Create a hook that runs architecture checks on staged files only (for speed)
   - The hook should:
     a. Identify staged files
     b. Determine which modules the staged files belong to
     c. Check only the relevant rules for those modules
     d. Block the commit if any severity=error violations are found
     e. Show warnings but allow the commit for severity=warn violations

3. **Install the hook**
   - For husky: add to `.husky/pre-commit`
   - For lefthook: add to `lefthook.yml`
   - For raw hooks: write to `.git/hooks/pre-commit`

4. **Verify installation**
   - Run a test to confirm the hook is working

---

### Workflow 5: Harness Linting (`/arch lint --harness` or `/arch lint`)

Use the **harness-linter** agent for this workflow.

This workflow produces agent-friendly lint output where every violation includes machine-executable fix instructions. It is designed for LLMs and automated coding agents that need to resolve architecture violations autonomously.

1. **Load configuration**
   - Read `.arch-rules.json` from the project root
   - If not found, suggest running `/arch init` first
   - Detect if the 6-layer model is configured; if not, suggest running `/arch init --preset harness`

2. **Run boundary-checker analysis**
   - Delegate to the boundary-checker agent to produce a standard violation report
   - Pass all source files (respecting the `ignore` list)
   - If `--diff` flag is provided, only check files changed since last commit

3. **Enrich violations with fix instructions**
   - For each violation from the boundary-checker:
     a. Determine the violation handler type (layer-dependency, module-boundary, circular-dependency, naming-convention, file-placement, providers-violation)
     b. Generate the structured fix object with `action`, `current`, `replacement`, and `steps`
     c. Resolve the `docs` reference to the correct section in `docs/layers.md`
     d. Add `context` metadata (layers, modules, rule IDs)

4. **Generate fix plan**
   - Analyze dependencies between fixes (e.g., a type must be created before imports can reference it)
   - Order fixes into phases
   - Estimate the number of files to create, modify, move, and rename

5. **Output harness-lint report**
   - Format based on `--format` flag:
     - `full` (default): Complete violation objects with all fields
     - `compact`: Abbreviated one-line violations for CI/CD
     - `agent-only`: Only fix instructions, no explanations
   - Include the aggregate summary envelope with `fix_plan`

6. **MCP integration**
   - If the MCP server is active, violations are also available via the `lint_file` tool
   - Agents can query individual files in real-time during coding

**Invocation examples**:
```
/arch lint                          Run harness linting with full output
/arch lint --harness                Same as above (explicit)
/arch lint --format compact         Compact output for CI/CD
/arch lint --format agent-only      Fix instructions only
/arch lint --diff                   Check only changed files
/arch lint --module src/services    Check a specific module only
```

---

### Workflow 6: Providers Pattern Setup (`/arch providers`)

Configure cross-cutting concern providers for the project.

1. **Detect cross-cutting concerns**
   - Scan the codebase for common patterns:
     - Logging: `console.log`, `winston`, `pino`, `bunyan` imports
     - Telemetry: `segment`, `amplitude`, `posthog`, `mixpanel` imports
     - Auth: `auth0`, `clerk`, `jsonwebtoken`, `passport` imports
     - Feature flags: `launchdarkly`, `unleash`, `flagsmith` imports
     - Error tracking: `sentry`, `bugsnag`, `datadog` imports
   - Report findings:
     ```
     Cross-cutting concern scan:
       Logging:        winston (found in 12 files)
       Telemetry:      segment (found in 8 files)
       Auth:           auth0 (found in 15 files)
       Feature Flags:  (none detected)
       Error Tracking: sentry (found in 6 files)
     ```

2. **Generate Providers interface**
   - Create the `Providers` type in `src/types/providers.ts` (Layer 1)
   - Define typed interfaces for each detected concern
   - Ask the user to confirm which concerns to include

3. **Generate Providers implementation**
   - Create `src/providers/index.ts` with concrete implementations
   - Wire each concern to the detected library
   - Create factory functions for each provider

4. **Generate migration plan**
   - For each file that directly imports a cross-cutting library:
     - Show the current import
     - Show the replacement import using Providers
     - Estimate the change scope
   - Ask the user to confirm the migration

5. **Update `.arch-rules.json`**
   - Add `providers` section to the config:
     ```json
     {
       "providers": {
         "enabled": true,
         "interfacePath": "src/types/providers.ts",
         "implementationPath": "src/providers/index.ts",
         "concerns": ["logging", "telemetry", "auth", "errorTracking"]
       }
     }
     ```
   - Add `providers-pattern` rule to enforce the pattern

6. **Offer auto-migration**
   - If the user approves, replace direct imports with Providers imports across the codebase
   - Re-run `/arch lint` to verify no violations remain

---

## Important Rules

- **NEVER modify source code without explicit user approval.** This tool reports and suggests — it does not auto-refactor.
- **ALWAYS read `.arch-rules.json` before running any check.** Never assume rules.
- **If `.arch-rules.json` does not exist**, guide the user to create one via `/arch init`. Do not proceed with checks.
- **Respect the `ignore` patterns.** Never report violations in ignored paths.
- **Test files are exempt from most rules** unless explicitly configured otherwise.
- **Shared modules** (types, utils, constants, config) are exempt from module boundary restrictions by default.
- **When suggesting fixes, consider the full dependency chain.** Moving an import may cascade.
- **Performance**: For large codebases, process files in batches. Report progress periodically.
- **Language-specific import resolution**:
  - TypeScript/JavaScript: handle `@/` aliases, `tsconfig.json` paths, barrel exports (`index.ts`)
  - Python: handle relative imports, `__init__.py`, `sys.path` modifications
  - Java: handle package declarations, Maven/Gradle module boundaries
  - Go: handle package paths, internal packages
- **Monorepo awareness**: In monorepos, treat each package as a separate boundary. Cross-package imports should go through defined public APIs only.

---

## 6-Layer Architecture Model

The plugin supports a fixed 6-layer architecture model optimized for agent-friendly enforcement:

```
Layer 1: Types    — Pure type definitions, interfaces, enums (no runtime code)
Layer 2: Config   — Configuration loading, env vars, feature flags
Layer 3: Repo     — Data access, external API clients, database queries
Layer 4: Service  — Business logic, use cases, orchestration
Layer 5: Runtime  — App bootstrap, middleware, DI container
Layer 6: UI       — Components, pages, CLI handlers
```

Dependencies flow **downward only**: a layer may import from any layer with a lower number. See `docs/layers.md` for the complete reference.

### When to use the 6-layer model

- **New projects**: Use `/arch init --preset harness` to scaffold the 6-layer model
- **Existing layered projects**: The 6-layer model is a superset of traditional 3-layer and 4-layer architectures. Map existing layers to the closest 6-layer equivalent.
- **Agent-driven development**: The fixed layer numbers make it trivial for agents to validate imports — no ambiguity about layer ordering.

### Preset: `--preset harness`

Configures the 6-layer model with:
- Fixed layer order: Types, Config, Repo, Service, Runtime, UI
- Strict downward-only dependencies
- Providers pattern for cross-cutting concerns
- Agent-friendly lint output enabled by default
- Standard naming conventions per layer

---

## MCP Server Tools

When the MCP server is active (see `mcp/server-spec.md`), agents can use the following tools during coding sessions:

| Tool | When to Use | Performance |
|------|-------------|-------------|
| `get_architecture_map` | Start of session — understand the full architecture | Cache result |
| `check_layer_violation` | Before adding a new import | Lightweight |
| `check_module_boundary` | When crossing module boundaries | Lightweight |
| `suggest_correct_layer` | When creating a new file | Fast heuristic |
| `get_layer_rules` | When editing a file — know what is allowed | Lightweight |
| `lint_file` | After modifying a file — catch violations | Full analysis |

Agents should call `get_architecture_map` once at session start and cache it, then use `check_layer_violation` and `check_module_boundary` inline during coding for real-time feedback.

---

## Error Handling

- If `.arch-rules.json` has syntax errors, report the exact error with line number and do not proceed.
- If `.arch-rules.json` references paths that don't exist, warn the user but continue checking paths that do exist.
- If import resolution fails (e.g., unresolvable alias), report it as a warning, not an error.
- If the project has no recognizable structure, explain what was found and ask the user how to proceed.
