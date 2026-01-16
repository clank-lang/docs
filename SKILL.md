---
name: clank
description: Generates and compiles Clank programs using AST JSON. Clank is an agent-oriented language where programs are submitted as JSON AST, and the compiler returns machine-actionable repair patches. Use when writing Clank code, working with .clank files, or when the user mentions Clank, refinement types, or agent-oriented compilation.
---

# Clank Language

Clank is an agent-oriented programming language where **AST JSON is canonical**. Submit programs as JSON, receive structured compiler feedback with machine-actionable repairs.

## Quick Start

### Compile from source (human debugging)
```bash
clank compile main.clank -o dist/
clank check main.clank          # Type check only
clank run main.clank            # Compile and execute
```

### Compile from AST JSON (agent workflow)
```bash
clank compile program.json --input=ast -o dist/
clank compile program.json --input=ast --emit=json  # Get structured diagnostics
```

## Core Workflow

```
1. Construct AST JSON (or use source fragments)
2. Submit: clank compile program.json --input=ast --emit=json
3. Receive CompileResult with repairs[] and diagnostics[]
4. Apply repairs using PatchOps on canonical_ast
5. Resubmit until status == "success"
```

**Key principle**: Always operate on `canonical_ast` from the compiler response, not your original input.

## AST JSON Basics

Every program is a JSON object with `kind: "program"`:

```json
{
  "kind": "program",
  "declarations": [
    {
      "kind": "fn",
      "name": "add",
      "params": [
        { "name": "a", "type": { "kind": "named", "name": "Int" } },
        { "name": "b", "type": { "kind": "named", "name": "Int" } }
      ],
      "returnType": { "kind": "named", "name": "Int" },
      "body": { "kind": "block", "statements": [], "expr": { "source": "a + b" } }
    }
  ]
}
```

**Source fragments**: Any node can use `{ "source": "..." }` for convenience:
```json
{ "body": { "source": "{ a + b }" } }
```

For complete AST schema, see [AST-JSON.md](AST-JSON.md).

## Type System Overview

| Type | Description |
|------|-------------|
| `Int`, `Float`, `Bool`, `Str` | Primitives |
| `[T]` | Array |
| `(T, U)` | Tuple |
| `{name: Str, age: Int}` | Record |
| `(T) -> U` | Function |
| `Int{x > 0}` | Refinement type |
| `IO[T]`, `Err[E, T]` | Effect types |

For complete type reference, see [TYPES.md](TYPES.md).

## Compiler Output

The compiler returns `CompileResult`:

```json
{
  "status": "success" | "incomplete" | "error",
  "canonical_ast": { ... },
  "repairs": [ ... ],
  "diagnostics": [ ... ],
  "obligations": [ ... ],
  "output": { "js": "..." }
}
```

| Field | Purpose |
|-------|---------|
| `canonical_ast` | Normalized AST - always operate on this |
| `repairs` | Ranked repair candidates with PatchOps |
| `diagnostics` | Errors/warnings with node IDs |
| `obligations` | Proof obligations with solver results |
| `output.js` | Generated JavaScript (if successful) |

For complete compiler interface, see [COMPILER.md](COMPILER.md).

## Repair System

When errors occur, the compiler provides machine-actionable repairs:

```json
{
  "id": "repair_001",
  "title": "Rename 'helo' to 'hello'",
  "confidence": "high",
  "safety": "behavior_changing",
  "edits": [{ "op": "rename_symbol", "node_id": "n5", "old_name": "helo", "new_name": "hello" }],
  "expected_delta": { "diagnostics_resolved": ["d1"] }
}
```

**Safety classifications**:
- `behavior_preserving`: Apply automatically (e.g., adding type annotation)
- `likely_preserving`: Apply by default (e.g., guard insertion)
- `behavior_changing`: Require approval (e.g., changing logic)

**Repair priority**:
1. Prefer compiler-suggested repairs over manual edits
2. Prefer `behavior_preserving` > `likely_preserving` > `behavior_changing`
3. Prefer `high` confidence > `medium` > `low`
4. Prefer `local_fix` > `refactor` > `semantics_change`

For complete repair documentation, see [REPAIRS.md](REPAIRS.md).

## Common Patterns

### Function Declaration
```json
{
  "kind": "fn",
  "name": "factorial",
  "params": [{ "name": "n", "type": { "kind": "named", "name": "Int" } }],
  "returnType": { "kind": "named", "name": "Int" },
  "body": {
    "kind": "if",
    "condition": { "kind": "binary", "op": "<=", "left": { "kind": "ident", "name": "n" }, "right": { "kind": "literal", "value": { "kind": "int", "value": "1" } } },
    "thenBranch": { "kind": "block", "statements": [], "expr": { "kind": "literal", "value": { "kind": "int", "value": "1" } } },
    "elseBranch": { "kind": "block", "statements": [], "expr": { "source": "n * factorial(n - 1)" } }
  }
}
```

### Refinement Type
```json
{
  "kind": "refined",
  "base": { "kind": "named", "name": "Int" },
  "predicate": { "source": "x > 0" }
}
```

### Effect Type
```json
{
  "kind": "effect",
  "effect": "IO",
  "inner": { "kind": "tuple", "elements": [] }
}
```

### Record Type
```json
{
  "kind": "rec",
  "name": "User",
  "fields": [
    { "name": "id", "type": { "kind": "named", "name": "Int" } },
    { "name": "name", "type": { "kind": "named", "name": "Str" } }
  ]
}
```

### Sum Type (Tagged Union)
```json
{
  "kind": "sum",
  "name": "Option",
  "typeParams": [{ "name": "T" }],
  "variants": [
    { "name": "Some", "fields": [{ "kind": "named", "name": "T" }] },
    { "name": "None", "fields": [] }
  ]
}
```

## Error Codes

| Range | Category |
|-------|----------|
| E0xxx | Syntax errors |
| E1xxx | Name resolution |
| E2xxx | Type errors |
| E3xxx | Refinement errors |
| E4xxx | Effect errors |
| E5xxx | Linearity errors |

## Toolchain

Clank uses mise to manage Bun:

```bash
mise run install     # Install dependencies
mise run check       # Type check
mise run test        # Run tests
mise run dev <file>  # Dev mode
```

**Important**: Don't install Bun system-wide. Use `mise exec -- bun ...` or `mise run ...`.

## Reference Documentation

- [AST-JSON.md](AST-JSON.md) - Complete AST node schema
- [TYPES.md](TYPES.md) - Type system reference
- [COMPILER.md](COMPILER.md) - Compiler interface and output format
- [REPAIRS.md](REPAIRS.md) - Repair system and PatchOp reference
