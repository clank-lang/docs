# Repair System Reference

Complete reference for Clank's machine-actionable repair system.

## Contents

- [Overview](#overview)
- [RepairCandidate Schema](#repaircandidate-schema)
- [PatchOp Types](#patchop-types)
- [Repair Strategy](#repair-strategy)
- [Repair Compatibility](#repair-compatibility)
- [Implemented Repairs](#implemented-repairs)
- [Examples](#examples)

## Overview

The Clank compiler is a **repair engine**. When errors occur, it provides ranked repair candidates with machine-applicable patches that agents can directly apply.

**Key principle**: Treat repair candidates as authoritative. The compiler has analyzed the program and produced fixes. Manual edits should only be introduced when no suitable compiler-provided repair exists.

## RepairCandidate Schema

```typescript
interface RepairCandidate {
  // Unique identifier
  id: string;

  // Short label for the fix
  title: string;

  // Confidence level
  confidence: "high" | "medium" | "low";

  // Safety classification (CRITICAL for agent trust)
  safety: "behavior_preserving" | "likely_preserving" | "behavior_changing";

  // Category of repair
  kind: "local_fix" | "refactor" | "boundary_validation" | "semantics_change";

  // Scope of the repair
  scope: {
    node_count: number;        // How many nodes are touched
    crosses_function: boolean; // Does it affect multiple functions?
  };

  // What this repair targets
  targets: {
    node_ids?: string[];        // AST nodes affected
    diagnostic_codes?: string[]; // Diagnostics this should resolve
    obligation_ids?: string[];   // Obligations this should discharge
    hole_ids?: string[];         // Holes this should fill
  };

  // Optional preconditions
  preconditions?: Precondition[];

  // The actual edits to apply
  edits: PatchOp[];

  // What should change after applying (REQUIRED)
  expected_delta: {
    diagnostics_resolved: string[];
    obligations_discharged: string[];
    holes_filled: string[];
  };

  // Human-readable explanation
  rationale: string;
}
```

### Safety Classifications

| Safety | Meaning | Agent Behavior |
|--------|---------|----------------|
| `behavior_preserving` | Semantics unchanged | Apply automatically |
| `likely_preserving` | High confidence semantics unchanged | Apply by default |
| `behavior_changing` | May alter runtime behavior | Require explicit approval |

**Examples**:
- `behavior_preserving`: Adding type annotation, renaming to `_unused`
- `likely_preserving`: Adding guard, inserting assertion
- `behavior_changing`: Changing logic, renaming symbols, adding fields

### Kind Classifications

| Kind | Description | Preference |
|------|-------------|------------|
| `local_fix` | Affects small number of nodes | Preferred |
| `refactor` | Restructures across functions | Lower |
| `boundary_validation` | Adds runtime validation | Medium |
| `semantics_change` | Changes program behavior | Lowest |

## PatchOp Types

### replace_node

Replace an entire node with a new node:

```json
{
  "op": "replace_node",
  "node_id": "n42",
  "new_node": {
    "kind": "binary",
    "op": "+",
    "left": { "kind": "ident", "name": "x" },
    "right": { "kind": "literal", "value": { "kind": "int", "value": "1" } }
  }
}
```

### insert_before

Insert a statement before another:

```json
{
  "op": "insert_before",
  "target_id": "stmt_10",
  "new_statement": {
    "kind": "assert",
    "condition": { "source": "x > 0" }
  }
}
```

### insert_after

Insert a statement after another:

```json
{
  "op": "insert_after",
  "target_id": "stmt_10",
  "new_statement": {
    "kind": "let",
    "pattern": { "kind": "ident", "name": "y" },
    "init": { "source": "x + 1" }
  }
}
```

### wrap

Wrap a node in a new construct:

```json
{
  "op": "wrap",
  "node_id": "expr_42",
  "wrapper": {
    "kind": "if",
    "condition": { "source": "d != 0" },
    "thenBranch": {
      "kind": "block",
      "statements": [],
      "expr": { "hole_ref": "wrapped_expr" }
    },
    "elseBranch": { "source": "{ panic(\"division by zero\") }" }
  },
  "hole_ref": "wrapped_expr"
}
```

The `hole_ref` indicates where the original expression is placed.

### delete_node

Delete a node:

```json
{
  "op": "delete_node",
  "node_id": "stmt_15"
}
```

### widen_effect

Add effects to a function signature:

```json
{
  "op": "widen_effect",
  "fn_id": "fn_main",
  "add_effects": ["IO"]
}
```

### rename_symbol

Rename an identifier symbol:

```json
{
  "op": "rename_symbol",
  "node_id": "n5",
  "old_name": "helo",
  "new_name": "hello"
}
```

### rename

Rename a symbol by ID (used for non-identifier renames):

```json
{
  "op": "rename",
  "symbol_id": "sym_1",
  "new_name": "updated"
}
```

### rename_field

Rename a record field:

```json
{
  "op": "rename_field",
  "node_id": "n10",
  "old_name": "nmae",
  "new_name": "name"
}
```

### add_field

Add a field to a record:

```json
{
  "op": "add_field",
  "type_id": "rec_user",
  "field": {
    "name": "email",
    "type": { "kind": "named", "name": "Str" }
  }
}
```

### add_param

Add a parameter to a function:

```json
{
  "op": "add_param",
  "fn_id": "fn_process",
  "param": {
    "name": "config",
    "type": { "kind": "named", "name": "Config" }
  },
  "position": 0
}
```

### add_refinement

Add a refinement predicate to a type:

```json
{
  "op": "add_refinement",
  "type_id": "type_42",
  "predicate": { "source": "x > 0" }
}
```

## Repair Strategy

### Priority Order

Optimize for monotonic progress:

1. **Errors** — Must be zero before proceeding
2. **Obligations** — Discharge proof obligations
3. **Holes** — Fill type holes
4. **Warnings** — Address if relevant

### Repair Selection

1. **Prefer compiler-suggested repairs** — Never invent manual edits when repairs exist
2. **Prefer behavior-preserving** — `behavior_preserving` > `likely_preserving` > `behavior_changing`
3. **Prefer high confidence** — `high` > `medium` > `low`
4. **Prefer local fixes** — `local_fix` > `refactor` > `semantics_change`
5. **Prefer smaller scope** — Fewer `node_count`, no `crosses_function`
6. **Check expected_delta** — Choose repairs that resolve the most issues

### Application Rules

1. **Apply one repair at a time** (or small batch if marked compatible)
2. **Recompile after each repair** — Always operate on fresh `canonical_ast`
3. **Verify expected_delta** — If repair didn't resolve what it claimed, investigate
4. **Never modify original input** — Always patch the `canonical_ast`
5. **Never manually optimize output** — Code quality is the compiler's responsibility

### What Agents Should NOT Do

- Don't invent manual fixes when compiler repairs exist
- Don't refactor or optimize generated TypeScript
- Don't apply behavior-changing repairs without explicit approval
- Don't guess at fixes when no repair candidates exist — ask for clarification

### Example Workflow

```
1. Submit full AST → receive CompileResult
2. If status == "success": done
3. Filter repairs: behavior_preserving or likely_preserving only
4. Sort remaining by: confidence → kind → scope
5. Apply top repair via PatchOp
6. Recompile with --input=ast (updated canonical_ast)
7. Goto 2
```

## Repair Compatibility

Repairs include compatibility metadata that enables batch application, reducing agent↔compiler iterations.

### Compatibility Schema

```typescript
interface RepairCompatibility {
  /** Repairs that cannot be applied together with this one */
  conflicts_with?: string[];  // repair IDs

  /** Repairs that must be applied before this one */
  requires?: string[];  // repair IDs

  /** Repairs with the same batch_key commute and can be applied together */
  batch_key?: string;
}
```

### Compatibility Rules

| Rule | Description |
|------|-------------|
| **Same diagnostic** | Multiple repairs for the same diagnostic conflict (they're alternatives) |
| **Same node** | Repairs touching the same `node_id` conflict |
| **Effect widening** | Multiple `widen_effect` on the same function conflict |
| **Disjoint renames** | `rename_symbol` repairs with disjoint targets share a `batch_key` |
| **Cascading fixes** | Child repairs `require` parent repairs |

### Batch Application

When applying a compatible batch:

```
1. Check compatibility.conflicts_with — exclude conflicting repairs
2. Check compatibility.requires — apply prerequisites first
3. Group by batch_key — repairs with same key can be applied together
4. Apply all repairs in batch to canonical_ast
5. Recompile once
6. Verify: combined expected_delta achieved
```

### Compatibility Example

```json
{
  "repairs": [
    {
      "id": "rc1",
      "title": "Rename 'alph' to 'alpha'",
      "edits": [{ "op": "rename_symbol", "node_id": "n5", "old_name": "alph", "new_name": "alpha" }],
      "compatibility": {
        "conflicts_with": ["rc2"],
        "batch_key": "batch:local_fix:rename_symbol"
      }
    },
    {
      "id": "rc2",
      "title": "Rename 'alph' to 'alps'",
      "edits": [{ "op": "rename_symbol", "node_id": "n5", "old_name": "alph", "new_name": "alps" }],
      "compatibility": {
        "conflicts_with": ["rc1"]
      }
    },
    {
      "id": "rc3",
      "title": "Rename 'bet' to 'beta'",
      "edits": [{ "op": "rename_symbol", "node_id": "n8", "old_name": "bet", "new_name": "beta" }],
      "compatibility": {
        "batch_key": "batch:local_fix:rename_symbol"
      }
    }
  ]
}
```

In this example:
- `rc1` and `rc2` conflict (alternatives for same diagnostic)
- `rc1` and `rc3` can be batched (same `batch_key`, disjoint nodes)

## Implemented Repairs

| Error Code | Error | Repair | Safety | Confidence |
|------------|-------|--------|--------|------------|
| E1001 | UnresolvedName | `rename_symbol` to similar name | behavior_changing | high/medium |
| E1005 | UnresolvedType | `rename_symbol` to similar type | behavior_changing | high/medium |
| E2001 | TypeMismatch | `wrap` with type conversion | behavior_changing | high/medium |
| E2002 | ArityMismatch | `replace_node` add/remove args | behavior_changing | medium |
| E2003 | MissingField | `replace_node` add field | behavior_changing | high |
| E2004 | UnknownField | `rename_field` to similar field | behavior_changing | high/medium |
| E2013 | ImmutableAssign | `replace_node` adding `mut` | behavior_preserving | high |
| E2015 | NonExhaustiveMatch | `replace_node` add wildcard arm | likely_preserving | medium |
| E3001 | UnprovableRefinement | `wrap`/`insert_before` from hints | likely_preserving | high/medium |
| E4001 | EffectNotAllowed | `widen_effect` adding effect | likely_preserving | medium |
| E4002 | UnhandledEffect | `widen_effect` adding Err | likely_preserving | medium |
| W0001 | UnusedVariable | `replace_node` prefix underscore | behavior_preserving | high |

## Examples

### Example 1: Unresolved Name

**Error**: `E1001 - UnresolvedName: 'helo' is not defined`

```json
{
  "id": "repair_001",
  "title": "Rename 'helo' to 'hello'",
  "confidence": "high",
  "safety": "behavior_changing",
  "kind": "local_fix",
  "scope": { "node_count": 1, "crosses_function": false },
  "targets": {
    "node_ids": ["n5"],
    "diagnostic_codes": ["E1001"]
  },
  "edits": [{
    "op": "rename_symbol",
    "node_id": "n5",
    "old_name": "helo",
    "new_name": "hello"
  }],
  "expected_delta": {
    "diagnostics_resolved": ["diag_001"],
    "obligations_discharged": [],
    "holes_filled": []
  },
  "rationale": "'helo' is not defined. Did you mean 'hello'? (edit distance: 1)"
}
```

### Example 2: Unprovable Refinement

**Error**: `E3001 - Cannot prove: y != 0`

```json
{
  "id": "repair_guard_001",
  "title": "Add guard for y != 0",
  "confidence": "high",
  "safety": "likely_preserving",
  "kind": "local_fix",
  "scope": { "node_count": 2, "crosses_function": false },
  "targets": {
    "obligation_ids": ["obl_001"]
  },
  "edits": [{
    "op": "wrap",
    "node_id": "expr_50",
    "wrapper": {
      "kind": "if",
      "condition": { "source": "y != 0" },
      "thenBranch": {
        "kind": "block",
        "statements": [],
        "expr": { "hole_ref": "wrapped" }
      },
      "elseBranch": { "source": "{ panic(\"y must not be zero\") }" }
    },
    "hole_ref": "wrapped"
  }],
  "expected_delta": {
    "diagnostics_resolved": [],
    "obligations_discharged": ["obl_001"],
    "holes_filled": []
  },
  "rationale": "Add guard to prove y != 0 before division"
}
```

### Example 3: Effect Not Allowed

**Error**: `E4001 - Effect 'IO' not allowed in pure function`

```json
{
  "id": "repair_effect_001",
  "title": "Add IO effect to function",
  "confidence": "medium",
  "safety": "likely_preserving",
  "kind": "local_fix",
  "scope": { "node_count": 1, "crosses_function": false },
  "targets": {
    "diagnostic_codes": ["E4001"],
    "node_ids": ["fn_001"]
  },
  "edits": [{
    "op": "widen_effect",
    "fn_id": "fn_001",
    "add_effects": ["IO"]
  }],
  "expected_delta": {
    "diagnostics_resolved": ["diag_002"],
    "obligations_discharged": [],
    "holes_filled": []
  },
  "rationale": "Function calls println() which requires IO effect"
}
```

### Example 4: Missing Field

**Error**: `E2003 - Missing field 'email' in record`

```json
{
  "id": "repair_field_001",
  "title": "Add missing field 'email'",
  "confidence": "high",
  "safety": "behavior_changing",
  "kind": "local_fix",
  "scope": { "node_count": 1, "crosses_function": false },
  "targets": {
    "diagnostic_codes": ["E2003"],
    "node_ids": ["record_42"]
  },
  "edits": [{
    "op": "replace_node",
    "node_id": "record_42",
    "new_node": {
      "kind": "record",
      "fields": [
        { "name": "name", "value": { "source": "\"John\"" } },
        { "name": "age", "value": { "source": "30" } },
        { "name": "email", "value": { "source": "\"\"" } }
      ]
    }
  }],
  "expected_delta": {
    "diagnostics_resolved": ["diag_003"],
    "obligations_discharged": [],
    "holes_filled": []
  },
  "rationale": "Record literal is missing required field 'email'"
}
```

### Example 5: Immutable Assignment

**Error**: `E2013 - Cannot assign to immutable variable 'x'`

```json
{
  "id": "repair_mut_001",
  "title": "Make 'x' mutable",
  "confidence": "high",
  "safety": "behavior_preserving",
  "kind": "local_fix",
  "scope": { "node_count": 1, "crosses_function": false },
  "targets": {
    "diagnostic_codes": ["E2013"],
    "node_ids": ["let_x"]
  },
  "edits": [{
    "op": "replace_node",
    "node_id": "let_x",
    "new_node": {
      "kind": "let",
      "pattern": { "kind": "ident", "name": "x" },
      "mutable": true,
      "value": { "source": "0" }
    }
  }],
  "expected_delta": {
    "diagnostics_resolved": ["diag_004"],
    "obligations_discharged": [],
    "holes_filled": []
  },
  "rationale": "Variable 'x' is reassigned but declared without 'mut'"
}
```
