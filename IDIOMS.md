# Idiomatic Clank

Preferred patterns, anti-patterns, and best practices for writing Clank code.

## Contents

- [Type Design](#type-design)
- [Functions](#functions)
- [Error Handling](#error-handling)
- [Control Flow](#control-flow)
- [Effects](#effects)
- [Naming Conventions](#naming-conventions)
- [Module Organization](#module-organization)
- [Performance](#performance)
- [Working with the Compiler](#working-with-the-compiler)

---

## Type Design

### Prefer Refinement Types over Runtime Checks

**Do this:**
```clank
τ PositiveInt = ℤ{self > 0}

ƒ sqrt(n: PositiveInt) → ℝ {
  // n is guaranteed positive, no runtime check needed
  builtin_sqrt(n)
}
```

**Don't do this:**
```clank
ƒ sqrt(n: ℤ) → ℝ {
  if n <= 0 {
    panic("n must be positive")  // Runtime failure
  }
  builtin_sqrt(n)
}
```

**Why:** Refinement types catch errors at compile time. The caller must prove the constraint is satisfied, pushing validation to the boundary.

---

### Make Invalid States Unrepresentable

**Do this:**
```clank
sum LoadingState[T] {
  NotStarted,
  Loading,
  Loaded(T),
  Failed(Error)
}
```

**Don't do this:**
```clank
rec LoadingState[T] {
  is_loading: 𝔹,
  data: Option[T],
  error: Option[Error]
}
// What does is_loading=true, data=Some(x), error=Some(e) mean?
```

**Why:** Sum types enforce that only valid combinations exist. You can't have both data and error, or be loading with data already present.

---

### Use Newtypes for Domain Concepts

**Do this:**
```clank
τ UserId = ℤ{self > 0}
τ OrderId = ℤ{self > 0}

ƒ get_order(user: UserId, order: OrderId) → Option[Order] { ... }
```

**Don't do this:**
```clank
ƒ get_order(user: ℤ, order: ℤ) → Option[Order] { ... }

// Easy to swap arguments by mistake:
get_order(order_id, user_id)  // Compiles but wrong!
```

**Why:** Distinct types prevent accidental mixing of values that have the same underlying representation.

---

### Keep Refinements Simple

**Do this:**
```clank
τ Percentage = ℝ{self ≥ 0.0 ∧ self ≤ 100.0}
```

**Don't do this:**
```clank
τ ComplexConstraint = ℤ{
  self > 0 ∧
  is_prime(self) ∧
  digit_sum(self) % 3 == 0 ∧
  self < 1000000
}
```

**Why:** Complex predicates are harder for the solver to prove. If you need complex validation, consider a smart constructor:

```clank
ƒ validate_complex(n: ℤ) → Option[ComplexValue] {
  if satisfies_constraints(n) {
    Some(unsafe_mk_complex(n))
  } else {
    None
  }
}
```

---

## Functions

### Prefer Expression-Oriented Style

**Do this:**
```clank
ƒ classify(n: ℤ) → 𝕊 {
  if n > 0 { "positive" }
  else if n < 0 { "negative" }
  else { "zero" }
}
```

**Don't do this:**
```clank
ƒ classify(n: ℤ) → 𝕊 {
  let mut result = ""
  if n > 0 {
    result = "positive"
  } else if n < 0 {
    result = "negative"
  } else {
    result = "zero"
  }
  return result
}
```

**Why:** Expression-oriented code is more concise and clearly returns a value from each branch.

---

### Use the Pipe Operator for Transformations

**Do this:**
```clank
ƒ process(data: [𝕊]) → [ℤ] {
  data
    |> filter(λs → len(s) > 0)
    |> map(parse_int)
    |> filter(λopt → opt.is_some())
    |> map(λopt → opt.unwrap())
}
```

**Don't do this:**
```clank
ƒ process(data: [𝕊]) → [ℤ] {
  let step1 = filter(data, λs → len(s) > 0)
  let step2 = map(step1, parse_int)
  let step3 = filter(step2, λopt → opt.is_some())
  map(step3, λopt → opt.unwrap())
}
```

**Why:** Pipes read top-to-bottom in transformation order, reducing intermediate variable clutter.

---

### Keep Functions Small and Focused

**Do this:**
```clank
ƒ validate_user(user: User) → Result[ValidatedUser, ValidationError] {
  let name = validate_name(user.name)?
  let email = validate_email(user.email)?
  let age = validate_age(user.age)?
  Ok(ValidatedUser { name, email, age })
}

ƒ validate_name(name: 𝕊) → Result[ValidName, ValidationError] { ... }
ƒ validate_email(email: 𝕊) → Result[ValidEmail, ValidationError] { ... }
ƒ validate_age(age: ℤ) → Result[ValidAge, ValidationError] { ... }
```

**Don't do this:**
```clank
ƒ validate_user(user: User) → Result[ValidatedUser, ValidationError] {
  // 50 lines of validation logic all in one function
  if len(user.name) == 0 { return Err(...) }
  if len(user.name) > 100 { return Err(...) }
  // ... etc
}
```

**Why:** Small functions are easier to test, reuse, and understand. They also produce better error messages.

---

### Use Pre/Post Conditions for Complex Contracts

**Do this:**
```clank
ƒ binary_search[T: Ord](arr: [T], target: T) → Option[ℕ]
  pre is_sorted(arr)
  post match result {
    Some(i) → i < len(arr) ∧ arr[i] == target,
    None → ¬contains(arr, target)
  }
{
  // implementation
}
```

**Don't do this:**
```clank
ƒ binary_search[T: Ord](arr: [T], target: T) → Option[ℕ] {
  // Assumes array is sorted (undocumented)
  // No guarantees about return value
}
```

**Why:** Pre/post conditions document expectations and enable the compiler to verify correctness.

---

## Error Handling

### Use Result for Recoverable Errors

**Do this:**
```clank
ƒ parse_config(path: 𝕊) → IO[Result[Config, ConfigError]] {
  let content = read_file(path)?
  match content {
    Ok(text) → parse(text),
    Err(e) → Err(ConfigError::IoError(e))
  }
}
```

**Don't do this:**
```clank
ƒ parse_config(path: 𝕊) → IO[Config] {
  let content = read_file(path)
  match content {
    Ok(text) → parse(text).unwrap(),  // Panics on parse error!
    Err(_) → panic("Could not read file")
  }
}
```

**Why:** Result makes errors explicit and composable. Callers can decide how to handle failures.

---

### Prefer `?` over Explicit Matching

**Do this:**
```clank
ƒ process() → Result[Output, Error] {
  let a = step_one()?
  let b = step_two(a)?
  let c = step_three(b)?
  Ok(c)
}
```

**Don't do this:**
```clank
ƒ process() → Result[Output, Error] {
  match step_one() {
    Err(e) → Err(e),
    Ok(a) → match step_two(a) {
      Err(e) → Err(e),
      Ok(b) → match step_three(b) {
        Err(e) → Err(e),
        Ok(c) → Ok(c)
      }
    }
  }
}
```

**Why:** The `?` operator reduces boilerplate and flattens the happy path.

---

### Use Option for Absence, Result for Failure

**Do this:**
```clank
// Absence is normal — use Option
ƒ find(arr: [T], pred: (T) → 𝔹) → Option[T]

// Absence is an error — use Result
ƒ get_required_config(key: 𝕊) → Result[𝕊, ConfigError]
```

**Don't do this:**
```clank
// Using Option when absence is an error
ƒ get_required_config(key: 𝕊) → Option[𝕊]  // Caller doesn't know why None

// Using Result when absence is normal
ƒ find(arr: [T], pred: (T) → 𝔹) → Result[T, NotFoundError]  // Overly dramatic
```

---

### Provide Context in Errors

**Do this:**
```clank
sum ParseError {
  InvalidSyntax { line: ℕ, column: ℕ, message: 𝕊 },
  UnexpectedEof { expected: 𝕊 },
  TypeMismatch { expected: 𝕊, actual: 𝕊, location: Span }
}
```

**Don't do this:**
```clank
sum ParseError {
  Error(𝕊)  // Just a string, no structured info
}
```

**Why:** Structured errors enable programmatic handling and better user messages.

---

## Control Flow

### Use Match for Multiple Conditions

**Do this:**
```clank
ƒ http_status_text(code: ℕ) → 𝕊 {
  match code {
    200 → "OK",
    201 → "Created",
    400 → "Bad Request",
    404 → "Not Found",
    500 → "Internal Server Error",
    _ → "Unknown"
  }
}
```

**Don't do this:**
```clank
ƒ http_status_text(code: ℕ) → 𝕊 {
  if code == 200 { "OK" }
  else if code == 201 { "Created" }
  else if code == 400 { "Bad Request" }
  else if code == 404 { "Not Found" }
  else if code == 500 { "Internal Server Error" }
  else { "Unknown" }
}
```

**Why:** Match is more readable for value-based branching and enables exhaustiveness checking.

---

### Prefer Iteration over Recursion for Simple Loops

**Do this:**
```clank
ƒ sum(numbers: [ℤ]) → ℤ {
  let mut total = 0
  for n ∈ numbers {
    total = total + n
  }
  total
}
```

**Don't do this (for simple cases):**
```clank
ƒ sum(numbers: [ℤ]) → ℤ {
  match numbers {
    [] → 0,
    [head, ...tail] → head + sum(tail)  // Stack overflow risk
  }
}
```

**Why:** Iteration is clearer for linear traversal and avoids stack overflow. Use recursion for tree structures.

---

### Avoid Early Returns in Complex Functions

**Do this:**
```clank
ƒ validate(input: Input) → Result[Valid, Error] {
  let a = validate_a(input.a)?
  let b = validate_b(input.b)?
  let c = validate_c(input.c)?
  Ok(Valid { a, b, c })
}
```

**Don't do this:**
```clank
ƒ validate(input: Input) → Result[Valid, Error] {
  if ¬valid_a(input.a) { return Err(ErrorA) }
  if ¬valid_b(input.b) { return Err(ErrorB) }
  if ¬valid_c(input.c) { return Err(ErrorC) }
  Ok(Valid { a: input.a, b: input.b, c: input.c })
}
```

**Why:** Early returns can make control flow hard to follow. The `?` operator provides structured early return.

---

## Effects

### Minimize Effect Scope

**Do this:**
```clank
ƒ process_data(data: [Item]) → [Result] {
  // Pure transformation
  data |> map(transform) |> filter(valid)
}

ƒ main() → IO[()] {
  let data = load_data()  // IO here
  let results = process_data(data)  // Pure
  save_results(results)  // IO here
}
```

**Don't do this:**
```clank
ƒ process_data(data: [Item]) → IO[[Result]] {
  // Unnecessary IO effect pollutes the function
  for item ∈ data {
    println("Processing: " ++ str(item))  // Debug logging adds IO
  }
  data |> map(transform) |> filter(valid)
}
```

**Why:** Pure functions are easier to test and compose. Push effects to the edges.

---

### Be Explicit About Effects

**Do this:**
```clank
ƒ fetch_user(id: UserId) → IO[Result[User, ApiError]] {
  // Clearly shows: does IO, can fail
}
```

**Don't do this:**
```clank
ƒ fetch_user(id: UserId) → User {
  // Hides IO and potential failure
  // Likely panics internally
}
```

**Why:** Effect types document what a function does beyond computing a value.

---

## Naming Conventions

### Use Descriptive Names

| Kind | Convention | Examples |
|------|------------|----------|
| Types | PascalCase | `User`, `HttpResponse`, `ParseError` |
| Functions | snake_case | `get_user`, `parse_config` |
| Variables | snake_case | `user_id`, `config_path` |
| Type params | Single uppercase | `T`, `E`, `K`, `V` |
| Constants | SCREAMING_SNAKE | `MAX_SIZE`, `DEFAULT_PORT` |

### Predicate Functions

**Do this:**
```clank
ƒ is_empty(arr: [T]) → 𝔹
ƒ has_permission(user: User, perm: Permission) → 𝔹
ƒ can_edit(doc: Document, user: User) → 𝔹
```

### Conversion Functions

**Do this:**
```clank
ƒ to_string(n: ℤ) → 𝕊
ƒ from_json(json: Json) → Result[T, ParseError]
ƒ into_bytes(s: 𝕊) → [Byte]
ƒ as_slice(arr: [T]) → Slice[T]
```

---

## Module Organization

### Group by Feature, Not by Type

**Do this:**
```
src/
├── user/
│   ├── types.clank      # User, UserId, etc.
│   ├── validation.clank # validate_user, etc.
│   └── repository.clank # get_user, save_user, etc.
├── order/
│   ├── types.clank
│   ├── validation.clank
│   └── repository.clank
└── main.clank
```

**Don't do this:**
```
src/
├── types/
│   ├── user.clank
│   ├── order.clank
│   └── ...
├── validation/
│   ├── user.clank
│   ├── order.clank
│   └── ...
└── main.clank
```

**Why:** Feature grouping keeps related code together, making changes localized.

---

### Export Deliberately

**Do this:**
```clank
// In user/mod.clank
pub use types::{User, UserId}
pub use validation::validate_user
// Internal types not exported
```

**Don't do this:**
```clank
// Export everything
pub use types::*
pub use validation::*
```

**Why:** Explicit exports define your public API and prevent accidental dependencies.

---

## Performance

### Avoid Repeated Computation

**Do this:**
```clank
ƒ process(items: [Item]) → [Result] {
  let config = load_expensive_config()  // Once
  items |> map(λitem → process_item(item, config))
}
```

**Don't do this:**
```clank
ƒ process(items: [Item]) → [Result] {
  items |> map(λitem → {
    let config = load_expensive_config()  // Every iteration!
    process_item(item, config)
  })
}
```

---

### Use Appropriate Collection Types

| Need | Type | Notes |
|------|------|-------|
| Ordered sequence | `[T]` | Default choice |
| Key-value lookup | `Map[K, V]` | O(log n) access |
| Unique elements | `Set[T]` | O(log n) membership |
| FIFO queue | `Queue[T]` | O(1) push/pop |
| Stack | `Stack[T]` | O(1) push/pop |

---

## Working with the Compiler

### Trust Compiler Repairs

**Do this:**
```
1. Submit AST
2. If repairs available, apply highest-confidence behavior_preserving repair
3. Resubmit
4. Repeat until success
```

**Don't do this:**
- Manually edit when repairs are available
- Ignore repair suggestions
- Apply behavior_changing repairs without review

---

### Use Type Holes for Exploration

**Do this:**
```clank
ƒ example() → ??? {  // Let compiler infer return type
  let x: ??? = some_expression()  // Let compiler infer type
  x
}
```

The compiler will report what types satisfy the holes.

---

### Add Assertions to Help the Solver

**Do this (when stuck on refinement):**
```clank
ƒ example(x: ℤ) → ℤ{self > 0} {
  let y = compute(x)
  assert y > 0  // Helps solver understand y's constraint
  y
}
```

**Why:** Assertions add facts to the solver's context, potentially enabling proof discharge.

---

## Summary: Core Principles

1. **Types first** — Design your types before your functions
2. **Make illegal states unrepresentable** — Use sum types and refinements
3. **Be explicit** — About effects, errors, and constraints
4. **Prefer expressions** — Over statements
5. **Trust the compiler** — Apply its repairs, use its suggestions
6. **Push effects to edges** — Keep core logic pure
7. **Small, focused functions** — Compose larger behaviors from small parts
