---
name: clank
description: Generates and compiles Clank programs using AST JSON. Clank is an agent-oriented language where programs are submitted as JSON AST, and the compiler returns machine-actionable repair patches.
---

# Clank Language Skill

Clank is an **agent-first programming language**. The canonical representation is **AST JSON**, not source text. Agents construct JSON AST, submit it to the compiler, receive structured diagnostics with repair patches, and iterate until compilation succeeds.

## Two Representations

| Format | Purpose | Who Uses It |
|--------|---------|-------------|
| **AST JSON** | Compiler communication, iteration, repairs | Agents |
| **`.clank` files** | Human reading, git storage, code review | Humans |

> **Key insight**: Agents work in AST JSON for fast iteration. Once code compiles successfully, serialize to `.clank` for humans and version control.

## Agent Workflow (Primary)

```
┌─────────────────────────────────────────────────────────────────┐
│  DEVELOPMENT LOOP (AST JSON)                                    │
│  1. Construct AST JSON (use source fragments for convenience)   │
│  2. Submit: clank compile program.json --input=ast --emit=json  │
│  3. Receive CompileResult with repairs[] and diagnostics[]      │
│  4. If status == "success": goto FINALIZE                       │
│  5. Apply top repair to canonical_ast                           │
│  6. Goto 2                                                      │
├─────────────────────────────────────────────────────────────────┤
│  FINALIZE (for humans and git)                                  │
│  7. Write .clank file from canonical_ast                        │
│  8. Commit to git, deploy output.js                             │
└─────────────────────────────────────────────────────────────────┘
```

**Critical rule**: Always operate on `canonical_ast` from the compiler response, never your original input.

### Loading Existing Code

To modify existing `.clank` files, first convert to AST:
```bash
clank compile existing.clank --emit=ast > program.json
# Now iterate on program.json in AST mode
```

## AST JSON Quick Start

### Minimal Program

```json
{
  "kind": "program",
  "declarations": [
    {
      "kind": "fn",
      "name": "main",
      "params": [],
      "returnType": { "kind": "named", "name": "Int" },
      "body": { "kind": "literal", "value": { "kind": "int", "value": "42" } }
    }
  ]
}
```

### Source Fragments (Recommended)

Any AST node can use `{ "source": "..." }` for convenience. This is the sweet spot for agents—structured where it matters, text where it's easier:

```json
{
  "kind": "fn",
  "name": "add",
  "params": [
    { "name": "a", "type": { "source": "Int" } },
    { "name": "b", "type": { "source": "Int" } }
  ],
  "returnType": { "source": "Int" },
  "body": { "source": "a + b" }
}
```

**Use source fragments for**: types, expressions, function bodies
**Use full AST for**: programmatic generation, when you need node IDs for repairs

## Repair Loop

When compilation fails, the compiler returns ranked `repairs[]` with machine-applicable patches.

### Selection Priority

1. **Safety**: `behavior_preserving` > `likely_preserving` > `behavior_changing`
2. **Confidence**: `high` > `medium` > `low`
3. **Scope**: `local_fix` > `refactor`

Never apply `behavior_changing` repairs without user approval.

### Example Repair

```json
{
  "id": "repair_001",
  "title": "Add IO effect to 'main'",
  "confidence": "medium",
  "safety": "likely_preserving",
  "edits": [{ "op": "widen_effect", "fn_id": "n8", "add_effects": ["IO"] }],
  "expected_delta": { "diagnostics_resolved": ["d1"] }
}
```

Apply the `edits` to `canonical_ast`, then recompile.

## Key AST Patterns

### Function with Effects

```json
{
  "kind": "fn",
  "name": "greet",
  "params": [{ "name": "name", "type": { "source": "String" } }],
  "returnType": { "source": "IO + Unit" },
  "body": { "source": "println(\"Hello, \" ++ name)" }
}
```

### Record Type

```json
{
  "kind": "rec",
  "name": "User",
  "fields": [
    { "name": "id", "type": { "source": "Int{self > 0}" } },
    { "name": "name", "type": { "source": "String" } }
  ]
}
```

### External Function (JS Interop)

```json
{
  "kind": "externalFn",
  "name": "now",
  "params": [],
  "returnType": { "source": "Int" },
  "jsName": "Date.now"
}
```

## Reserved Words

**Do not use as identifiers**: `fn`, `let`, `mut`, `if`, `else`, `match`, `for`, `while`, `loop`, `return`, `break`, `continue`, `type`, `rec`, `sum`, `mod`, `use`, `external`, `pre`, `post`, `assert`, `unsafe`, `js`, `true`, `false`

> ⚠️ `post` is commonly mistaken for a valid field name—it's reserved for postconditions.

## CLI Reference

```bash
# Agent workflow (primary)
clank compile program.json --input=ast --emit=json

# Load existing .clank to AST
clank compile main.clank --emit=ast > program.json

# Human workflow (debugging)
clank compile main.clank              # Compile to JS
clank check main.clank                # Type check only
clank run main.clank                  # Compile and execute
```

## Skill Files

| File | Purpose |
|------|---------|
| [COMPILER.md](COMPILER.md) | CompileResult schema, error codes, CLI flags |
| [REPAIRS.md](REPAIRS.md) | Repair system, PatchOp types, selection strategy |
| [SYNTAX.md](SYNTAX.md) | Grammar, operators, keywords (for source fragments) |
| [TYPES.md](TYPES.md) | Type system, refinements, effects |
| [EXAMPLES.md](EXAMPLES.md) | Annotated code examples |
| [IDIOMS.md](IDIOMS.md) | Patterns and anti-patterns |

## Quick Syntax Reference (for source fragments)

| Construct | Syntax |
|-----------|--------|
| Function | `fn name(x: T) -> R { body }` |
| Effect | `fn name() -> IO + Unit { ... }` |
| Refinement | `Int{self > 0}` |
| Record | `rec Name { field: Type }` |
| Sum type | `sum Name { A(T), B }` |
| Type alias | `type Name = T` |
| Lambda | `\(x: T) -> expr` |
| Match | `match x { A -> ..., B -> ... }` |

### Unicode Operators (optional)

| Unicode | ASCII | Meaning |
|---------|-------|---------|
| `ƒ` | `fn` | function |
| `→` | `->` | returns |
| `λ` | `\` | lambda |
| `×` | `*` | multiply |
| `≠` | `!=` | not equal |
| `≤` `≥` | `<=` `>=` | comparison |
| `∧` | `&&` | and |
| `∨` | `\|\|` | or |

## What Agents Should Do

- ✅ Work in AST JSON with source fragments
- ✅ Submit with `--input=ast --emit=json`
- ✅ Apply compiler-suggested repairs to `canonical_ast`
- ✅ Iterate until `status == "success"`
- ✅ Write `.clank` files for git/humans when done

## What Agents Should NOT Do

- ❌ Write `.clank` files directly during iteration
- ❌ Invent fixes when compiler repairs exist
- ❌ Apply `behavior_changing` repairs without approval
- ❌ Modify original input instead of `canonical_ast`

## Links

- Repository: https://github.com/clank-lang/clank
- Skill repo: https://github.com/clank-lang/skill
