# Clank Documentation

Documentation for [Clank](https://github.com/clank-lang/clank), an agent-first programming language designed for LLMs.

## What is Clank?

Clank is a programming language where:

- **AST JSON is the canonical representation** — Programs can be submitted as JSON, not just source text
- **The compiler returns machine-actionable repairs** — Errors come with ranked fix suggestions that agents can apply directly
- **Refinement types encode invariants** — Express constraints like "positive integer" or "non-empty array" in the type system
- **Unicode syntax is first-class** — Use `ƒ`, `→`, `∧`, `ℤ` for clarity (with ASCII fallbacks)

```clank
// Function with refinement type
ƒ divide(n: ℤ, d: ℤ{self ≠ 0}) → ℤ {
  n ÷ d
}

// The compiler proves d ≠ 0 at call sites
// No runtime division-by-zero possible
```

## For Humans

- [Language Website](https://clank-lang.org) — Tutorials and guides
- [Language Repository](https://github.com/clank-lang/clank) — Compiler source
- [Specification](https://github.com/clank-lang/clank/blob/main/spec/) — Formal language spec

## For AI Agents

This repository is optimized as a **Claude Code skill**. The documentation is structured for LLM consumption with:

- Consistent formatting across all files
- Machine-readable examples
- Explicit "do this / don't do this" patterns
- Structured error catalogs with fix patterns

### Using as a Skill

Add to your Claude Code configuration:

```json
{
  "skills": ["github:clank-lang/docs"]
}
```

Claude Code will automatically load relevant documentation when working with `.clank` files.

## Documentation Files

| File | Description |
|------|-------------|
| [SKILL.md](SKILL.md) | Skill entry point and quick reference |
| [SYNTAX.md](SYNTAX.md) | Complete grammar, operators, and keywords |
| TYPES.md *(planned)* | Type system — primitives, refinements, effects |
| AST-JSON.md *(planned)* | AST node schema for JSON input/output |
| COMPILER.md *(planned)* | Compiler interface, flags, and output format |
| REPAIRS.md *(planned)* | Repair system, PatchOps, and fix patterns |
| [EXAMPLES.md](EXAMPLES.md) | Annotated code examples by category |
| [IDIOMS.md](IDIOMS.md) | Best practices and anti-patterns |

## Quick Start

### Install

```bash
# Clone the Clank compiler
git clone https://github.com/clank-lang/clank
cd clank

# Install with mise (manages Bun)
mise install
mise run install
```

### Compile a Program

```bash
# From source file
clank compile hello.clank -o dist/

# Type check only
clank check hello.clank

# Compile and run
clank run hello.clank
```

### Agent Workflow (AST JSON)

```bash
# Compile from JSON AST
clank compile program.json --input=ast -o dist/

# Get structured diagnostics
clank compile program.json --input=ast --emit=json
```

## Example Program

```clank
// Types first
τ PositiveInt = ℤ{self > 0}

rec User {
  id: PositiveInt,
  name: 𝕊{len(self) > 0},
  email: 𝕊
}

// Function with effects
ƒ greet(user: User) → IO[()] {
  println("Hello, " ++ user.name ++ "!")
}

// Entry point
ƒ main() → IO[()] {
  let user = User { id: 1, name: "Alice", email: "alice@example.com" }
  greet(user)
}
```

## Contributing

Found an error or want to improve the documentation?

1. Fork this repository
2. Make your changes
3. Submit a pull request

Please ensure examples compile and follow the patterns established in existing docs.

## License

MIT License — see [LICENSE](LICENSE) for details.
