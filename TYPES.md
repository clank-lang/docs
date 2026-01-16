# Type System Reference

Complete reference for Clank's type system.

## Contents

- [Primitive Types](#primitive-types)
- [Compound Types](#compound-types)
- [Named Types](#named-types)
- [Refinement Types](#refinement-types)
- [Effect Types](#effect-types)
- [Linear Types](#linear-types)
- [Type Parameters](#type-parameters)

## Primitive Types

| Type | Aliases | Description | JS Representation |
|------|---------|-------------|-------------------|
| `Bool` | - | Boolean | `boolean` |
| `Int` | `ℤ` | Arbitrary precision integer | `BigInt` |
| `Int32` | `ℤ32` | 32-bit signed integer | `number` |
| `Int64` | `ℤ64` | 64-bit signed integer | `BigInt` |
| `Nat` | `ℕ` | Natural number (≥0) | `BigInt` with refinement |
| `Float` | `ℝ` | IEEE 754 double | `number` |
| `Str` | - | UTF-8 string | `string` |
| `Unit` | `()` | Unit type | `undefined` |
| `unknown` | - | Unknown type (must validate) | `unknown` |
| `never` | - | Bottom type (no values) | `never` |

### AST JSON

```json
{ "kind": "named", "name": "Int" }
{ "kind": "named", "name": "Bool" }
{ "kind": "named", "name": "Str" }
{ "kind": "tuple", "elements": [] }  // Unit
```

## Compound Types

### Arrays

```
[T]              // Array of T, unknown length
Vec[T, n]        // Array with compile-time known length (post-MVP)
```

AST JSON:
```json
{
  "kind": "array",
  "element": { "kind": "named", "name": "Int" }
}
```

### Tuples

```
(T, U)           // Pair
(T, U, V)        // Triple
()               // Unit (empty tuple)
```

AST JSON:
```json
{
  "kind": "tuple",
  "elements": [
    { "kind": "named", "name": "Int" },
    { "kind": "named", "name": "Str" }
  ]
}
```

### Records (Anonymous)

```
{name: Str, age: Int}           // Anonymous record
{name: Str, age: Int, ...}      // Open record (extensible)
```

AST JSON:
```json
{
  "kind": "recordType",
  "fields": [
    { "name": "name", "type": { "kind": "named", "name": "Str" } },
    { "name": "age", "type": { "kind": "named", "name": "Int" } }
  ]
}
```

### Functions

```
(T) -> U                       // Function from T to U
(T, U) -> V                    // Multiple arguments
() -> T                        // No arguments
```

AST JSON:
```json
{
  "kind": "function",
  "params": [
    { "kind": "named", "name": "Int" },
    { "kind": "named", "name": "Int" }
  ],
  "returnType": { "kind": "named", "name": "Int" }
}
```

### Option/Result Sugar

```
T?                            // Sugar for Option[T]
T!E                           // Sugar for Result[T, E] (post-MVP)
```

## Named Types

### Type Aliases

```clank
type UserId = Int{x > 0}
type Name = Str{len(s) > 0 && len(s) <= 100}
type Pair[T] = (T, T)
```

AST JSON:
```json
{
  "kind": "typeAlias",
  "name": "UserId",
  "type": {
    "kind": "refined",
    "base": { "kind": "named", "name": "Int" },
    "predicate": { "source": "x > 0" }
  }
}
```

### Record Types (Product Types)

```clank
rec User {
  id: UserId,
  name: Name,
  email: Str,
  active: Bool
}
```

AST JSON:
```json
{
  "kind": "rec",
  "name": "User",
  "fields": [
    { "name": "id", "type": { "kind": "named", "name": "UserId" } },
    { "name": "name", "type": { "kind": "named", "name": "Name" } },
    { "name": "email", "type": { "kind": "named", "name": "Str" } },
    { "name": "active", "type": { "kind": "named", "name": "Bool" } }
  ]
}
```

### Sum Types (Tagged Unions)

```clank
sum Option[T] {
  Some(T),
  None
}

sum Result[T, E] {
  Ok(T),
  Err(E)
}

sum Json {
  Null,
  Bool(Bool),
  Number(Float),
  String(Str),
  Array([Json]),
  Object(Map[Str, Json])
}
```

AST JSON:
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

## Refinement Types

Refinement types attach predicates to base types.

### Syntax

```
T{predicate}              // Type T refined by predicate
T{var | predicate}        // Explicit variable name
```

### Examples

```clank
Int{x > 0}                     // Positive integer
Int{x >= 0 && x <= 100}        // Integer 0-100
[T]{len(arr) > 0}              // Non-empty array
Str{len(s) > 0}                // Non-empty string
```

### AST JSON

```json
{
  "kind": "refined",
  "base": { "kind": "named", "name": "Int" },
  "variable": "x",
  "predicate": { "source": "x > 0" }
}
```

For array refinements:
```json
{
  "kind": "refined",
  "base": {
    "kind": "array",
    "element": { "kind": "named", "name": "T" }
  },
  "variable": "arr",
  "predicate": { "source": "len(arr) > 0" }
}
```

### Predicates

Predicates support a subset of expressions:

| Category | Operators |
|----------|-----------|
| Arithmetic | `+`, `-`, `*`, `/`, `%` |
| Comparison | `==`, `!=`, `<`, `>`, `<=`, `>=` |
| Logical | `&&`, `\|\|`, `!` |
| Functions | `len(x)`, `is_empty(x)` |
| Array access | `arr[i]` (with bounds) |

### Proof Obligations

When refined types are used, the compiler generates proof obligations:

```clank
fn div(n: Int, d: Int{d != 0}) -> Int { n / d }

fn example() -> Int {
  let y = get_input()   // Returns Int (unrefined)
  div(10, y)            // OBLIGATION: prove y != 0
}
```

The solver attempts to discharge obligations automatically. When it cannot, it returns hints or counterexamples.

### Pre/Post Conditions

```clank
fn binary_search[T: Ord](arr: [T], target: T) -> Option[Nat{i < len(arr)}]
  pre is_sorted(arr)
  post match result {
    Some(i) -> arr[i] == target,
    None -> true
  }
{
  // implementation
}
```

AST JSON:
```json
{
  "kind": "fn",
  "name": "binary_search",
  "precondition": { "source": "is_sorted(arr)" },
  "postcondition": { "source": "match result { Some(i) -> arr[i] == target, None -> true }" },
  ...
}
```

## Effect Types

Effects track side effects in the type system.

| Effect | Meaning | Example |
|--------|---------|---------|
| `Pure[T]` | No side effects | Pure computation |
| `IO[T]` | Performs I/O | `print`, `read_file` |
| `Err[E, T]` | Can fail with E | Fallible operations |
| `Async[T]` | Asynchronous | Returns Promise |
| `Mut[T]` | Mutates state | Mutable operations |

### Combined Effects

```
IO + Err[E, T]          // Multiple effects
```

### AST JSON

```json
{
  "kind": "effect",
  "effect": "IO",
  "inner": { "kind": "tuple", "elements": [] }
}
```

For `Err[MyError, Int]`:
```json
{
  "kind": "effect",
  "effect": "Err",
  "errorType": { "kind": "named", "name": "MyError" },
  "inner": { "kind": "named", "name": "Int" }
}
```

### Effect Subtyping

```
Pure[T] <: IO[T]         // Pure can be used where IO expected
Pure[T] <: Err[E, T]     // Pure is subtype of any effect
IO[T] <: IO + Err[E, T]  // Adding effects is fine
```

### Effect Inference

Effects are inferred within function bodies. If a function calls `print()`, it requires `IO`.

## Linear Types

Linear types ensure resources are used exactly once.

| Type | Meaning |
|------|---------|
| `Linear[T]` | Must be used exactly once |
| `Affine[T]` | Must be used at most once (can drop) |

### Example

```clank
fn open_file(path: Str) -> IO[Linear[FileHandle]]

fn read_all(h: Linear[FileHandle]) -> IO[(Str, Linear[FileHandle])] {
  // Returns handle so it can be used again or closed
}

fn close_file(h: Linear[FileHandle]) -> IO[()] {
  // Consumes the handle
}

fn example() -> IO[()] {
  let h = open_file("test.txt")
  let (contents, h) = read_all(h)
  close_file(h)                     // h consumed
  // h cannot be used here
}
```

### Enforcement

- Statically checked by the Clank compiler
- Erased at runtime (no JS enforcement)
- Double-use or non-consumption produces errors

## Type Parameters

### Basic Generics

```clank
fn identity[T](x: T) -> T { x }

fn map[T, U](arr: [T], f: (T) -> U) -> [U] {
  // ...
}
```

AST JSON:
```json
{
  "kind": "fn",
  "name": "identity",
  "typeParams": [{ "name": "T" }],
  "params": [{ "name": "x", "type": { "kind": "named", "name": "T" } }],
  "returnType": { "kind": "named", "name": "T" },
  ...
}
```

### Constrained Generics

```clank
fn sort[T: Ord](arr: [T]) -> [T] { ... }

fn show[T: Show](x: T) -> Str { ... }

fn compare[T: Eq + Ord](a: T, b: T) -> Ordering { ... }
```

AST JSON:
```json
{
  "kind": "fn",
  "name": "sort",
  "typeParams": [{ "name": "T", "constraint": { "kind": "named", "name": "Ord" } }],
  ...
}
```

### Type Application

```clank
Option[Int]
Result[User, ApiError]
Map[Str, Json]
```

AST JSON:
```json
{
  "kind": "named",
  "name": "Option",
  "typeArgs": [{ "kind": "named", "name": "Int" }]
}
```

## Standard Types

### Option

```clank
sum Option[T] {
  Some(T),
  None
}
```

### Result

```clank
sum Result[T, E] {
  Ok(T),
  Err(E)
}
```

### Ordering

```clank
sum Ordering {
  Less,
  Equal,
  Greater
}
```
