# MCP Server Specification: Architecture Enforcer

This document specifies the Model Context Protocol (MCP) server interface for the architecture-enforcer plugin. It allows coding agents to check architecture rules in real-time during coding sessions, preventing violations before they are committed.

## Server Metadata

```json
{
  "name": "architecture-enforcer-mcp",
  "version": "1.0.0",
  "description": "Real-time architecture rule checking for coding agents",
  "protocol": "mcp",
  "capabilities": {
    "tools": true,
    "resources": true
  }
}
```

## Tools

### Tool 1: `check_layer_violation`

Check if a specific import statement violates layer dependency rules. Agents should call this tool **before adding a new import** to a file.

**When to use**: During coding, when an agent is about to write or suggest an import statement. Call this tool first to verify the import is architecturally valid.

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "source_file": {
      "type": "string",
      "description": "Absolute or project-relative path to the file that contains the import"
    },
    "import_path": {
      "type": "string",
      "description": "The import path being added (e.g., '../components/OrderForm' or '@/services/auth')"
    },
    "import_type": {
      "type": "string",
      "enum": ["value", "type"],
      "default": "value",
      "description": "Whether this is a value import or type-only import"
    }
  },
  "required": ["source_file", "import_path"]
}
```

**Output Schema**:
```json
{
  "type": "object",
  "properties": {
    "allowed": {
      "type": "boolean",
      "description": "Whether the import is allowed by layer rules"
    },
    "source_layer": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "level": { "type": "integer", "minimum": 1, "maximum": 6 }
      }
    },
    "target_layer": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "level": { "type": "integer", "minimum": 1, "maximum": 6 }
      }
    },
    "violation": {
      "type": "object",
      "nullable": true,
      "description": "Present only if allowed=false. Contains harness-lint violation object.",
      "properties": {
        "what": { "type": "string" },
        "why": { "type": "string" },
        "fix": { "type": "object" }
      }
    }
  }
}
```

**Example request**:
```json
{
  "tool": "check_layer_violation",
  "arguments": {
    "source_file": "src/services/orderService.ts",
    "import_path": "../components/OrderForm",
    "import_type": "value"
  }
}
```

**Example response (violation)**:
```json
{
  "allowed": false,
  "source_layer": { "name": "service", "level": 4 },
  "target_layer": { "name": "ui", "level": 6 },
  "violation": {
    "what": "Service layer (L4) imports from UI layer (L6)",
    "why": "Dependencies must flow downward: Types(1)->Config(2)->Repo(3)->Service(4)->Runtime(5)->UI(6). Services must not depend on UI components.",
    "fix": {
      "action": "replace_import",
      "replacement": "import type { OrderFormData } from '../types/order'",
      "steps": [
        "1. Define OrderFormData type in src/types/order.ts",
        "2. Use the type instead of the UI component"
      ]
    }
  }
}
```

**Example response (allowed)**:
```json
{
  "allowed": true,
  "source_layer": { "name": "service", "level": 4 },
  "target_layer": { "name": "repo", "level": 3 }
}
```

---

### Tool 2: `get_layer_rules`

Retrieve the layer configuration and dependency rules for a specific module or file. Agents should call this tool when they need to understand what imports are allowed from a given location.

**When to use**: When an agent is working in a file and needs to know what modules/layers it can import from. Useful at the start of editing a file.

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "file_or_module": {
      "type": "string",
      "description": "Path to a file or module directory (e.g., 'src/services/orderService.ts' or 'src/services')"
    }
  },
  "required": ["file_or_module"]
}
```

**Output Schema**:
```json
{
  "type": "object",
  "properties": {
    "layer": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "level": { "type": "integer" },
        "description": { "type": "string" }
      }
    },
    "module": {
      "type": "object",
      "properties": {
        "path": { "type": "string" },
        "description": { "type": "string" }
      }
    },
    "allowed_imports": {
      "type": "object",
      "properties": {
        "layers": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Layer names this file can import from"
        },
        "modules": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Specific module paths allowed as dependencies"
        },
        "shared": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Shared modules accessible from all layers"
        }
      }
    },
    "forbidden_imports": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Modules explicitly forbidden as dependencies"
    },
    "providers_available": {
      "type": "boolean",
      "description": "Whether the Providers pattern is configured for cross-cutting concerns"
    }
  }
}
```

**Example response**:
```json
{
  "layer": {
    "name": "service",
    "level": 4,
    "description": "Business logic, use cases, orchestration"
  },
  "module": {
    "path": "src/services",
    "description": "Business logic services"
  },
  "allowed_imports": {
    "layers": ["types", "config", "repo"],
    "modules": ["src/repositories", "src/types", "src/config", "src/utils"],
    "shared": ["src/types", "src/utils", "src/constants", "src/config"]
  },
  "forbidden_imports": ["src/api", "src/components", "src/pages"],
  "providers_available": true
}
```

---

### Tool 3: `check_module_boundary`

Verify if a dependency between two modules is allowed by boundary rules. Agents should call this tool when adding a cross-module import.

**When to use**: During coding, when an agent detects that an import crosses module boundaries. This is a more targeted check than `check_layer_violation` — it focuses on module-level allowedDeps/forbiddenDeps.

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "from_module": {
      "type": "string",
      "description": "Module path of the importing file (e.g., 'src/services')"
    },
    "to_module": {
      "type": "string",
      "description": "Module path of the imported file (e.g., 'src/api')"
    },
    "imported_symbol": {
      "type": "string",
      "description": "Optional: the specific symbol being imported (helps generate better fix suggestions)"
    }
  },
  "required": ["from_module", "to_module"]
}
```

**Output Schema**:
```json
{
  "type": "object",
  "properties": {
    "allowed": {
      "type": "boolean"
    },
    "reason": {
      "type": "string",
      "enum": ["in_allowed_deps", "shared_module", "not_in_allowed_deps", "in_forbidden_deps", "isolated_module"],
      "description": "Why the dependency is allowed or disallowed"
    },
    "suggestion": {
      "type": "string",
      "nullable": true,
      "description": "If disallowed, a suggestion for the correct module to import from"
    },
    "alternative_modules": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Modules that are allowed and might contain the needed symbol"
    }
  }
}
```

**Example response (forbidden)**:
```json
{
  "allowed": false,
  "reason": "in_forbidden_deps",
  "suggestion": "Move shared code to 'src/types' or 'src/utils', or use the Providers pattern for cross-cutting concerns",
  "alternative_modules": ["src/types", "src/utils", "src/config"]
}
```

---

### Tool 4: `suggest_correct_layer`

Given a type, function, or file description, suggest which layer it belongs to in the 6-layer model. Agents should call this tool when creating a new file or moving code.

**When to use**: When an agent creates a new file and needs to determine the correct directory placement. Also useful when refactoring code from one layer to another.

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "description": {
      "type": "string",
      "description": "Description of what the code does (e.g., 'fetches user data from PostgreSQL', 'validates order input', 'renders a user profile card')"
    },
    "code_snippet": {
      "type": "string",
      "description": "Optional: a code snippet to analyze for layer classification"
    },
    "file_name": {
      "type": "string",
      "description": "Optional: proposed file name (e.g., 'userRepository.ts', 'OrderForm.tsx')"
    }
  },
  "required": ["description"]
}
```

**Output Schema**:
```json
{
  "type": "object",
  "properties": {
    "suggested_layer": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "level": { "type": "integer" },
        "description": { "type": "string" }
      }
    },
    "suggested_path": {
      "type": "string",
      "description": "Recommended file path based on the layer and naming conventions"
    },
    "confidence": {
      "type": "string",
      "enum": ["high", "medium", "low"]
    },
    "reasoning": {
      "type": "string",
      "description": "Explanation of why this layer was suggested"
    },
    "alternative_layers": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "name": { "type": "string" },
          "reason": { "type": "string" }
        }
      },
      "description": "Other possible layers if the classification is ambiguous"
    },
    "naming_convention": {
      "type": "object",
      "properties": {
        "convention": { "type": "string" },
        "suggested_name": { "type": "string" }
      }
    }
  }
}
```

**Example request**:
```json
{
  "tool": "suggest_correct_layer",
  "arguments": {
    "description": "fetches user data from PostgreSQL database",
    "file_name": "userDataAccess.ts"
  }
}
```

**Example response**:
```json
{
  "suggested_layer": {
    "name": "repo",
    "level": 3,
    "description": "Data access, external API clients, database queries"
  },
  "suggested_path": "src/repositories/userRepository.ts",
  "confidence": "high",
  "reasoning": "Database access is a Repository (Layer 3) concern. The file directly interacts with PostgreSQL which is an infrastructure/data access responsibility.",
  "alternative_layers": [],
  "naming_convention": {
    "convention": "camelCase with Repository suffix",
    "suggested_name": "userRepository.ts"
  }
}
```

---

### Tool 5: `get_architecture_map`

Return the full architecture map for the current project, including all layers, modules, boundaries, and rules. Agents should call this tool once at the start of a session to understand the project's architecture.

**When to use**: At the beginning of a coding session, or when an agent needs a comprehensive understanding of the project architecture. This is a "big picture" tool — call it once and cache the result.

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "include_violations": {
      "type": "boolean",
      "default": false,
      "description": "If true, include current violation count per module"
    },
    "include_metrics": {
      "type": "boolean",
      "default": false,
      "description": "If true, include coupling metrics per module"
    }
  }
}
```

**Output Schema**:
```json
{
  "type": "object",
  "properties": {
    "project": {
      "type": "object",
      "properties": {
        "name": { "type": "string" },
        "type": { "type": "string" },
        "language": { "type": "string" },
        "root_dir": { "type": "string" }
      }
    },
    "layer_model": {
      "type": "object",
      "properties": {
        "type": { "type": "string", "description": "e.g., '6-layer' or 'custom'" },
        "strict": { "type": "boolean" },
        "layers": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": { "type": "string" },
              "level": { "type": "integer" },
              "paths": { "type": "array", "items": { "type": "string" } },
              "description": { "type": "string" }
            }
          }
        }
      }
    },
    "modules": {
      "type": "object",
      "additionalProperties": {
        "type": "object",
        "properties": {
          "layer": { "type": "string" },
          "allowed_deps": { "type": "array", "items": { "type": "string" } },
          "forbidden_deps": { "type": "array", "items": { "type": "string" } },
          "isolated": { "type": "boolean" },
          "file_count": { "type": "integer" },
          "violations": { "type": "integer" },
          "metrics": {
            "type": "object",
            "properties": {
              "ca": { "type": "integer" },
              "ce": { "type": "integer" },
              "instability": { "type": "number" }
            }
          }
        }
      }
    },
    "providers": {
      "type": "object",
      "nullable": true,
      "properties": {
        "enabled": { "type": "boolean" },
        "interface_path": { "type": "string" },
        "concerns": {
          "type": "array",
          "items": { "type": "string" },
          "description": "Cross-cutting concerns managed by Providers (e.g., auth, logging, telemetry, feature-flags)"
        }
      }
    },
    "shared_modules": {
      "type": "array",
      "items": { "type": "string" }
    },
    "dependency_graph": {
      "type": "object",
      "description": "Module-level dependency adjacency list",
      "additionalProperties": {
        "type": "array",
        "items": { "type": "string" }
      }
    },
    "rules_summary": {
      "type": "object",
      "properties": {
        "total_rules": { "type": "integer" },
        "active_rules": { "type": "integer" },
        "rule_types": {
          "type": "object",
          "additionalProperties": { "type": "integer" }
        }
      }
    }
  }
}
```

---

### Tool 6: `lint_file`

Run harness linting on a single file and return agent-friendly violations with fix instructions. This is the MCP entry point for the harness-linter agent.

**When to use**: After modifying a file, or during a code review. Provides the full harness-lint output for a specific file.

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "file_path": {
      "type": "string",
      "description": "Path to the file to lint"
    },
    "format": {
      "type": "string",
      "enum": ["full", "compact", "agent-only"],
      "default": "full"
    }
  },
  "required": ["file_path"]
}
```

**Output Schema**:
```json
{
  "type": "object",
  "properties": {
    "file": { "type": "string" },
    "violations": {
      "type": "array",
      "items": {
        "type": "object",
        "description": "Harness-lint violation object (see harness-linter agent spec)"
      }
    },
    "clean": {
      "type": "boolean",
      "description": "True if no violations found"
    }
  }
}
```

## Resources

The MCP server exposes the following read-only resources:

### Resource: `architecture://rules`
Returns the current `.arch-rules.json` configuration.

### Resource: `architecture://layer-diagram`
Returns a Mermaid diagram of the current layer model with dependency arrows.

### Resource: `architecture://violations`
Returns the most recent violation report (if a check has been run).

## Agent Integration Guide

### Recommended workflow for coding agents

1. **Session start**: Call `get_architecture_map` to understand the project architecture. Cache the result.

2. **Before adding imports**: Call `check_layer_violation` with the source file and proposed import path. If the import is disallowed, use the violation's `fix` field to find an alternative.

3. **Creating new files**: Call `suggest_correct_layer` with a description of the file's purpose. Use the suggested path and naming convention.

4. **Cross-module dependencies**: Call `check_module_boundary` before importing from another module. If disallowed, check the `alternative_modules` list.

5. **After editing**: Call `lint_file` on modified files to catch any violations introduced during the session.

6. **Before committing**: Call `lint_file` on all modified files to ensure no violations are committed.

### Error handling

All tools return standard MCP error responses for:
- `config_not_found`: No `.arch-rules.json` in the project. Suggest running `/arch-init`.
- `invalid_config`: The `.arch-rules.json` has syntax or schema errors.
- `file_not_found`: The specified file does not exist.
- `module_not_classified`: The file does not belong to any defined module (informational, not an error).

### Performance considerations

- `get_architecture_map` reads the full config and optionally scans the project. Cache the result for the session.
- `check_layer_violation` and `check_module_boundary` are lightweight lookups against the config. Safe to call frequently.
- `lint_file` performs full import analysis on a single file. Call on modified files, not the entire project.
- `suggest_correct_layer` uses heuristics and does not scan the filesystem. Fast to call.
