# Code Examples

Annotated Clank examples organized by complexity and category. Each example includes what it demonstrates, common mistakes, and TypeScript comparison.

## Contents

- [Basics](#basics)
- [Collections](#collections)
- [Refinement Types](#refinement-types)
- [Pattern Matching](#pattern-matching)
- [Error Handling](#error-handling)
- [Real-World Snippets](#real-world-snippets)

---

## Basics

### Hello World

```clank
ƒ main() → IO[()] {
  println("Hello, World!")
}
```

**What it demonstrates:**
- Basic function definition with `ƒ`
- The `IO` effect for side effects
- Unit return type `()`

**TypeScript equivalent:**
```typescript
function main(): void {
  console.log("Hello, World!");
}
```

**Common mistakes:**
- Forgetting the `IO` effect — `println` requires it
- Using `→ ()` instead of `→ IO[()]` — missing the effect wrapper

---

### Simple Function

```clank
ƒ add(a: ℤ, b: ℤ) → ℤ {
  a + b
}
```

**What it demonstrates:**
- Function parameters with type annotations
- Implicit return (last expression is the return value)
- Integer type `ℤ`

**TypeScript equivalent:**
```typescript
function add(a: bigint, b: bigint): bigint {
  return a + b;
}
```

**Common mistakes:**
- Adding explicit `return` — not wrong, but unnecessary
- Using `Int` for small numbers when `ℤ32` would be more efficient

---

### Variables and Mutation

```clank
ƒ counter() → ℤ {
  let x = 0           // Immutable binding
  let mut y = 0       // Mutable binding

  y = y + 1           // OK: y is mutable
  // x = x + 1        // ERROR: x is immutable

  y
}
```

**What it demonstrates:**
- Immutable bindings by default (`let`)
- Explicit mutability with `let mut`
- Assignment syntax

**TypeScript equivalent:**
```typescript
function counter(): bigint {
  const x = 0n;       // Immutable
  let y = 0n;         // Mutable

  y = y + 1n;

  return y;
}
```

**Common mistakes:**
- Forgetting `mut` when you need to reassign
- Using `mut` everywhere — prefer immutability

---

### Generic Functions

```clank
ƒ identity[T](x: T) → T {
  x
}

ƒ swap[T, U](pair: (T, U)) → (U, T) {
  let (a, b) = pair
  (b, a)
}
```

**What it demonstrates:**
- Type parameters with `[T]` syntax
- Multiple type parameters
- Tuple destructuring in `let`

**TypeScript equivalent:**
```typescript
function identity<T>(x: T): T {
  return x;
}

function swap<T, U>(pair: [T, U]): [U, T] {
  const [a, b] = pair;
  return [b, a];
}
```

---

## Collections

### Working with Arrays

```clank
ƒ sum(numbers: [ℤ]) → ℤ {
  let mut total = 0
  for n ∈ numbers {
    total = total + n
  }
  total
}

// Using fold
ƒ sum_fold(numbers: [ℤ]) → ℤ {
  numbers |> fold(0, λ(acc, n) → acc + n)
}
```

**What it demonstrates:**
- Array type syntax `[ℤ]`
- For loop with `∈` (or `in`)
- Pipe operator `|>` for chaining
- Lambda syntax `λ(args) → body`

**TypeScript equivalent:**
```typescript
function sum(numbers: bigint[]): bigint {
  let total = 0n;
  for (const n of numbers) {
    total = total + n;
  }
  return total;
}

// Using reduce
function sumFold(numbers: bigint[]): bigint {
  return numbers.reduce((acc, n) => acc + n, 0n);
}
```

---

### Map, Filter, Reduce

```clank
ƒ process_numbers(nums: [ℤ]) → ℤ {
  nums
    |> map(λx → x × 2)           // Double each
    |> filter(λx → x > 10)       // Keep > 10
    |> fold(0, λ(a, b) → a + b)  // Sum
}
```

**What it demonstrates:**
- Chained transformations with pipe
- Concise lambda syntax
- Functional style processing

**TypeScript equivalent:**
```typescript
function processNumbers(nums: bigint[]): bigint {
  return nums
    .map(x => x * 2n)
    .filter(x => x > 10n)
    .reduce((a, b) => a + b, 0n);
}
```

---

### Working with Maps

```clank
ƒ word_count(words: [𝕊]) → Map[𝕊, ℤ] {
  let mut counts: Map[𝕊, ℤ] = Map::new()

  for word ∈ words {
    let current = counts.get(word) ?? 0
    counts = counts.insert(word, current + 1)
  }

  counts
}
```

**What it demonstrates:**
- Generic Map type
- Null coalescing `??`
- Immutable map operations (returns new map)

**TypeScript equivalent:**
```typescript
function wordCount(words: string[]): Map<string, bigint> {
  const counts = new Map<string, bigint>();

  for (const word of words) {
    const current = counts.get(word) ?? 0n;
    counts.set(word, current + 1n);
  }

  return counts;
}
```

---

## Refinement Types

### Basic Refinements

```clank
// Positive integer
τ Positive = ℤ{self > 0}

// Non-empty string
τ NonEmptyString = 𝕊{len(self) > 0}

// Percentage (0-100)
τ Percentage = ℝ{self ≥ 0.0 ∧ self ≤ 100.0}

// Using refined types
ƒ safe_divide(n: ℤ, d: Positive) → ℤ {
  n ÷ d  // Cannot divide by zero — d is guaranteed > 0
}
```

**What it demonstrates:**
- Type aliases with `τ`
- Refinement predicates with `{predicate}`
- Using `self` to refer to the value
- Compile-time safety guarantees

**TypeScript equivalent:**
```typescript
// TypeScript cannot express refinement types
// You would use runtime validation:
function safeDivide(n: bigint, d: bigint): bigint {
  if (d <= 0n) throw new Error("d must be positive");
  return n / d;
}
```

**Common mistakes:**
- Forgetting that refinements are checked at compile time
- Using runtime values without proving the refinement

---

### Refinements with Arrays

```clank
// Non-empty array
τ NonEmpty[T] = [T]{len(self) > 0}

// Safe head function — cannot fail!
ƒ head[T](arr: NonEmpty[T]) → T {
  arr[0]
}

// Bounded array
τ SmallArray[T] = [T]{len(self) ≤ 100}

// Array with valid indices
ƒ get_at[T](arr: [T], i: ℕ{i < len(arr)}) → T {
  arr[i]  // Index is guaranteed in bounds
}
```

**What it demonstrates:**
- Generic refinement types
- Dependent refinements (predicate references other values)
- Safe indexing without runtime checks

---

### Progressively Complex Constraints

```clank
// Simple constraint
τ Age = ℕ{self ≤ 150}

// Multiple constraints
τ ValidEmail = 𝕊{
  len(self) > 0 ∧
  contains(self, "@") ∧
  len(self) ≤ 254
}

// Dependent constraint
rec Range {
  min: ℤ,
  max: ℤ{self ≥ min}  // max must be >= min
}

// Constraint referencing function parameter
ƒ clamp(value: ℤ, range: Range) → ℤ{self ≥ range.min ∧ self ≤ range.max} {
  if value < range.min {
    range.min
  } else if value > range.max {
    range.max
  } else {
    value
  }
}
```

---

## Pattern Matching

### Basic Patterns

```clank
ƒ describe_number(n: ℤ) → 𝕊 {
  match n {
    0 → "zero",
    1 → "one",
    2 → "two",
    _ → "many"
  }
}
```

**What it demonstrates:**
- Match expression syntax
- Literal patterns
- Wildcard pattern `_`
- Match is an expression (returns value)

**TypeScript equivalent:**
```typescript
function describeNumber(n: bigint): string {
  switch (n) {
    case 0n: return "zero";
    case 1n: return "one";
    case 2n: return "two";
    default: return "many";
  }
}
```

---

### Pattern Guards

```clank
ƒ classify(n: ℤ) → 𝕊 {
  match n {
    0 → "zero",
    n if n < 0 → "negative",
    n if n % 2 == 0 → "positive even",
    _ → "positive odd"
  }
}
```

**What it demonstrates:**
- Guards with `if` after pattern
- Binding in patterns (`n` captures the value)

---

### Destructuring Sum Types

```clank
sum Option[T] {
  Some(T),
  None
}

ƒ unwrap_or[T](opt: Option[T], default: T) → T {
  match opt {
    Some(value) → value,
    None → default
  }
}

ƒ map_option[T, U](opt: Option[T], f: (T) → U) → Option[U] {
  match opt {
    Some(value) → Some(f(value)),
    None → None
  }
}
```

**What it demonstrates:**
- Sum type definition
- Destructuring in match arms
- Preserving type structure in transformations

---

### Nested Patterns

```clank
rec Point { x: ℤ, y: ℤ }

sum Shape {
  Circle(Point, ℤ),        // center, radius
  Rectangle(Point, Point)  // top-left, bottom-right
}

ƒ describe(shape: Shape) → 𝕊 {
  match shape {
    Circle({ x: 0, y: 0 }, r) → "Circle at origin with radius " ++ str(r),
    Circle(center, r) → "Circle at " ++ str(center) ++ " with radius " ++ str(r),
    Rectangle({ x: 0, y: 0 }, _) → "Rectangle from origin",
    Rectangle(tl, br) → "Rectangle from " ++ str(tl) ++ " to " ++ str(br)
  }
}
```

---

### Exhaustiveness Checking

```clank
sum TrafficLight {
  Red,
  Yellow,
  Green
}

// Compiler error: non-exhaustive match
ƒ bad_action(light: TrafficLight) → 𝕊 {
  match light {
    Red → "stop",
    Green → "go"
    // Missing: Yellow
  }
}

// Correct: all cases covered
ƒ action(light: TrafficLight) → 𝕊 {
  match light {
    Red → "stop",
    Yellow → "caution",
    Green → "go"
  }
}
```

---

## Error Handling

### Option Type

```clank
ƒ find_first[T](arr: [T], pred: (T) → 𝔹) → Option[T] {
  for item ∈ arr {
    if pred(item) {
      return Some(item)
    }
  }
  None
}

// Using the result
ƒ example() → 𝕊 {
  let numbers = [1, 2, 3, 4, 5]
  let found = find_first(numbers, λx → x > 3)

  match found {
    Some(n) → "Found: " ++ str(n),
    None → "Not found"
  }
}
```

---

### Result Type and Propagation

```clank
sum Result[T, E] {
  Ok(T),
  Err(E)
}

rec ParseError { message: 𝕊, position: ℕ }

ƒ parse_int(s: 𝕊) → Result[ℤ, ParseError] {
  // ... implementation
}

// Using ? for propagation
ƒ parse_and_double(s: 𝕊) → Result[ℤ, ParseError] {
  let n = parse_int(s)?   // Early return on Err
  Ok(n × 2)
}

// Chaining multiple fallible operations
ƒ parse_sum(a: 𝕊, b: 𝕊) → Result[ℤ, ParseError] {
  let x = parse_int(a)?
  let y = parse_int(b)?
  Ok(x + y)
}
```

**What it demonstrates:**
- Result type for explicit error handling
- The `?` operator for early return on error
- Clean composition of fallible operations

**TypeScript equivalent:**
```typescript
// TypeScript would use exceptions or explicit checks
function parseAndDouble(s: string): bigint {
  const n = parseInt(s);  // Throws or returns NaN
  if (isNaN(n)) throw new Error("Parse error");
  return BigInt(n) * 2n;
}
```

---

### Combining IO and Errors

```clank
rec IoError { kind: 𝕊, message: 𝕊 }
rec Config { host: 𝕊, port: ℕ }

ƒ read_file(path: 𝕊) → IO[Result[𝕊, IoError]] {
  // ... implementation
}

ƒ parse_config(content: 𝕊) → Result[Config, ParseError] {
  // ... implementation
}

ƒ load_config(path: 𝕊) → IO[Result[Config, 𝕊]] {
  let content = read_file(path)?

  match content {
    Ok(text) → {
      match parse_config(text) {
        Ok(cfg) → Ok(cfg),
        Err(e) → Err("Parse error: " ++ e.message)
      }
    },
    Err(e) → Err("IO error: " ++ e.message)
  }
}
```

---

## Real-World Snippets

### FizzBuzz

```clank
ƒ fizzbuzz(n: ℕ{n > 0}) → IO[()] {
  for i ∈ 1..=n {
    let result = match (i % 3, i % 5) {
      (0, 0) → "FizzBuzz",
      (0, _) → "Fizz",
      (_, 0) → "Buzz",
      _ → str(i)
    }
    println(result)
  }
}
```

---

### Binary Search

```clank
ƒ binary_search[T: Ord](arr: [T], target: T) → Option[ℕ]
  pre is_sorted(arr)
{
  let mut low = 0
  let mut high = len(arr)

  while low < high {
    let mid = low + (high - low) ÷ 2
    match compare(arr[mid], target) {
      Less → low = mid + 1,
      Greater → high = mid,
      Equal → return Some(mid)
    }
  }

  None
}
```

---

### Simple HTTP Response

```clank
rec HttpResponse {
  status: ℕ{self ≥ 100 ∧ self < 600},
  headers: Map[𝕊, 𝕊],
  body: 𝕊
}

ƒ ok(body: 𝕊) → HttpResponse {
  HttpResponse {
    status: 200,
    headers: Map::from([("Content-Type", "text/plain")]),
    body
  }
}

ƒ not_found() → HttpResponse {
  HttpResponse {
    status: 404,
    headers: Map::new(),
    body: "Not Found"
  }
}

ƒ json_response[T: Serialize](data: T) → HttpResponse {
  HttpResponse {
    status: 200,
    headers: Map::from([("Content-Type", "application/json")]),
    body: to_json(data)
  }
}
```

---

### Simple State Machine

```clank
sum ConnectionState {
  Disconnected,
  Connecting,
  Connected(Socket),
  Disconnecting
}

sum ConnectionEvent {
  Connect,
  Connected(Socket),
  Disconnect,
  Disconnected
}

ƒ transition(state: ConnectionState, event: ConnectionEvent) → ConnectionState {
  match (state, event) {
    (Disconnected, Connect) → Connecting,
    (Connecting, Connected(socket)) → Connected(socket),
    (Connected(_), Disconnect) → Disconnecting,
    (Disconnecting, Disconnected) → Disconnected,
    (s, _) → s  // Ignore invalid transitions
  }
}
```

---

### Validated User Input

```clank
τ Username = 𝕊{
  len(self) ≥ 3 ∧
  len(self) ≤ 20 ∧
  all(chars(self), is_alphanumeric)
}

τ Email = 𝕊{
  len(self) > 0 ∧
  contains(self, "@") ∧
  len(self) ≤ 254
}

τ Password = 𝕊{
  len(self) ≥ 8 ∧
  any(chars(self), is_uppercase) ∧
  any(chars(self), is_digit)
}

rec UserRegistration {
  username: Username,
  email: Email,
  password: Password
}

ƒ register(reg: UserRegistration) → IO[Result[User, RegistrationError]] {
  // At this point, all validation is guaranteed by the type system
  // No need for runtime validation of the constraints
  create_user(reg.username, reg.email, hash_password(reg.password))
}
```

---

### Tree Traversal

```clank
sum Tree[T] {
  Leaf(T),
  Node(Tree[T], Tree[T])
}

ƒ map_tree[T, U](tree: Tree[T], f: (T) → U) → Tree[U] {
  match tree {
    Leaf(value) → Leaf(f(value)),
    Node(left, right) → Node(
      map_tree(left, f),
      map_tree(right, f)
    )
  }
}

ƒ fold_tree[T, U](tree: Tree[T], leaf_fn: (T) → U, node_fn: (U, U) → U) → U {
  match tree {
    Leaf(value) → leaf_fn(value),
    Node(left, right) → node_fn(
      fold_tree(left, leaf_fn, node_fn),
      fold_tree(right, leaf_fn, node_fn)
    )
  }
}

// Sum all values in a tree of integers
ƒ sum_tree(tree: Tree[ℤ]) → ℤ {
  fold_tree(tree, λx → x, λ(a, b) → a + b)
}
```

---

### JSON-like Structure

```clank
sum Json {
  Null,
  Bool(𝔹),
  Number(ℝ),
  String(𝕊),
  Array([Json]),
  Object(Map[𝕊, Json])
}

ƒ get_string(json: Json, key: 𝕊) → Option[𝕊] {
  match json {
    Object(obj) → {
      match obj.get(key) {
        Some(String(s)) → Some(s),
        _ → None
      }
    },
    _ → None
  }
}

ƒ pretty_print(json: Json, indent: ℕ) → 𝕊 {
  let spaces = " " × indent
  match json {
    Null → "null",
    Bool(b) → if b { "true" } else { "false" },
    Number(n) → str(n),
    String(s) → "\"" ++ escape(s) ++ "\"",
    Array(items) → {
      let inner = items
        |> map(λj → pretty_print(j, indent + 2))
        |> join(",\n" ++ spaces ++ "  ")
      "[\n" ++ spaces ++ "  " ++ inner ++ "\n" ++ spaces ++ "]"
    },
    Object(obj) → {
      let inner = obj
        |> entries
        |> map(λ(k, v) → "\"" ++ k ++ "\": " ++ pretty_print(v, indent + 2))
        |> join(",\n" ++ spaces ++ "  ")
      "{\n" ++ spaces ++ "  " ++ inner ++ "\n" ++ spaces ++ "}"
    }
  }
}
```

---

## Common Patterns Summary

| Pattern | Use When | Example |
|---------|----------|---------|
| Refinement types | Encoding invariants | `ℕ{self > 0}` |
| Sum types | Representing variants | `Option`, `Result` |
| Pattern matching | Branching on structure | `match x { ... }` |
| Pipe operator | Chaining transformations | `x \|> f \|> g` |
| `?` propagation | Early return on error | `parse(s)?` |
| Lambda | Inline functions | `λx → x + 1` |
| Pre/post conditions | Contract checking | `pre len(arr) > 0` |
