# Command: arch-init

Initialize architecture rules for a project through interactive setup.

## Usage

```
/arch-init                     Interactive setup with auto-detection
/arch-init --preset layered    Use a preset architecture template
/arch-init --preset hexagonal  Use hexagonal architecture template
/arch-init --preset ddd        Use DDD template
/arch-init --preset modular    Use modular architecture template
/arch-init --minimal           Create minimal rules (layers + no-circular only)
/arch-init --from-structure    Infer all rules from existing code structure
```

## Execution Steps

### Step 1: Project Detection

Scan the project to understand its current state:

- Read `package.json` (or equivalent) to identify the language, framework, and project name
- Read `tsconfig.json` to understand path aliases (e.g., `@/` → `src/`)
- Scan the directory tree of the source root (typically `src/`)
- List all top-level directories and their file counts
- Identify existing patterns:
  - Are files grouped by feature or by type?
  - Do directory names suggest layers (components, services, repositories)?
  - Are there barrel exports (index.ts files)?
  - Is there a `domains/` or `modules/` directory suggesting DDD or modular architecture?

Present findings:
```
Project Analysis
════════════════
Name:       my-app
Language:   TypeScript
Framework:  Next.js 15
Structure:  src/ based

Detected directories:
  src/app/          (14 files) — Next.js App Router pages
  src/components/   (32 files) — React components
  src/hooks/        (8 files)  — Custom hooks
  src/services/     (6 files)  — Business logic
  src/repositories/ (4 files)  — Data access
  src/types/        (5 files)  — Type definitions
  src/utils/        (12 files) — Utility functions
  src/lib/          (3 files)  — Library wrappers

Suggested architecture: Layered Architecture
```

### Step 2: Choose Architecture Style

Ask the user to confirm or select:

```
Which architecture style best describes your project?

1. Layered Architecture (presentation → business → data)
   Best for: Traditional web apps, APIs

2. Hexagonal Architecture (ports & adapters)
   Best for: Domain-heavy apps, clean architecture

3. Modular Architecture (self-contained modules)
   Best for: Large apps with independent features

4. DDD with Bounded Contexts
   Best for: Complex domains with multiple subdomains

5. Custom (define your own structure)

Recommended based on your structure: [1]
```

### Step 3: Define Layers

Based on the chosen style, configure layers:

```
Layer Configuration
═══════════════════
Define the layers in your architecture (top to bottom).
Higher layers can depend on lower layers, but not vice versa.

Detected layers from your structure:
  1. presentation  →  src/app/, src/components/
  2. application   →  src/hooks/, src/services/
  3. data          →  src/repositories/

Shared (accessible from all layers):
  - src/types/
  - src/utils/
  - src/lib/

Accept this configuration? (Y/n/customize)
```

If the user chooses to customize:
- Allow adding/removing/reordering layers
- Allow mapping directories to layers
- Allow choosing strict mode (only adjacent layer deps) vs. relaxed (any downward dep)

### Step 4: Define Module Boundaries

For each top-level directory, define allowed dependencies:

```
Module Boundary Configuration
═════════════════════════════

For each module, specify which other modules it may import from.

src/app/ (presentation layer)
  Suggested allowed deps: [src/components, src/hooks, src/services, src/types, src/utils]
  Accept? (Y/n/customize)

src/services/ (application layer)
  Suggested allowed deps: [src/repositories, src/types, src/utils, src/lib]
  Accept? (Y/n/customize)

src/repositories/ (data layer)
  Suggested allowed deps: [src/types, src/utils, src/lib]
  Accept? (Y/n/customize)
```

The suggestions should be inferred from actual import analysis of the codebase — scan existing imports to see what each module currently depends on, then present that as the baseline.

### Step 5: Define Naming Conventions

Analyze existing file names to detect conventions:

```
Naming Convention Configuration
════════════════════════════════

Detected patterns:
  src/components/ → PascalCase (32/32 files match)  ✓
  src/hooks/      → camelCase with 'use' prefix (8/8 files match)  ✓
  src/services/   → camelCase with 'Service' suffix (4/6 files match)  ⚠️
    Non-matching: helpers.ts, constants.ts
  src/utils/      → camelCase (12/12 files match)  ✓

Enforce these conventions? (Y/n/customize)
```

### Step 6: Define File Placement Rules

```
File Placement Configuration
═════════════════════════════

Define where specific file types should be placed:

  *.controller.ts  →  src/api/ or src/controllers/
  *.service.ts     →  src/services/
  *.repository.ts  →  src/repositories/
  *.component.tsx  →  src/components/
  *.hook.ts        →  src/hooks/
  *.test.ts        →  alongside source file or __tests__/

Accept? (Y/n/customize)
```

### Step 7: Additional Rules

```
Additional Rules
════════════════

Enable the following rules?

  [x] No circular dependencies (recommended)
  [x] Module coupling limits (max 10 afferent, 8 efferent)
  [ ] DDD bounded context enforcement
  [ ] Strict barrel export requirement (all module imports through index.ts)
  [x] No direct database imports outside repositories
  [ ] Custom rules (define later)

Select rules to enable/disable:
```

### Step 8: Generate Configuration

- Assemble all choices into a valid `.arch-rules.json`
- Write the file to the project root
- Display the generated configuration with syntax highlighting

### Step 9: Initial Baseline Check

- Run `/arch-check` immediately with the new rules
- Report any existing violations
- Ask the user:

```
Found 7 existing violations (3 errors, 4 warnings).

Options:
  1. Save as baseline (ignore existing violations, enforce rules for new code only)
  2. Show me the violations so I can fix them now
  3. Adjust rules to accommodate current structure

Choose:
```

### Step 10: Pre-commit Hook Offer

```
Would you like to set up a pre-commit hook to enforce these rules?

This will check architecture rules on every commit and block commits
that introduce new violations.

  1. Yes, set up with husky
  2. Yes, set up with lefthook
  3. Yes, set up with simple-git-hooks
  4. No, I'll run checks manually

Choose:
```

## Presets

### `--preset layered`
Standard 3-layer architecture: presentation → business → data
- Layers: presentation, business, data
- Strict layer dependencies
- Standard naming conventions
- No circular dependencies

### `--preset hexagonal`
Hexagonal / Ports & Adapters architecture:
- Core domain with no external dependencies
- Ports (interfaces) in domain layer
- Adapters in infrastructure layer
- Application services orchestrate use cases
- Layers: adapters → application → domain

### `--preset ddd`
Domain-Driven Design with bounded contexts:
- Enables DDD configuration
- Bounded contexts with public APIs
- Anti-corruption layers between contexts
- Aggregate root enforcement
- Shared kernel for cross-cutting concerns

### `--preset modular`
Feature-based modular architecture:
- Each feature is a self-contained module
- Modules communicate through defined APIs
- Shared module for cross-cutting concerns
- No direct cross-module imports (must go through public API)

### `--preset harness`
Harness Engineering 6계층 아키텍처:
- 레이어: Types(L1) → Config(L2) → Repo(L3) → Service(L4) → Runtime(L5) → UI(L6)
- Providers: 교차 관심사를 단일 인터페이스로 주입
- 에이전트 친화적 린트 메시지 활성화
- 엄격한 단방향 의존성 (하위 레이어만 참조 가능)

## Important Notes

- If `.arch-rules.json` already exists, ask the user before overwriting
- All path patterns use forward slashes, even on Windows
- The generated config should be committed to version control
- Include helpful comments in the output explaining each section
