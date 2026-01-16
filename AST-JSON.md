# AST JSON Reference

Complete schema for Clank AST nodes. Every node has a `kind` field identifying its type.

## Contents

- [Program Structure](#program-structure)
- [Declarations](#declarations)
- [Expressions](#expressions)
- [Statements](#statements)
- [Types](#types)
- [Patterns](#patterns)
- [Literals](#literals)
- [Source Fragments](#source-fragments)

## Program Structure

```json
{
  "kind": "program",
  "declarations": [ /* Declaration[] */ ]
}
```

## Declarations

### Function Declaration

```json
{
  "kind": "fn",
  "name": "functionName",
  "typeParams": [{ "name": "T", "constraint": null }],  // optional
  "params": [
    { "name": "paramName", "type": { /* TypeExpr */ } }
  ],
  "returnType": { /* TypeExpr */ },
  "precondition": { /* Expr */ },   // optional
  "postcondition": { /* Expr */ },  // optional
  "body": { /* BlockExpr */ }
}
```

### Record Declaration

```json
{
  "kind": "rec",
  "name": "RecordName",
  "typeParams": [{ "name": "T" }],  // optional
  "fields": [
    { "name": "fieldName", "type": { /* TypeExpr */ } }
  ]
}
```

### Sum Type Declaration

```json
{
  "kind": "sum",
  "name": "SumName",
  "typeParams": [{ "name": "T" }],  // optional
  "variants": [
    { "name": "VariantName", "fields": [{ /* TypeExpr */ }] }
  ]
}
```

### Type Alias

```json
{
  "kind": "typeAlias",
  "name": "AliasName",
  "typeParams": [{ "name": "T" }],  // optional
  "type": { /* TypeExpr */ }
}
```

### External Function

```json
{
  "kind": "externalFn",
  "name": "externalName",
  "params": [
    { "name": "paramName", "type": { /* TypeExpr */ } }
  ],
  "returnType": { /* TypeExpr */ },
  "jsName": "console.log"
}
```

### Use Declaration

```json
{
  "kind": "use",
  "path": ["std", "io"],
  "names": ["print", "println"]  // optional, imports all if omitted
}
```

### Module Declaration

```json
{
  "kind": "mod",
  "name": "moduleName"
}
```

## Expressions

### Literal

```json
{
  "kind": "literal",
  "value": { /* LiteralValue */ }
}
```

### Identifier

```json
{
  "kind": "ident",
  "name": "variableName"
}
```

### Binary Operation

```json
{
  "kind": "binary",
  "op": "+",  // +, -, *, /, %, ^, ==, !=, <, >, <=, >=, &&, ||, ++, |>
  "left": { /* Expr */ },
  "right": { /* Expr */ }
}
```

### Unary Operation

```json
{
  "kind": "unary",
  "op": "-",  // -, !
  "operand": { /* Expr */ }
}
```

### Function Call

```json
{
  "kind": "call",
  "callee": { /* Expr */ },
  "typeArgs": [{ /* TypeExpr */ }],  // optional
  "args": [{ /* Expr */ }]
}
```

### If Expression

```json
{
  "kind": "if",
  "condition": { /* Expr */ },
  "thenBranch": { /* BlockExpr */ },
  "elseBranch": { /* BlockExpr */ }  // optional
}
```

### Match Expression

```json
{
  "kind": "match",
  "scrutinee": { /* Expr */ },
  "arms": [
    {
      "pattern": { /* Pattern */ },
      "guard": { /* Expr */ },  // optional
      "body": { /* Expr */ }
    }
  ]
}
```

### Block Expression

```json
{
  "kind": "block",
  "statements": [{ /* Statement */ }],
  "expr": { /* Expr */ }  // optional, the block's value
}
```

### Array Expression

```json
{
  "kind": "array",
  "elements": [{ /* Expr */ }]
}
```

### Tuple Expression

```json
{
  "kind": "tuple",
  "elements": [{ /* Expr */ }]
}
```

### Record Expression

```json
{
  "kind": "record",
  "fields": [
    { "name": "fieldName", "value": { /* Expr */ } }
  ]
}
```

### Lambda Expression

```json
{
  "kind": "lambda",
  "params": [
    { "name": "paramName", "type": { /* TypeExpr */ } }  // type optional
  ],
  "body": { /* Expr */ }
}
```

### Field Access

```json
{
  "kind": "field",
  "object": { /* Expr */ },
  "field": "fieldName"
}
```

### Index Access

```json
{
  "kind": "index",
  "object": { /* Expr */ },
  "index": { /* Expr */ }
}
```

### Range Expression

```json
{
  "kind": "range",
  "start": { /* Expr */ },
  "end": { /* Expr */ },
  "inclusive": false
}
```

### Error Propagation

```json
{
  "kind": "propagate",
  "expr": { /* Expr */ }
}
```

This is the `?` operator for Result/Option propagation.

## Statements

### Let Statement

```json
{
  "kind": "let",
  "pattern": { /* Pattern */ },
  "type": { /* TypeExpr */ },  // optional
  "mutable": false,
  "value": { /* Expr */ }
}
```

### Assignment Statement

```json
{
  "kind": "assign",
  "target": { /* Expr */ },
  "value": { /* Expr */ }
}
```

### Expression Statement

```json
{
  "kind": "expr",
  "expr": { /* Expr */ }
}
```

### For Loop

```json
{
  "kind": "for",
  "pattern": { /* Pattern */ },
  "iterable": { /* Expr */ },
  "body": { /* BlockExpr */ }
}
```

### While Loop

```json
{
  "kind": "while",
  "condition": { /* Expr */ },
  "body": { /* BlockExpr */ }
}
```

### Loop (Infinite)

```json
{
  "kind": "loop",
  "body": { /* BlockExpr */ }
}
```

### Return Statement

```json
{
  "kind": "return",
  "value": { /* Expr */ }  // optional
}
```

### Break Statement

```json
{
  "kind": "break"
}
```

### Continue Statement

```json
{
  "kind": "continue"
}
```

### Assert Statement

```json
{
  "kind": "assert",
  "condition": { /* Expr */ },
  "message": "optional error message"  // optional
}
```

## Types

### Named Type

```json
{
  "kind": "named",
  "name": "Int"
}
```

Built-in types: `Int`, `Float`, `Bool`, `Str`, `Unit`, `unknown`, `never`

### Array Type

```json
{
  "kind": "array",
  "element": { /* TypeExpr */ }
}
```

### Tuple Type

```json
{
  "kind": "tuple",
  "elements": [{ /* TypeExpr */ }]
}
```

### Function Type

```json
{
  "kind": "function",
  "params": [{ /* TypeExpr */ }],
  "returnType": { /* TypeExpr */ }
}
```

### Refined Type

```json
{
  "kind": "refined",
  "base": { /* TypeExpr */ },
  "variable": "x",  // optional, defaults to base type name
  "predicate": { /* Expr */ }
}
```

Example: `Int{x > 0}`:
```json
{
  "kind": "refined",
  "base": { "kind": "named", "name": "Int" },
  "variable": "x",
  "predicate": { "source": "x > 0" }
}
```

### Effect Type

```json
{
  "kind": "effect",
  "effect": "IO",  // IO, Err, Async, Mut
  "inner": { /* TypeExpr */ },
  "errorType": { /* TypeExpr */ }  // only for Err
}
```

Example: `IO[()]`:
```json
{
  "kind": "effect",
  "effect": "IO",
  "inner": { "kind": "tuple", "elements": [] }
}
```

Example: `Err[MyError, Int]`:
```json
{
  "kind": "effect",
  "effect": "Err",
  "errorType": { "kind": "named", "name": "MyError" },
  "inner": { "kind": "named", "name": "Int" }
}
```

### Record Type (Anonymous)

```json
{
  "kind": "recordType",
  "fields": [
    { "name": "fieldName", "type": { /* TypeExpr */ } }
  ]
}
```

### Type Application (Generics)

```json
{
  "kind": "named",
  "name": "Option",
  "typeArgs": [{ "kind": "named", "name": "Int" }]
}
```

## Patterns

### Wildcard Pattern

```json
{
  "kind": "wildcard"
}
```

### Identifier Pattern

```json
{
  "kind": "ident",
  "name": "x"
}
```

### Literal Pattern

```json
{
  "kind": "literal",
  "value": { /* LiteralValue */ }
}
```

### Tuple Pattern

```json
{
  "kind": "tuple",
  "elements": [{ /* Pattern */ }]
}
```

### Record Pattern

```json
{
  "kind": "record",
  "fields": [
    { "name": "fieldName", "pattern": { /* Pattern */ } }
  ]
}
```

### Variant Pattern

```json
{
  "kind": "variant",
  "name": "Some",
  "fields": [{ /* Pattern */ }]
}
```

## Literals

### Integer Literal

```json
{ "kind": "int", "value": "42" }
```

Note: Integer values are strings to support BigInt.

### Float Literal

```json
{ "kind": "float", "value": 3.14 }
```

### String Literal

```json
{ "kind": "string", "value": "hello" }
```

### Boolean Literal

```json
{ "kind": "bool", "value": true }
```

### Unit Literal

```json
{ "kind": "unit" }
```

## Source Fragments

Any node can be replaced with a source string:

```json
{ "source": "x + 1" }
```

This is parsed and converted to the appropriate AST node. Useful for:
- Complex expressions easier to write as text
- Mixing generated structure with hand-written logic
- Predicates in refinement types

Works for: expressions, types, patterns, statements, declarations.

## Node Identity

In compiler output, every node has a stable `id` field:

```json
{
  "id": "node_001",
  "kind": "ident",
  "name": "x"
}
```

- IDs are **stable within a session**
- IDs are **deterministic** (same input produces same IDs)
- Diagnostics reference nodes by `primary_node_id`
- Repairs reference nodes by ID in PatchOps

When submitting AST, IDs are optional. The compiler assigns them to nodes without IDs.

## Spans (Optional)

Source locations are included in compiler output but optional on input:

```json
{
  "kind": "ident",
  "name": "x",
  "span": {
    "file": "main.clank",
    "start": { "line": 1, "column": 5, "offset": 4 },
    "end": { "line": 1, "column": 6, "offset": 5 }
  }
}
```
