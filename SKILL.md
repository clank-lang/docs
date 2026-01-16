---
name: clank
description: Generates and compiles Clank programs using AST JSON. Clank is an agent-oriented language where programs are submitted as JSON AST, and the compiler returns machine-actionable repair patches. Use when writing Clank code, working with .clank files, or when the user mentions Clank, refinement types, or agent-oriented compilation.
---

# Clank Language Skill

Clank is an agent-first programming language designed for LLMs, not humans. It features Unicode syntax, refinement types, and structured compiler output optimized for AI consumption.

## When to Use This Skill

Use this skill when:
- Writing new Clank code
- Debugging or repairing Clank programs
- Translating code from other languages into Clank
- Understanding Clank compiler output or error messages

## Setup

If Clank isn't installed, help the user set it up:

```bash
# Install Clank globally with bun
bun install -g clank-lang

# Verify installation
clank --version
```

If bun isn't installed, the user can get it from [bun.sh](https://bun.sh).

## Skill Files

| File | Purpose |
|------|---------|
| `SYNTAX.md` | Unicode operators, keywords, and grammar |
| `EXAMPLES.md` | Annotated code examples for common patterns |
| `IDIOMS.md` | Idiomatic Clank — preferred patterns and anti-patterns |
| `TYPES.md` *(planned)* | Type system reference — refinement types, dependent types, type inference |
| `AST-JSON.md` *(planned)* | AST structure for reading/writing Clank's JSON intermediate representation |
| `COMPILER.md` *(planned)* | Compiler phases, flags, and output formats |
| `REPAIRS.md` *(planned)* | Error recovery patterns — how to fix common compiler errors |

## Quick Reference

### File Extension
`.clank`

### Running the Compiler
```bash
clank compile <file.clank> [--emit ast|ir|js] [--check-only]
```

### Core Syntax at a Glance

```clank
// Function definition with refinement type
ƒ factorial(n: ℕ) → ℕ where n ≥ 0 {
  match n {
    0 → 1
    _ → n × factorial(n - 1)
  }
}

// Type alias with constraint
τ PositiveInt = ℤ where self > 0

// Record type
τ User = {
  name: String,
  age: ℕ where self ≥ 0 ∧ self ≤ 150
}
```

### Unicode Operator Quick Reference

| Symbol | Meaning | ASCII Fallback |
|--------|---------|----------------|
| `ƒ` | function | `fn` |
| `τ` | type | `type` |
| `→` | returns / maps to | `->` |
| `×` | multiply | `*` |
| `÷` | divide | `/` |
| `≠` | not equal | `!=` |
| `≤` `≥` | comparison | `<=` `>=` |
| `∧` | logical and | `&&` |
| `∨` | logical or | `\|\|` |
| `¬` | logical not | `!` |
| `∈` | element of | `in` |
| `∀` | for all | `forall` |
| `∃` | exists | `exists` |
| `λ` | lambda | `\` |
| `ℕ` | natural numbers | `Nat` |
| `ℤ` | integers | `Int` |
| `ℝ` | reals | `Real` |
| `𝔹` | booleans | `Bool` |

## Workflow

### Writing New Code

1. Start with types — define your data structures and constraints first
2. Write function signatures with refinement types
3. Implement function bodies
4. Run `clank compile --check-only` to verify types
5. If errors occur, consult `REPAIRS.md` for fix patterns

### Debugging Compiler Errors

1. Read the structured JSON error output
2. Look up the error code in `REPAIRS.md`
3. Apply the suggested fix pattern
4. Re-run compiler

### Reading Existing Code

1. Start with type definitions (`τ` declarations)
2. Trace function signatures before bodies
3. Use `--emit ast` to get machine-readable structure if needed

## Key Concepts

### Refinement Types
Types can have predicates that constrain their values at compile time:
```clank
τ Percentage = ℝ where self ≥ 0.0 ∧ self ≤ 100.0
```

### Structured Errors
Compiler outputs JSON errors designed for LLM consumption:
```json
{
  "error": "E0042",
  "message": "Type constraint violation",
  "location": {"line": 12, "col": 5},
  "expected": "ℕ where self > 0",
  "actual": "ℕ",
  "suggestion": "Add guard: if n > 0 then ..."
}
```

### No Implicit Coercion
Clank never silently converts types. All conversions must be explicit.

## Core Workflow (Agent Mode)

```
1. Construct AST JSON (or use source fragments)
2. Submit: clank compile program.json --input=ast --emit=json
3. Receive CompileResult with repairs[] and diagnostics[]
4. Apply repairs using PatchOps on canonical_ast
5. Resubmit until status == "success"
```

**Key principle**: Always operate on `canonical_ast` from the compiler response, not your original input.

### AST JSON Example

A simple function in AST JSON format:

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
      "body": { "kind": "binary", "op": "+", "left": { "kind": "ident", "name": "a" }, "right": { "kind": "ident", "name": "b" } }
    }
  ]
}
```

**Source fragments**: Any node can use `{ "source": "..." }` for convenience:
```json
{ "body": { "source": "a + b" } }
```

## Links

- Main repository: https://github.com/clank-lang/clank
- Language specification: https://github.com/clank-lang/clank/blob/main/spec/

## Reference Documentation

- [SYNTAX.md](SYNTAX.md) - Complete grammar and operator reference
- [EXAMPLES.md](EXAMPLES.md) - Annotated code examples
- [IDIOMS.md](IDIOMS.md) - Idiomatic patterns and anti-patterns
- TYPES.md *(planned)* - Type system reference
- AST-JSON.md *(planned)* - Complete AST node schema
- COMPILER.md *(planned)* - Compiler interface and output format
- REPAIRS.md *(planned)* - Repair system and PatchOp reference
