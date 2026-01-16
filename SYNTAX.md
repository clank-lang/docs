# Syntax Reference

Complete grammar reference for the Clank programming language.

## Contents

- [Unicode and ASCII Modes](#unicode-and-ascii-modes)
- [Keywords](#keywords)
- [Operators](#operators)
- [Operator Precedence](#operator-precedence)
- [Literals](#literals)
- [Comments](#comments)
- [Grammar (EBNF)](#grammar-ebnf)
- [Syntactic Examples](#syntactic-examples)

## Unicode and ASCII Modes

Clank supports both Unicode operators (preferred) and ASCII fallbacks. The compiler accepts either form and normalizes to Unicode in the canonical AST.

### Unicode Operators

| Symbol | Name | ASCII Fallback | Category |
|--------|------|----------------|----------|
| `ƒ` | function | `fn` | Declaration |
| `τ` | type | `type` | Declaration |
| `λ` | lambda | `\` | Expression |
| `→` | arrow | `->` | Type/Match |
| `←` | assign | `<-` | Statement |
| `⇒` | fat arrow | `=>` | Match arm |
| `×` | multiply | `*` | Arithmetic |
| `÷` | divide | `/` | Arithmetic |
| `≠` | not equal | `!=` | Comparison |
| `≤` | less or equal | `<=` | Comparison |
| `≥` | greater or equal | `>=` | Comparison |
| `∧` | logical and | `&&` | Logical |
| `∨` | logical or | `\|\|` | Logical |
| `¬` | logical not | `!` | Logical |
| `∈` | element of | `in` | Membership |
| `∉` | not element of | `!in` | Membership |
| `∀` | for all | `forall` | Quantifier |
| `∃` | exists | `exists` | Quantifier |
| `⊂` | subset | `subset` | Set |
| `⊃` | superset | `superset` | Set |
| `∪` | union | `union` | Set |
| `∩` | intersection | `intersect` | Set |
| `∅` | empty set | `empty` | Set |
| `∞` | infinity | `inf` | Numeric |
| `≡` | identical | `===` | Identity |
| `≢` | not identical | `!==` | Identity |
| `⊤` | top/true | `true` | Boolean |
| `⊥` | bottom/false | `false` | Boolean |

### Type Symbols

| Symbol | Name | ASCII Fallback | Description |
|--------|------|----------------|-------------|
| `ℕ` | natural | `Nat` | Natural numbers (≥0) |
| `ℤ` | integer | `Int` | Integers |
| `ℝ` | real | `Real` or `Float` | Real/floating-point numbers |
| `ℚ` | rational | `Rational` | Rational numbers |
| `𝔹` | boolean | `Bool` | Boolean values |
| `ℂ` | complex | `Complex` | Complex numbers |
| `𝕊` | string | `Str` | UTF-8 strings |

## Keywords

### Reserved Keywords

```
fn       function declaration (ASCII for ƒ)
type     type alias (ASCII for τ)
rec      record type
sum      sum type (tagged union)
let      variable binding
mut      mutable binding
if       conditional
else     conditional branch
match    pattern matching
for      iteration
while    loop
loop     infinite loop
break    exit loop
continue skip iteration
return   early return
use      import
mod      module
pub      public visibility
priv     private visibility
where    refinement constraint
pre      precondition
post     postcondition
assert   runtime assertion
external FFI declaration
as       type coercion
is       type test
in       membership test (ASCII for ∈)
true     boolean true (ASCII for ⊤)
false    boolean false (ASCII for ⊥)
```

### Contextual Keywords

```
self     self-reference in refinements
result   return value in postconditions
it       implicit parameter
_        wildcard pattern
```

### Effect Keywords

```
IO       I/O effect
Err      Error effect
Async    Asynchronous effect
Mut      Mutation effect
Pure     No effects (default)
Linear   Linear resource
Affine   Affine resource
```

## Operators

### Arithmetic Operators

| Operator | Unicode | Meaning | Associativity |
|----------|---------|---------|---------------|
| `+` | - | Addition | Left |
| `-` | - | Subtraction | Left |
| `*` | `×` | Multiplication | Left |
| `/` | `÷` | Division | Left |
| `%` | - | Modulo | Left |
| `^` | - | Exponentiation | Right |
| `-` | - | Unary negation | Prefix |

### Comparison Operators

| Operator | Unicode | Meaning |
|----------|---------|---------|
| `==` | - | Equality |
| `!=` | `≠` | Inequality |
| `<` | - | Less than |
| `>` | - | Greater than |
| `<=` | `≤` | Less or equal |
| `>=` | `≥` | Greater or equal |

### Logical Operators

| Operator | Unicode | Meaning | Associativity |
|----------|---------|---------|---------------|
| `&&` | `∧` | Logical AND | Left |
| `\|\|` | `∨` | Logical OR | Left |
| `!` | `¬` | Logical NOT | Prefix |

### Other Operators

| Operator | Meaning | Usage |
|----------|---------|-------|
| `\|>` | Pipe | `x \|> f` is `f(x)` |
| `++` | Concatenation | Arrays, strings |
| `?` | Propagation | Early return on None/Err |
| `??` | Null coalesce | Default on None |
| `..` | Range (exclusive) | `0..10` |
| `..=` | Range (inclusive) | `0..=10` |
| `.` | Field access | `obj.field` |
| `::` | Path separator | `Mod::Type` |

## Operator Precedence

From highest to lowest precedence:

| Level | Operators | Associativity |
|-------|-----------|---------------|
| 1 | `.` `::` `[]` `()` | Left |
| 2 | `?` | Postfix |
| 3 | `-` `!` `¬` (unary) | Prefix |
| 4 | `^` | Right |
| 5 | `*` `×` `/` `÷` `%` | Left |
| 6 | `+` `-` | Left |
| 7 | `++` | Left |
| 8 | `..` `..=` | Non-associative |
| 9 | `in` `∈` `∉` | Non-associative |
| 10 | `<` `>` `<=` `≤` `>=` `≥` | Non-associative |
| 11 | `==` `!=` `≠` `===` `≡` `!==` `≢` | Non-associative |
| 12 | `&&` `∧` | Left |
| 13 | `\|\|` `∨` | Left |
| 14 | `??` | Right |
| 15 | `\|>` | Left |
| 16 | `=` `←` | Right |

## Literals

### Integer Literals

```clank
42           // Decimal
0x2A         // Hexadecimal
0o52         // Octal
0b101010     // Binary
1_000_000    // With separators
```

### Float Literals

```clank
3.14         // Basic
3.14e10      // Scientific
3.14e-10     // Negative exponent
1_234.567_8  // With separators
```

### String Literals

```clank
"hello"              // Basic string
"line1\nline2"       // Escape sequences
"tab\there"          // Tab
"quote: \""          // Escaped quote
"unicode: \u{1F600}" // Unicode escape
```

**Escape sequences:**

| Sequence | Meaning |
|----------|---------|
| `\n` | Newline |
| `\r` | Carriage return |
| `\t` | Tab |
| `\\` | Backslash |
| `\"` | Double quote |
| `\0` | Null |
| `\u{XXXX}` | Unicode codepoint |

### Raw Strings

```clank
r"no \escapes here"
r#"can contain "quotes""#
r##"can contain "#" too"##
```

### Multiline Strings

```clank
"""
  Multiline string
  with automatic indent stripping
"""
```

### Boolean Literals

```clank
true   // or ⊤
false  // or ⊥
```

### Unit Literal

```clank
()     // Unit value
```

### Array Literals

```clank
[]                  // Empty array
[1, 2, 3]           // With elements
[1, 2, 3,]          // Trailing comma allowed
[x; 10]             // Repeat: [x, x, x, ...] 10 times
```

### Tuple Literals

```clank
()                  // Unit (empty tuple)
(1,)                // Single-element tuple (note comma)
(1, 2)              // Pair
(1, "a", true)      // Triple
```

### Record Literals

```clank
{ name: "Alice", age: 30 }
{ name, age }       // Shorthand when variable matches field
{ ...other, age: 31 }  // Spread with override
```

## Comments

### Line Comments

```clank
// This is a line comment
let x = 42  // Comment at end of line
```

### Block Comments

```clank
/* This is a
   block comment */

/* Block comments /* can nest */ properly */
```

### Doc Comments

```clank
/// Function documentation
/// Multiple lines are combined
ƒ add(a: ℤ, b: ℤ) → ℤ { a + b }

//! Module-level documentation
//! Describes the entire module
```

## Grammar (EBNF)

### Program Structure

```ebnf
program     = declaration* ;

declaration = fn_decl
            | rec_decl
            | sum_decl
            | type_alias
            | use_decl
            | mod_decl
            | external_decl ;
```

### Function Declaration

```ebnf
fn_decl     = visibility? "fn" IDENT type_params? "(" params? ")" return_type?
              precond? postcond? block ;

visibility  = "pub" | "priv" ;

type_params = "[" type_param ("," type_param)* ","? "]" ;
type_param  = IDENT (":" type_constraint)? ;
type_constraint = IDENT ("+" IDENT)* ;

params      = param ("," param)* ","? ;
param       = IDENT ":" type_expr ;

return_type = "→" type_expr | "->" type_expr ;

precond     = "pre" expr ;
postcond    = "post" expr ;
```

### Type Declarations

```ebnf
rec_decl    = "rec" IDENT type_params? "{" field_decls "}" ;
field_decls = field_decl ("," field_decl)* ","? ;
field_decl  = IDENT ":" type_expr ;

sum_decl    = "sum" IDENT type_params? "{" variants "}" ;
variants    = variant ("," variant)* ","? ;
variant     = IDENT ("(" type_expr ("," type_expr)* ")")? ;

type_alias  = "type" IDENT type_params? "=" type_expr ;
```

### Type Expressions

```ebnf
type_expr   = named_type
            | array_type
            | tuple_type
            | function_type
            | refined_type
            | effect_type
            | record_type ;

named_type  = IDENT type_args? ;
type_args   = "[" type_expr ("," type_expr)* ","? "]" ;

array_type  = "[" type_expr "]" ;

tuple_type  = "(" ")"
            | "(" type_expr "," ")"
            | "(" type_expr ("," type_expr)+ ","? ")" ;

function_type = "(" type_expr ("," type_expr)* ")" "→" type_expr ;

refined_type = type_expr "{" IDENT? "|"? expr "}" ;

effect_type = IDENT "[" type_expr "]" ;

record_type = "{" field_types ("," "...")? "}" ;
field_types = field_type ("," field_type)* ","? ;
field_type  = IDENT ":" type_expr ;
```

### Expressions

```ebnf
expr        = assignment ;

assignment  = IDENT "=" expr
            | logical_or ;

logical_or  = logical_and ("||" logical_and)* ;
logical_and = equality ("&&" equality)* ;
equality    = comparison (("==" | "!=") comparison)* ;
comparison  = range (("<" | ">" | "<=" | ">=") range)* ;
range       = concat ((".." | "..=") concat)? ;
concat      = additive ("++" additive)* ;
additive    = multiplicative (("+" | "-") multiplicative)* ;
multiplicative = power (("*" | "/" | "%") power)* ;
power       = unary ("^" power)? ;
unary       = ("!" | "-") unary | postfix ;
postfix     = primary (call | index | field | propagate)* ;

call        = "(" args? ")" ;
args        = expr ("," expr)* ","? ;
index       = "[" expr "]" ;
field       = "." IDENT ;
propagate   = "?" ;

primary     = literal
            | IDENT
            | "(" expr ")"
            | block
            | if_expr
            | match_expr
            | lambda
            | array_expr
            | tuple_expr
            | record_expr ;
```

### Statements

```ebnf
statement   = let_stmt
            | assign_stmt
            | for_stmt
            | while_stmt
            | loop_stmt
            | return_stmt
            | break_stmt
            | continue_stmt
            | assert_stmt
            | expr_stmt ;

let_stmt    = "let" "mut"? pattern (":" type_expr)? "=" expr ;
assign_stmt = expr "=" expr ;
for_stmt    = "for" pattern "in" expr block ;
while_stmt  = "while" expr block ;
loop_stmt   = "loop" block ;
return_stmt = "return" expr? ;
break_stmt  = "break" ;
continue_stmt = "continue" ;
assert_stmt = "assert" expr ("," STRING)? ;
expr_stmt   = expr ;

block       = "{" statement* expr? "}" ;
```

### Control Flow

```ebnf
if_expr     = "if" expr block ("else" (if_expr | block))? ;

match_expr  = "match" expr "{" match_arms "}" ;
match_arms  = match_arm ("," match_arm)* ","? ;
match_arm   = pattern guard? "→" expr ;
guard       = "if" expr ;

lambda      = "\" params? "→" expr
            | "λ" params? "→" expr ;
```

### Patterns

```ebnf
pattern     = wildcard
            | literal_pattern
            | ident_pattern
            | tuple_pattern
            | record_pattern
            | variant_pattern
            | range_pattern ;

wildcard    = "_" ;
literal_pattern = INTEGER | FLOAT | STRING | "true" | "false" ;
ident_pattern = IDENT ;
tuple_pattern = "(" pattern ("," pattern)* ","? ")" ;
record_pattern = "{" field_pattern ("," field_pattern)* ","? "}" ;
field_pattern = IDENT (":" pattern)? ;
variant_pattern = IDENT ("(" pattern ("," pattern)* ")")? ;
range_pattern = literal ".." literal | literal "..=" literal ;
```

### Lexical Rules

```ebnf
IDENT       = (ALPHA | "_") (ALPHA | DIGIT | "_")* ;
INTEGER     = DIGIT (DIGIT | "_")* | "0x" HEX+ | "0o" OCTAL+ | "0b" BINARY+ ;
FLOAT       = DIGIT+ "." DIGIT+ (("e" | "E") ("+" | "-")? DIGIT+)? ;
STRING      = '"' (ESCAPE | [^"\\])* '"' ;
ESCAPE      = "\\" ([nrt\\'"0] | "u{" HEX+ "}") ;

ALPHA       = [a-zA-Z] | UNICODE_LETTER ;
DIGIT       = [0-9] ;
HEX         = [0-9a-fA-F] ;
OCTAL       = [0-7] ;
BINARY      = [01] ;
```

## Syntactic Examples

### Function Definitions

```clank
// Basic function
ƒ add(a: ℤ, b: ℤ) → ℤ {
  a + b
}

// ASCII equivalent
fn add(a: Int, b: Int) -> Int {
  a + b
}

// Generic function
ƒ identity[T](x: T) → T { x }

// Function with refinement
ƒ divide(n: ℤ, d: ℤ{d ≠ 0}) → ℤ {
  n ÷ d
}

// Function with pre/post conditions
ƒ sqrt(n: ℝ) → ℝ
  pre n ≥ 0
  post result × result == n
{
  // implementation
}
```

### Type Definitions

```clank
// Type alias
τ UserId = ℤ{self > 0}

// Record type
rec User {
  id: UserId,
  name: 𝕊,
  email: 𝕊{len(self) > 0}
}

// Sum type
sum Option[T] {
  Some(T),
  None
}

// Sum with multiple fields
sum Result[T, E] {
  Ok(T),
  Err(E)
}
```

### Pattern Matching

```clank
match value {
  0 → "zero",
  1 → "one",
  n if n < 0 → "negative",
  _ → "other"
}

match option {
  Some(x) → x,
  None → default
}

match point {
  (0, 0) → "origin",
  (x, 0) → "on x-axis",
  (0, y) → "on y-axis",
  (x, y) → "elsewhere"
}
```

### Control Flow

```clank
// If expression
let result = if x > 0 {
  "positive"
} else if x < 0 {
  "negative"
} else {
  "zero"
}

// For loop
for item ∈ items {
  process(item)
}

// While loop
while condition {
  step()
}

// Loop with break
loop {
  if done() { break }
  work()
}
```

### Lambdas

```clank
// Full form
let f = λ(x: ℤ) → x × 2

// ASCII form
let f = \(x: Int) -> x * 2

// Type-inferred
let f = λx → x × 2

// Multiple parameters
let add = λ(a, b) → a + b

// Used inline
items |> map(λx → x × 2) |> filter(λx → x > 10)
```

### Refinement Types

```clank
// Basic refinement
τ Positive = ℤ{self > 0}

// With explicit variable
τ Percentage = ℝ{p | p ≥ 0.0 ∧ p ≤ 100.0}

// Array refinement
τ NonEmpty[T] = [T]{len(self) > 0}

// Function parameter refinement
ƒ get_first[T](arr: [T]{len(arr) > 0}) → T {
  arr[0]
}
```

### Effects

```clank
// IO effect
ƒ greet(name: 𝕊) → IO[()] {
  println("Hello, " ++ name)
}

// Error effect
ƒ parse_int(s: 𝕊) → Err[ParseError, ℤ] {
  // ...
}

// Combined effects
ƒ read_config(path: 𝕊) → IO + Err[IoError, Config] {
  let contents = read_file(path)?
  parse_config(contents)?
}
```
