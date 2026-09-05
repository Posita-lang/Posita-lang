# Posita

*Explicit. Static. Verified.*

<p align="center">
  <img width="1254" height="1254" alt="image" src="https://github.com/user-attachments/assets/f76e6d3b-303e-4e8d-9866-46e672e0597e" />
</p>



Posita is a systems programming language designed for safety‑critical domains where every bit of representation, every overflow policy, and every error path must be explicitly stated and statically verified. It combines Ada’s precision, Rust’s modern type system, and compile‑time verification powered by SMT solvers to eliminate undefined behaviour without runtime overhead.

The compiler frontend is under active development and already covers a substantial portion of the core language. Parsing, name resolution, type inference, and type checking are implemented for the current feature set, with a growing test suite that catches regressions as the design evolves. Work is firmly focused on the frontend — the backend and SMT integration are planned but not yet underway.

> **Note on documentation and completeness:** The language specification and syntax documentation are being written alongside the implementation. In some areas the implementation leads the docs; in others (like lexer corner cases) we are still filling gaps. The lexer currently handles the great majority of keywords and token forms, but edge cases remain.

Current Status (Frontend)

The compiler currently provides:

- **Lexical analysis** covering most keywords, operators, literals (integer, float, char, string, byte string), common escape sequences, and comments (line, block, doc). Some less frequent token forms are still being added.
- **Parser** for the current grammar, including functions, types, traits, impl blocks, generics, pattern matching, closures, contracts, comptime blocks, poly/unbox, and quantified expressions.
- **Name resolution** with scoping, imports, aliases, and symbol tables, producing a resolution map consumed by the type checker.
- **Type system** with a rich set of built‑in types (integers, floats, rationals, arrays, slices, references, pointers, tuples, structs, enums, dyn traits, existential types, and first‑class polymorphic types via poly/unbox).
- **Hindley‑Milner style type inference** with support for:
  - Generic type parameters and automatic instantiation.
  - Subtyping and variance (covariant, contravariant, invariant).
  - OmniML‑inspired shape variables and partial generalization (PG/PI) for let‑polymorphism.
  - SMT‑assisted unicity checks (via Z3) to resolve ambiguous constraints.
  - Region tree for tracking break/continue targets and inference scopes.
- **Trait system** with built‑in traits (`Add`, `Sub`, `Mul`, `Div`, `Rem`, `Eq`, `Lt`, `And`, `Or`, `Future`, etc.) and support for user‑defined traits and impls, including associated types and method resolution.
- **Contract support** (`requires`, `ensures`, `invariant`, `decreases`, `terminates`) — initial integration with the type checker and SCAP‑style guarantee chaining for post‑condition verification.
- **Comptime evaluation** framework (including `comptime def`, `comptime { ... }`, and built‑in comptime functions like `@assert`, `@compile_error`, `@typeInfo`).
- **First‑class polymorphism** via `poly(expr)` (boxing) and `unbox(expr)` (instantiation).
- **Fixed‑precision rational types** `Rational<p,q>` with exact arithmetic for contract reasoning.
- **Exhaustiveness checking** for match expressions on enums and finite types.
- **Automatic dereferencing** for method and field access.
- A **growing test suite** with hundreds of test cases covering the implemented feature surface.

The frontend is written in Rust and is designed to be modular and incremental. The type checker and inference engine are the current focus; once the frontend stabilizes, work will begin on IR lowering, code generation, and SMT verification.

The language syntax and semantics are still evolving. We aim for stability and backward compatibility as the design matures, but breaking changes may occur during this development phase.

Building and Testing

The compiler uses a standard Rust toolchain. To build and run the tests:

```bash
cargo build
cargo test
```

All frontend tests should pass, including the integration tests in `checker/tests.rs`.

Next Steps (Backend and Verification — Future)

Once the frontend reaches a solid baseline, planned work includes:

- **IR generation** – Lower HIR to a typed intermediate representation suitable for optimisation and code generation.
- **Machine code generation** – Target common architectures (x86‑64, ARM) with explicit control over memory layout, alignment, and calling conventions.
- **SMT‑based verification** – Encode contracts and invariants into SMT‑LIB2 and integrate with Z3 to prove correctness at compile time.
- **Package manager and build system** – Support for multi‑crate projects and incremental compilation.

Get Involved

Posita is an open‑source project. Contributions, feedback, and bug reports are welcome. Please refer to the documentation in `docs/` for language specification and design rationale.
