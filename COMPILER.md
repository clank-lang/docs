# Compiler Interface Reference

Complete reference for the Clank compiler interface and output format.

## Contents

- [CLI Commands](#cli-commands)
- [Input Modes](#input-modes)
- [Output Modes](#output-modes)
- [CompileResult Schema](#compileresult-schema)
- [Diagnostics](#diagnostics)
- [Obligations](#obligations)
- [Type Holes](#type-holes)
- [Error Codes](#error-codes)

## CLI Commands

### Basic Commands

```bash
clank compile main.clank -o dist/    # Compile to JavaScript
clank check main.clank               # Type check only
clank run main.clank                 # Compile and execute with Bun
```

### Agent-Oriented Commands

```bash
# Compile from AST JSON
clank compile program.json --input=ast -o dist/

# Compile from patch operations
clank compile patch.json --input=patch

# Output structured JSON diagnostics
clank compile main.clank --emit=json

# Output canonical AST as JSON
clank compile main.clank --emit=ast

# Round-trip: source -> AST -> JavaScript
clank compile main.clank --emit=ast | clank compile --input=ast -o dist/
```

## Input Modes

| Flag | Description | Use Case |
|------|-------------|----------|
| (default) | `.clank` source text | Human authoring, debugging |
| `--input=ast` | Full AST JSON | Initial submission, major restructuring |
| `--input=patch` | List of PatchOps | Iterative refinement (preferred) |

### AST JSON Input

Submit a complete program as JSON:

```json
{
  "kind": "program",
  "declarations": [
    {
      "kind": "fn",
      "name": "main",
      "params": [],
      "returnType": { "kind": "effect", "effect": "IO", "inner": { "kind": "tuple", "elements": [] } },
      "body": { "source": "{ println(\"Hello\") }" }
    }
  ]
}
```

### Patch Input

Submit incremental changes:

```json
{
  "base_ast_hash": "abc123",
  "patches": [
    { "op": "replace_node", "node_id": "n5", "new_node": { ... } },
    { "op": "insert_before", "target_id": "n10", "new_statement": { ... } }
  ]
}
```

## Output Modes

| Flag | Description | Output |
|------|-------------|--------|
| (default) | JavaScript files | `dist/*.js` |
| `--emit=json` | Full CompileResult | Structured JSON with repairs |
| `--emit=ast` | Canonical AST | JSON AST only |
| `--emit=repairs` | Repairs only | Just repair candidates |
| `--emit=js` | JavaScript only | Generated code |
| `--emit=all` | Everything | Full output bundle |

## CompileResult Schema

The compiler always returns this structure when `--emit=json`:

```typescript
interface CompileResult {
  // Overall status
  status: "success" | "incomplete" | "error";

  // Compiler version
  compiler_version: string;

  // The canonical (normalized) AST
  // ALWAYS operate on this, not your original input
  canonical_ast: ProgramAST;

  // Ranked repair suggestions (primary feedback mechanism)
  repairs: RepairCandidate[];

  // Diagnostics with node references
  diagnostics: Diagnostic[];

  // Outstanding proof obligations
  obligations: Obligation[];

  // Unfilled type holes (synthesis requests)
  holes: TypeHole[];

  // Generated artifacts (only if status == "success")
  output?: {
    js: string;              // Generated JavaScript
    js_map?: string;         // Source map
    dts?: string;            // TypeScript declarations
  };

  // Statistics
  stats: CompileStats;
}
```

### Status Values

| Status | Meaning | Next Action |
|--------|---------|-------------|
| `success` | Compilation complete | Use `output.js` |
| `incomplete` | Has unprovable obligations | Apply repairs or add assertions |
| `error` | Has errors | Apply repairs to fix errors |

### Canonical AST

The `canonical_ast` is a normalized version of your input:

- **Desugared**: Unicode operators become ASCII, pipe operators expand
- **Normalized**: Explicit `else` branches, explicit `return` statements
- **Annotated**: Effects marked, validators inserted
- **Idempotent**: Running twice produces same result

**Always apply repairs to `canonical_ast`, not your original input.**

## Diagnostics

```typescript
interface Diagnostic {
  // Unique identifier
  id: string;

  // Severity
  severity: "error" | "warning" | "info" | "hint";

  // Error code (E0xxx, E1xxx, etc.)
  code: string;

  // Human-readable message
  message: string;

  // Primary node where error occurs
  primary_node_id: string;

  // Secondary nodes involved
  secondary_node_ids?: string[];

  // Source location (for debugging)
  location: SourceSpan;

  // Machine-readable structured data
  structured: {
    kind: string;           // e.g., "type_mismatch", "unresolved_name"
    expected?: string;
    actual?: string;
    [key: string]: unknown;
  };

  // IDs of RepairCandidates that address this
  repair_refs: string[];

  // Human-facing hints (secondary to repairs)
  hints: Hint[];
}
```

### Example Diagnostic

```json
{
  "id": "diag_001",
  "severity": "error",
  "code": "E2001",
  "message": "Type mismatch: expected Int, found Str",
  "primary_node_id": "n42",
  "location": {
    "file": "main.clank",
    "start": { "line": 10, "column": 5 },
    "end": { "line": 10, "column": 15 }
  },
  "structured": {
    "kind": "type_mismatch",
    "expected": "Int",
    "actual": "Str"
  },
  "repair_refs": ["repair_001", "repair_002"]
}
```

## Obligations

Proof obligations from refinement types:

```typescript
interface Obligation {
  // Unique identifier
  id: string;

  // Kind of proof needed
  kind: "refinement" | "precondition" | "postcondition" | "effect" | "linearity";

  // The proposition to prove
  goal: string;  // e.g., "d != 0"

  // Node where obligation arises
  primary_node_id: string;

  // Source location
  location: SourceSpan;

  // Available context for proving
  context: {
    bindings: Binding[];
    facts: Fact[];
  };

  // Solver result (required)
  solver_result: "discharged" | "counterexample" | "unknown";

  // Counterexample (if solver_result == "counterexample")
  counterexample?: { [variable: string]: string };

  // Why unknown (if solver_result == "unknown")
  unknown_reason?: {
    category: "incomplete_facts" | "nonlinear" | "quantified" | "timeout" | "unsupported";
    missing_fact?: string;
    description: string;
  };

  // IDs of RepairCandidates that address this
  repair_refs: string[];

  // Human-facing hints
  hints: Hint[];
}
```

### Solver Results

| Result | Meaning | Counterexample |
|--------|---------|----------------|
| `discharged` | Proven true | None |
| `counterexample` | Found violation | Concrete values |
| `unknown` | Cannot prove | Candidate (optional) |

### Example Obligation

```json
{
  "id": "obl_001",
  "kind": "refinement",
  "goal": "y != 0",
  "primary_node_id": "n50",
  "location": { "file": "main.clank", "start": { "line": 8, "column": 11 } },
  "context": {
    "bindings": [
      { "name": "x", "type": "Int", "mutable": false, "source": "let" },
      { "name": "y", "type": "Int", "mutable": false, "source": "let" }
    ],
    "facts": []
  },
  "solver_result": "unknown",
  "unknown_reason": {
    "category": "incomplete_facts",
    "description": "No constraints known about y"
  },
  "repair_refs": ["repair_guard_001", "repair_assert_001"],
  "hints": [
    { "strategy": "guard", "description": "Add if y != 0 { ... }", "confidence": "high" },
    { "strategy": "assert", "description": "Add assert y != 0", "confidence": "medium" }
  ]
}
```

### Counterexamples

When the solver finds a definite violation:

```json
{
  "solver_result": "counterexample",
  "counterexample": {
    "x": "6",
    "_explanation": "Predicate 'x <= 5' contradicts known fact 'x > 5'",
    "_violated": "x <= 5",
    "_contradicts": "x > 5 (from: parameter refinement)"
  }
}
```

## Type Holes

Unfilled type holes as synthesis requests:

```typescript
interface TypeHole {
  // Unique identifier
  id: string;

  // Node ID of the hole
  node_id: string;

  // Source location
  location: SourceSpan;

  // Type that must be satisfied
  goal_type: string;

  // Allowed effects in this context
  allowed_effects: string[];

  // Variables in scope
  in_scope_bindings: Binding[];

  // Candidate fills (as RepairCandidate IDs)
  fill_candidates: string[];

  // Repair references
  repair_refs: string[];
}
```

## Error Codes

### E0xxx - Syntax Errors

| Code | Name | Description |
|------|------|-------------|
| E0001 | UnexpectedToken | Unexpected token in input |
| E0002 | UnterminatedString | String literal not closed |
| E0003 | InvalidNumericLiteral | Invalid number format |
| E0004 | MismatchedBrackets | Brackets don't match |

### E1xxx - Name Resolution

| Code | Name | Description |
|------|------|-------------|
| E1001 | UnresolvedName | Name not found in scope |
| E1002 | DuplicateDefinition | Name already defined |
| E1003 | ImportNotFound | Imported name not found |
| E1004 | ModuleNotFound | Module not found |
| E1005 | UnresolvedType | Type name not found |

### E2xxx - Type Errors

| Code | Name | Description |
|------|------|-------------|
| E2001 | TypeMismatch | Expected type doesn't match actual |
| E2002 | ArityMismatch | Wrong number of arguments |
| E2003 | MissingField | Required field not provided |
| E2004 | UnknownField | Field doesn't exist on type |
| E2005 | NotCallable | Expression is not callable |
| E2006 | NotIndexable | Expression is not indexable |
| E2007 | MissingTypeAnnotation | Type annotation required |
| E2008 | RecursiveTypeWithoutIndirection | Recursive type needs indirection |
| E2013 | ImmutableAssign | Cannot assign to immutable variable |
| E2015 | NonExhaustiveMatch | Match doesn't cover all cases |

### E3xxx - Refinement Errors

| Code | Name | Description |
|------|------|-------------|
| E3001 | UnprovableRefinement | Cannot prove refinement predicate |
| E3002 | PreconditionNotSatisfied | Precondition cannot be proven |
| E3003 | PostconditionNotSatisfied | Postcondition cannot be proven |
| E3004 | AssertionUnprovable | Assertion cannot be proven |

### E4xxx - Effect Errors

| Code | Name | Description |
|------|------|-------------|
| E4001 | EffectNotAllowed | Effect not permitted in context |
| E4002 | UnhandledEffect | Effect not handled |
| E4003 | EffectMismatch | Effect types don't match |

### E5xxx - Linearity Errors

| Code | Name | Description |
|------|------|-------------|
| E5001 | LinearValueNotConsumed | Linear resource leaked |
| E5002 | LinearValueUsedMultipleTimes | Linear resource used twice |
| E5003 | LinearValueEscapesScope | Linear resource escapes |

### W0xxx - Warnings

| Code | Name | Description |
|------|------|-------------|
| W0001 | UnusedVariable | Variable declared but not used |
| W0002 | UnusedImport | Import not used |
| W0003 | UnreachableCode | Code will never execute |
| W0004 | ShadowedVariable | Variable shadows outer binding |

## Supporting Types

```typescript
interface Hint {
  strategy: string;      // e.g., "add_guard", "strengthen_type"
  description: string;
  template?: string;     // Code template
  confidence: "high" | "medium" | "low";
}

interface SourceSpan {
  file: string;
  start: Position;
  end: Position;
  snippet?: string;
}

interface Position {
  line: number;    // 1-indexed
  column: number;  // 1-indexed
  offset: number;  // 0-indexed byte offset
}

interface Binding {
  name: string;
  type: string;
  mutable: boolean;
  source: "parameter" | "let" | "for" | "match";
}

interface Fact {
  proposition: string;
  source: string;  // e.g., "branch_condition:line_42"
}

interface CompileStats {
  source_files: number;
  source_lines: number;
  source_tokens: number;
  output_lines: number;
  output_bytes: number;
  obligations_total: number;
  obligations_discharged: number;
  compile_time_ms: number;
}
```

## Example Full Output

```json
{
  "status": "incomplete",
  "compiler_version": "0.1.0",
  "canonical_ast": { "kind": "program", "declarations": [...] },
  "repairs": [
    {
      "id": "repair_001",
      "title": "Add guard for y != 0",
      "confidence": "high",
      "safety": "likely_preserving",
      "kind": "local_fix",
      "edits": [{ "op": "wrap", "node_id": "n50", ... }],
      "expected_delta": { "obligations_discharged": ["obl_001"] }
    }
  ],
  "diagnostics": [],
  "obligations": [
    {
      "id": "obl_001",
      "kind": "refinement",
      "goal": "y != 0",
      "solver_result": "unknown",
      "repair_refs": ["repair_001"]
    }
  ],
  "holes": [],
  "output": null,
  "stats": {
    "source_files": 1,
    "obligations_total": 1,
    "obligations_discharged": 0,
    "compile_time_ms": 23
  }
}
```
