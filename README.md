Posita

Explicit. Static. Safe.

Posita is a systems programming language designed for safety‑critical domains where every bit of representation, every overflow policy, and every error path must be explicitly stated and statically verified. It combines Ada’s precision, Rust’s modern type system, and compile‑time verification powered by SMT solvers to eliminate undefined behaviour without runtime overhead.

The compiler frontend is under active development and has already reached a significant milestone: a complete implementation of parsing, name resolution, type inference, and type checking for the full language surface, backed by a comprehensive test suite. This frontend serves as the foundation for the upcoming backend phases, which will generate machine code and integrate SMT‑based verification.

Current Status (Frontend)

The compiler currently provides:

- **Full lexical analysis** covering all keywords, operators, literals (integer, float, char, string, byte string), escape sequences, and comments (line, block, doc).
- **Complete parser** for the entire grammar, including functions, types, traits, impl blocks, generics, pattern matching, closures, contracts, comptime blocks, poly/unbox, and quantified expressions.
- **Name resolution** with scoping, imports, aliases, and symbol tables, producing a resolution map consumed by the type checker.
- **Type system** with a rich set of built‑in types (integers, floats, rationals, arrays, slices, references, pointers, tuples, structs, enums, dyn traits, existential types, and first‑class polymorphic types via poly/unbox).
- **Hindley‑Milner style type inference** with support for:
  - Generic type parameters and automatic instantiation.
  - Subtyping and variance (covariant, contravariant, invariant).
  - OmniML‑inspired shape variables and partial generalization (PG/PI) for let‑polymorphism.
  - SMT‑assisted unicity checks (via Z3) to resolve ambiguous constraints.
  - Region tree for tracking break/continue targets and inference scopes.
- **Trait system** with built‑in traits (`Add`, `Sub`, `Mul`, `Div`, `Rem`, `Eq`, `Lt`, `And`, `Or`, `Future`, etc.) and support for user‑defined traits and impls, including associated types and method resolution.
- **Contract support** (`requires`, `ensures`, `invariant`, `decreases`, `terminates`) with SCAP‑style guarantee chaining for post‑condition verification.
- **Comptime evaluation** framework (including `comptime def`, `comptime { ... }`, and built‑in comptime functions like `@assert`, `@compile_error`, `@typeInfo`).
- **First‑class polymorphism** via `poly(expr)` (boxing) and `unbox(expr)` (instantiation).
- **Fixed‑precision rational types** `Rational<p,q>` with exact arithmetic for contract reasoning.
- **Exhaustiveness checking** for match expressions on enums and finite types.
- **Automatic dereferencing** for method and field access.
- A **growing test suite** with hundreds of test cases covering all major language features.

The frontend is written in Rust and is designed to be modular, incremental, and ready for integration with a backend code generator and SMT solver (Z3) for full verification.

Building and Testing

The compiler uses a standard Rust toolchain. To build and run the tests:

```bash
cargo build
cargo test
```

All frontend tests should pass, including the extensive integration tests in `checker/tests.rs`.

Next Steps (Backend and Verification)

Work is now shifting towards the backend:

- **IR generation** – Lower HIR to a typed intermediate representation suitable for optimisation and code generation.
- **Machine code generation** – Target common architectures (x86‑64, ARM) with explicit control over memory layout, alignment, and calling conventions.
- **SMT‑based verification** – Encode contracts and invariants into SMT‑LIB2 and integrate with Z3 to prove correctness at compile time.
- **Package manager and build system** – Support for multi‑crate projects and incremental compilation.

The language syntax and semantics are considered stable for the current development phase. Future changes will be backward‑compatible where possible.

Get Involved

Posita is an open‑source project. Contributions, feedback, and bug reports are welcome. Please refer to the documentation in `docs/` for language specification and design rationale.
