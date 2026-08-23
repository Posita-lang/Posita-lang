# Posita Language Syntax
Document revision: 2026-08-23 (working draft, not a frozen specification)

> This revision integrates graded function types (GrTT), erased value
> indices, and the `Lease` modality. `@consume` is extended from a
> binder-local assertion to a component of function types (breaking;
> see Graded Function Types).

[!NOTE]
This document version tracks its own edits. It does not correspond to a language specification release.
Posita itself is in pre‑alpha; the syntax is under active design and may change without notice.

## Design Philosophy

Posita is a ultra‑static, systems programming language where the programmer explicitly posits every representation detail: bit widths, pointer sizes, default values, error paths, overflow behavior, and even resource consumption protocols. All decisions are made visible in source and enforced at compile time with zero runtime overhead.

- **Explicit over implicit:** No hidden ABI, no type‑erased errors, no null pointers, no implicit overflow, no invisible allocations. All critical semantic effects—compile‑time execution (`!`), error propagation (`?`), asynchronous suspension (`await`)—are marked with dedicated syntax at the point of use. Unsafe operations are confined to functions marked `@trusted` at the point of declaration, establishing explicit trust boundaries. Reviewers can see every important behavior without reading function definitions.

- **Explicit commitments, implicit verification.** Posita distinguishes
  *analysis* from *commitment*. The compiler implicitly analyzes all code —
  type inference, borrow checking, usage-grade inference — because analysis
  verifies properties without changing program behavior. What must be
  explicit is anything that enters an interface and constrains callers:
  types, declared grades, effects, contracts. An unannotated binder makes
  no usage commitment; a graded binder is an interface commitment,
  enforced and propagated mechanically.

- **Compile‑time over run‑time:** Error handling, defaults, optimizations, reflection, and resource tracking are resolved statically. A function may optionally defer contract checking to runtime via `@runtime_check`.

- **Readable as documentation**: English keywords (`def`, `set`, `leave`, `catch`), not cryptic symbols. Specification tags (`@spec`, `@requirement`, `@rationale`) link code directly to system requirements.

- **No undefined behavior in safe code:** Every operation either succeeds with defined semantics or is rejected at compile time.

## Lexical Structure

### Keywords

`def`, `set`, `type`, `with`, `default`, `return`, `if`, `else`, `for`, `in`, `while`, `loop`, `leave`,
`comptime`, `import`, `as`, `true`, `false`, `auto`, `and`, `or`, `not`, `sizeof`, `alignof`,
`catch`, `panic`, `unsafe`, `let`, `finally`,
`where`, `requires`, `ensures`, `invariant`, `constraint`, `move`, `dyn`, `by`, `copy`, `ref`, `mut`, `wrap`, `saturate`, `trap`, `Self`, `no_default`, `extern`, `pub`, `edition`, `deprecated`, `experimental`, `endian`, `bit_order`, `align`, `pad`, `packed`, `async`, `await`, `task`, `channel`, `linear`, `consume`, `pure`, `io`, `trusted`, `ghost`, `scope_cleanup`, `trigger`, `validate`, `missing_match`, `apply_lemma`, `exists`, `implies`, `trait`, `impl`, `decreases`, `terminates`, `cfg`, `isolate`, `hint`, `must_use`, `must_handle`, `link_proof`, `exhaustive`, `no_alloc_error`, `no_panic`, `debug_info`, `old`, `codomain`, `audit_log`, `interrupt`, `ieee`, `ieee_contracts`, `diverges`, `propagates`, `poly`, `unbox`, `overrides`, `layout`, `when`, `const`, `forall`

`Int`, `UInt`, `Ptr`, `Str`, `String`, `Result`, `Option`, `usize`, `Float` are built‑in type constructors, not reserved words. `linear`, `consume` are planned keywords; `by` is reserved for closure capture syntax.

### Identifiers

`[a‑zA-Z_][a‑zA-Z0-9_]*`

### Literals

Integers: `42`, `0xFF`, `0b1010`

Integer suffixes for explicit bit‑width: `42i32` (equivalent to `42: Int<32>`), `0xFFu8` (equivalent to `0xFF: UInt<8>`). The suffix is syntactic sugar and the type is fully checked.

Floats: `3.14`, `2.5e-3` (type `Float<64>` by default)

Characters: `'a'` (type `Char`, a Unicode scalar value)

Byte characters: `b'x'` (type `Byte`, an alias for `UInt<8>`)

Byte strings: `b"hello\n"` (type `&[Byte]`, see Escape Sequences)

Strings: `"hello"` (type `&Str`, guaranteed valid UTF‑8, see Escape Sequences)

Booleans: `true`, `false`

Type‑annotated literals: Any literal may be annotated with a type using the `: Type` syntax. For integer literals, this includes refined types (e.g., `1: PositiveInt`). The compiler checks at compile time that the literal satisfies the target type's invariants; failure is a compile‑time error. This annotation is particularly useful for disambiguating generic or overloaded contexts, and for initializing refined types without relying on type inference from a variable declaration.

```
set one: PositiveInt = 1;               // type inferred from declaration
set also_one = 1: PositiveInt;          // type explicitly annotated on the literal
```

Built‑in type aliases:

- `type Byte = UInt<8>;` — raw byte, used in byte strings and byte slices.
- `type Char = UInt<32>;` with invariant `value <= 0x10FFFF` and `not (0xD800 <= value <= 0xDFFF)` — a Unicode scalar value.

### Escape Sequences

The following escape sequences are recognized in string literals, byte string literals, and character literals. They are resolved by the lexer at compile time and replaced with the corresponding byte or Unicode scalar value.

| Sequence | Name | Byte Value |
|---|---|---|
| `\n` | Line Feed | 0x0A |
| `\r` | Carriage Return | 0x0D |
| `\t` | Horizontal Tab | 0x09 |
| `\\` | Backslash | 0x5C |
| `\"` | Double Quote | 0x22 |
| `\'` | Single Quote | 0x27 |
| `\0` | Null Character | 0x00 |
| `\xNN` | Hex Byte (2 digits) | 0xNN |
| `\u{NNNNNN}` | Unicode Scalar (1‑6 hex digits) | UTF‑8 encoded bytes |

In character literals (`'...'`), the escape must resolve to exactly one valid Unicode scalar value of type `Char`.

In byte characters (`b'...'`), `\u{...}` is not valid; only byte escapes may appear.

In byte strings (`b"..."`), the same restriction holds.

The sequences `\a`, `\b`, `\v`, `\f`, and `\?` are not recognized in Posita string literals. If needed, use the equivalent `\xNN` form.

### Comments

Line comment: `// ...`

Block comment: `/* ... */`

Documentation comment: `/// ...` (Markdown, code examples are automatically verified as `@comptime_test`)

Module‑level documentation comment: `//! ...` (applies to the enclosing module)

## Specification and Traceability

Posita integrates specification directly into the codebase via documentation comments, enabling reviewers and auditors to trace system requirements to their implementations without leaving the source file.

### Spec Tags

The following tags are used inside `///` documentation comments:

- `@spec` – references an external standard or system requirement document.
- `@requirement` – cites a specific requirement identifier and its description.
- `@rationale` – explains why the implementation satisfies the requirement.
- `@trace` – establishes a traceability link from requirement to implementation.
- `@safety_integrity` – marks the ASIL/DO‑178C/IEC 61508 level associated with the function.

Example:

```
/// @spec TCAS‑II v7.1, Section 3.2.1
/// @requirement REQ‑SAFE‑004: If intruder altitude is within ±1000 ft
///   and vertical separation is projected to be lost within 25 seconds,
///   a Resolution Advisory (RA) must be issued.
/// @rationale This function is the core of the collision avoidance logic.
/// @trace REQ‑SAFE‑004 → issue_resolution_advisory → `requires`/`ensures`
/// @safety_integrity DO‑178C Level A
def issue_resolution_advisory(intruder: AircraftState, own: AircraftState) -> Result<RA, Error>
    requires abs(intruder.altitude - own.altitude) <= 1000
    ensures codomain.is_ok() implies ra_issued()
{
    // ...
}
```

The `capsa spec` command collects all these tags and generates a traceability matrix suitable for certification submissions.

## Type System

Note on semicolons: A semicolon after a type definition is optional. It is typically omitted for simple one‑line definitions and may be used for clarity after complex definitions.

### Value Semantics

Posita defaults to copy semantics for all types that implement the `Copy` trait. A type automatically derives `Copy` if it consists only of integers, floats, other `Copy` types, and does not implement `Drop`. This ensures that "copy is a trivial bitwise replication" and eliminates "moved‑from" invalid states. Size is irrelevant to `Copy` derivation; a large struct of pure data is `Copy`, while a small struct that implements `Drop` is not. Large types like `Vector<T>` are not `Copy` because they manage resources (they implement `Drop`). Explicit `move` semantics are available via the `move` keyword for ownership transfer optimization.

To prevent automatic `Copy` derivation for a type that would otherwise qualify, either implement `Drop` (even with an empty body) or use the `@no_copy` attribute. `@no_copy` is syntactic sugar for implementing an empty `Drop` trait; it is preferred for clarity of intent.

```
@no_copy
type LargeStruct = struct { data: [Int<8>; 1024] };
```

Manual implementation of `Copy` is allowed for types where bitwise replication is semantically sound, even if large. By doing so, the programmer explicitly accepts the performance characteristics.

### Affine and Linear Types

Posita's ownership model is based on affine types: every non‑`Copy` value may be used at most once. "Using" a value means either moving it to a new owner, discarding it (implicitly at scope exit), or consuming it via a destructor.

- Copy types (see Value Semantics) are exempt: they can be duplicated arbitrarily.
- Non‑`Copy` types are affine. They cannot be duplicated. They can be moved, and if not moved out of a scope they are implicitly discarded.
  - If the type implements `Drop`, the compiler calls the destructor at the discard point.
  - If the type does not implement `Drop`, the compiler simply reclaims the memory with no user code executed.
- After a move, the source variable is statically dead – further use is a compile‑time error.

Affine types guarantee that a value is never duplicated or accidentally used after a move, and that every value is eventually cleaned up (destructor or plain deallocation). This eliminates double‑free, use‑after‑move, and forgotten‑resource bugs without runtime overhead.

**Linear types (`@linear`)**

Some resources must never be silently discarded, even without a destructor. For these, Posita provides the `@linear` attribute on a type definition:

```
@linear
type SessionToken = struct { id: Int<32> }
```

A `@linear` type is affine (non‑`Copy`, no `Drop`) with one additional rule: the compiler forbids implicit discarding. Every value of a linear type must be explicitly terminated before it goes out of scope. Termination is achieved by passing the value to a consuming function (e.g., `token.destroy()`), returning it to the caller, or explicitly discarding it via the standard library function `forget(token)`.

If a linear value could leave a scope without being moved or explicitly forgotten, the compiler emits an error.

Linear types are intended for protocol tokens, hardware descriptors, or any resource where an untracked disappearance would violate safety or audit requirements. All existing ownership and borrowing rules apply unchanged.

Note: `@linear` is a type‑level property, not a parameter qualifier. It only affects implicit discard. If you need to temporarily treat a normal affine value as linear (e.g., require explicit consumption in a critical function), use the `linear` keyword on a function parameter or a local block (planned for a future release).

Borrow checking is the built-in grade analysis specialized to reference types (`&mut T` ≙ exclusive use of grade 1, `&T` ≙ shared use of grade ∞); declared grades generalize this to user-defined resource protocols — see §Graded Function Types.

### Graded Function Types

Parameters may carry usage grades — elements of an ordered semiring —
following Graded Modal Dependent Type Theory (GrTT). A grade is a
component of the function's **type**, not merely a property of its body:

```
def f(@consume(s) x: T) -> B     // type: (x: T @ s) → B
```

**Subject and type grades.** Every binder carries two grades:

- the *subject grade* `s`: how many times `x` is used in the function body;
- the *type grade* `r`: how many times `x` occurs in the result type.

`s` is what `@consume` declares. `r` is derived from the signature and
has no surface syntax (GrTT's `(s, r)` binder pair internalizes exactly
these two numbers).

**Propagation (graded application).** At a call site `f(e)`, the caller's
grade account is charged `s · (subject grade of e)` and
`r · (type grade of e)`. Propagation is mechanical semiring algebra and
is the only mechanism by which usage constraints cross function
boundaries. Contracts never restate counts.

**Inference and declaration.** The compiler infers grades for all code:

1. *Soundness* — affine/linear constraints and `Lease` conservation.
   The borrow checker is this analysis specialized to the built-in
   reference types: `&mut T` is an exclusive use of grade 1, `&T` a
   shared use of grade ∞, and point-level liveness is the point-wise
   reading of grade vectors.
2. *Auditing* — `capsa audit` reports inferred grades and full
   call-chain accounts.
3. *Optimization* — a binder with type grade `r = 0` never occurs in
   the result type; substitutions into return types are elided during
   type checking and monomorphization.

Inferred grades are facts, not commitments; they never enter a
signature. A grade enters a signature only when **declared**.

**Declaration forms.**

- `@consume(s)` — exact: the body must use `x` exactly `s` times.
- `@consume(≤ s)` — bound: the body must use `x` at most `s` times.
  Recommended for public APIs: implementations may reduce usage without
  breaking callers. In both forms the caller is charged the declared
  grade (conservative).
- Affine (non-`Copy`) parameters may declare grade at most 1;
  consuming an affine value more than once requires `Lease`.

`s` may be a literal, a `const` generic parameter, or a `comptime`
function result.

**Composition.**

- With `@linear`: grades bound *how many times* a value is used;
  `@linear` forbids *implicit discarding*. A `@linear` parameter of
  grade 0 must still be explicitly terminated.
- With contracts: grades express counts mechanically; contracts express
  resource semantics (`old()`, state transitions). Grade facts discharge
  matching contract clauses without SMT involvement (one-directional).
- With traits: a trait method's declared grades bound its
  implementations. A closure's grades are inferred as the union of its
  captures' grades and its body.

**Context grades (implementation note).** The compiler internally tracks
how earlier parameters are used to type later parameters (GrTT's context
grade vector). This bookkeeping powers the substitution-elision
optimization; it has no surface syntax.

The following examples remain valid under the graded function type semantics:

```
@linear
type Token = struct { _id: UInt<32> };

type TokenBucket<const N: usize> = struct {
    tokens: [Token; N],
    next: usize,
};

impl<const N: usize> TokenBucket<N> {
    @consume(1)
    def take(&mut self) -> Result<Token, BucketEmpty>
        requires self.next < N        // contract: bucket still has tokens
        ensures self.next == old(self.next) + 1
    {
        if self.next >= N {
            leave with BucketEmpty;
        }
        let t = self.tokens[self.next];
        self.next += 1;
        return Ok(t);
    }
}

// @consume on ordinary function parameters
def compare(@consume(1) a: &Data, @consume(1) b: &Data) -> Bool {
    return a.hash() == b.hash();
}
```

The first example shows `@consume(1)` on a method parameter coexisting with
contracts—the contract constrains the bucket's internal state, while
`@consume(1)` constrains how many times the `self` parameter is used within
the body. The second example shows the same annotation on ordinary function
parameters without any `@linear` type involved.

### Usage Semirings

Grades range over an ordered semiring, selected per module (default
`usage`). Built-in semirings:

| Name | Elements | Meaning |
|---|---|---|
| `usage` | ℕ | exact use counts (default) |
| `bounded` | {0, 1, many} | library-friendly approximation |
| `relevance` | {0, 1} | relevant vs. irrelevant |
| `sec` | lattice lo ≤ hi | information-flow control |

All built-in semirings satisfy the quantitative axioms (1 ≠ 0;
r + s = 0 ⟹ r = s = 0; r · s = 0 ⟹ r = 0 ∨ s = 0), guaranteeing that
grade 0 means *irrelevant and erasable*. User-defined semirings
(`@experimental`) must prove these axioms via `@comptime_test`.

Under `@grades(sec)` the `@consume` syntax changes meaning: grade 0 =
High (must not flow into this context), grade 1 = Low. Violations are
grade mismatches; declassification requires an explicit downgrade
function, flagged by `capsa audit`, and integrates with
`[audit.rules] deny_effects`.

### Const Generics (Compile‑Time Value Parameters)

Types can be parameterized by compile‑time constants, enabling generic definitions
that depend on integer sizes, array lengths, or other configuration values.

#### Declaration

A `const` generic parameter is introduced with the `const` keyword inside the
generic parameter list, similar to a type parameter but binding a value:

```
type Vector<T, const N: usize> = struct {
    data: [T; N],
    length: usize,
}

def identity_matrix<const N: usize>() -> Matrix<N, N>
    requires N > 0
{ ... }

def resize_array<T, const M: usize, const N: usize>(src: &[T; M]) -> [T; N]
    where M <= N
{ ... }
```

- `const N: Type` declares a compile‑time constant named `N` of type `Type`. The type must be one of `usize`, `Int<B>`, `UInt<B>`, or any type whose values are fully known at compile time.
- `const` parameters are implicitly `comptime`; they never exist at runtime and do not contribute to memory layout beyond their instantiation.
- At call sites, `const` arguments must be compile‑time constant expressions (literals, `const` items, or `comptime` function results).

#### Usage in Contracts

`const` parameters participate in contracts exactly like any other value:

```
def get_element<T, const N: usize>(arr: &[T; N], idx: usize) -> &T
    requires idx < N
    ensures codomain == arr[idx]
{ &arr[idx] }
```

The SMT solver treats `const` parameters as symbolic constants, enabling proofs
over entire families of types (e.g., "for all N, ...").

#### Interaction with `comptime`

`comptime` type factories can produce types with `const` parameters by calling
functions that return such types.

Conversely, `const` generics provide a declarative alternative where the
compiler automatically monomorphizes the code for each concrete constant value,
enabling static verification without manual `comptime` expansion.

#### Monomorphization

Every distinct combination of `const` argument values results in a separate
monomorphized instance. The compiler guarantees identical runtime behavior as
if the code were hand‑written for each constant, with all checks resolved at
compile time.

### Ghost Value Indices

Generic parameter lists accept a third kind of parameter besides type
parameters and `const` parameters: a **ghost value index**, introduced
with the existing `ghost` keyword:

```
type Vector<T, ghost n: usize> = struct { ... }

def graded_map<T, U, ghost n: usize, F: Fn(&T) -> U>(
    f: Lease<F, n>,
    xs: &Vector<T, n>,
) -> Vector<U, n>;
```

`ghost` has one meaning everywhere it appears: *exists only in the
proof layer*. A ghost index is never stored, never affects layout, and
never monomorphizes — one code body serves all index values.

**Indices vs. runtime parameters.** A function that needs a value at
runtime takes it as an ordinary parameter and may pass it as an index
argument; its type-level occurrence is erased:

```
def replicate<T>(n: usize, x: T) -> Vector<T, n>;
-- n is a real runtime parameter (the loop count);
-- its occurrence in Vector<T, n> is proof-layer only.
```

A value needed only for type checking is declared `ghost` in the
generic list and is erased from the call entirely.

**The dependency channels:**

| Channel | Syntax | Enters runtime? | Monomorphizes? | Shapes layout? | Checked by |
|---|---|---|---|---|---|
| Type parameter | `T` | no | yes | via use | type checking |
| Const parameter | `const N: usize` | shapes layout | yes | yes | compile-time eval |
| Ghost index | `ghost n: usize` | no | no | no | types + SMT |
| Contract | `requires`/`ensures` | no | no | no | SMT |

**Constraints.** Index sorts are restricted to `usize`, `Int<B>`,
`UInt<B>`, and enums — the decidable fragment the SMT backend handles
(the same restriction already applied to `Regex` in contracts). Indices
and expressions over them must be `@pure` and `Copy`. Ghost indices
participate fully in contracts, invariants, and GADT index equalities.

**Grade interpretation.** The four binder kinds are four points of one
analysis:

| Binder | subject grade | at runtime | compile-time known |
|---|---|---|---|
| value parameter | any | yes | no |
| ghost index (`ghost n: usize`) | 0 | no | no |
| ghost proof parameter (`ghost p: P`) | 0 (contracts only) | no | no |
| const parameter (`const N: usize`) | 0 | erased | yes |

`@consume(0, r)` is the grade-theoretic definition of an index
variable; `ghost set` locals are variables whose subject grade is
always 0. Ghost variables, ghost indices, and const generics are one
mechanism at three strengths.

### Bit‑width Parameterized Integers

Signed: `Int<bits>`, `bits` must be compile‑time constant, 1..64 (up to 128 with `--enable-experimental`).

Unsigned: `UInt<bits>`.

Example: `Int<13>`, `UInt<8>`.

### Floating‑Point Types

`Float<bits>`: IEEE 754‑compliant floating‑point type with the specified bit width. `bits` must be 32 or 64. `Float<32>` corresponds to single precision, `Float<64>` to double precision.

Float literals (`3.14`, `2.5e-3`) have a default type of `Float<64>` unless explicitly annotated.

### Fixed‑Precision Rational Numbers

`Rational<p, q>`: A fixed‑point rational type with `p` integer bits and `q` fractional bits. Arithmetic is performed exactly over the rational domain for contracts. The default overflow policy is `saturate`; `with overflow = trap` may be specified. Conversion to floating‑point is explicit (`as Float<64>`), using round‑to‑nearest ties‑to‑even by default.

### Compile‑Time Regular Expressions

`Regex<"...">` is a type that validates the pattern at compile time and compiles it into a deterministic automaton. It supports safe, zero‑overhead string matching at runtime. Regex types cannot appear in contracts directly; they are for runtime use only.

### Type‑level Constraint Shorthand

Instead of `exists n: Int<32> invariant n > 0`, you may write:

```
type PositiveInt = Int<32> where value > 0;
```

This is syntactic sugar with identical semantics.

### Zero‑Size Types (ZST)

Posita guarantees that any `struct` composed entirely of ZST fields has size zero, and `[ZST; N]` also has size zero. ZST fields in `@packed` structs occupy no space. When `@align(N)` is applied to a ZST type, the alignment is honored but the size remains zero.

### Platform Word Type

`usize` is a built‑in type alias for the unsigned integer whose width equals the target platform's pointer width. On 64‑bit targets it is `UInt<64>`, on 32‑bit targets `UInt<32>`, etc. It is intended for array indexing and pointer‑sized values. Programmers may always write `UInt<64>` or `UInt<32>` explicitly when portability is not a concern.

### Overflow Behavior

Integer overflow is never undefined. The default overflow policy is `trap` (compile‑time error in strict mode, runtime panic otherwise). Programmers can override at the type level:

```
type WrapCount = Int<32> with overflow = wrap;     // two's complement wrap
type SatCount  = Int<32> with overflow = saturate;  // saturation
type StrictCount = Int<32> with overflow = trap;    // trap (default)
```

Or at the operator level with suffixes:

- `+%`, `-%`, `*%` : wrap
- `+?`, `-?`, `*?` : saturate
- `+!`, `-!`, `*!` : trap (assert no overflow)

Division (`/`) and remainder (`%`) do not accept overflow suffixes; division‑by‑zero is handled by contract (`requires b != 0`) or runtime panic.

For signed division, `MIN / -1` (where `MIN` is the most negative value of the type) always traps regardless of the type's overflow policy. This is a representability issue, not a standard overflow, and the compiler will attempt to prove this case unreachable via range analysis.

The compiler uses range analysis and type invariants to statically eliminate overflow checks where possible.

Floating-point overflow is never undefined. The default overflow policy
is `trap` (compile-time error in strict mode, runtime panic otherwise).
In strict mode, the `trap` policy cannot be overridden; in non-strict
mode, programmers can override at the type or operator level.

Type-level override:

```
type IEEE = Float<64> with overflow = ieee;     // IEEE 754 (→ ±∞, NaN, ±∞)
type Safe = Float<64> with overflow = trap;     // trap (default)
type SatF = Float<32> with overflow = saturate; // saturation (clamp to range)
```

Operator-level override (same suffixes as integer overflow):

- `+%`, `-%`, `*%` : IEEE 754 semantics
- `+?`, `-?`, `*?` : saturate
- `+!`, `-!`, `*!` : trap (compile-time check for constant expressions; runtime panic for IEEE anomalies)

Division (`/`) and remainder (`%`) do not accept overflow suffixes; the
overflow policy of the operand type applies.

For compile-time constant floating-point expressions, the compiler
evaluates the operation using IEEE 754 semantics on the host FPU and
checks the exception flags (FE_OVERFLOW, FE_INVALID, FE_DIVBYZERO).
If any flag is set, a compile-time error is raised.

The `@ieee_contracts` attribute controls the interpretation of floating‑point
operations in contracts (`requires`/`ensures`), switching them from the
default mathematical real domain to IEEE 754 semantics. It is orthogonal
to the `overflow` policy: `@ieee_contracts` affects SMT solver reasoning,
while `overflow` affects runtime behavior and `comptime` evaluation.

### Pointers and References

Raw pointer type: `Ptr<size = SizeType, pointee = PointeeType>`

- `size`: type that determines the pointer's own width (e.g., `UInt<16>`).
- `pointee`: the type it points to.

Syntactic sugar: `*T` is a platform‑word‑sized pointer to `T`.

References: `&T` (immutable), `&mut T` (mutable). References are checked at compile time and do not support pointer arithmetic. No null references are allowed; use `Option<&T>` for nullable semantics.

Exclusive borrow: `&mut T` is an exclusive, non‑copyable borrow. While it is live, the original variable is frozen—neither readable nor writable—preventing data races statically.

### Pointer Arithmetic

Pointer arithmetic is permitted only within `unsafe` blocks and is subject to the following rules. The compiler does not enforce memory safety for pointer arithmetic because `unsafe` code explicitly opts out of the language's automatic safety guarantees; it is the programmer's responsibility to ensure that the resulting pointer remains within the bounds of a valid allocation.

**Supported operations**

| Expression | Semantics |
|---|---|
| `ptr + offset` | Advances `ptr` by `offset` elements of type `Pointee`. `offset` must be of type `Ptr::size` or implicitly convertible to it. |
| `ptr - offset` | Reverses `ptr` by `offset` elements. Same type constraint on `offset`. |
| `ptr1 - ptr2` | Returns the number of `Pointee` elements between `ptr1` and `ptr2`. Both pointers must have the same `pointee` type. Result type is `Ptr::size`. |
| `ptr[i]` | Equivalent to `*(ptr + i)`. `i` is subject to the same type constraint as `offset`. |

**Prohibited operations**

The following are rejected at compile time:

- `ptr1 + ptr2`
- `offset + ptr` (commutativity is not assumed; write `ptr + offset`)
- `ptr * n`, `ptr / n`, `ptr % n`
- Any pointer arithmetic involving reference types (`&T`, `&mut T`). References do not support arithmetic; cast to a `Ptr` first.

**Alignment and bounds**

The compiler emits a diagnostic if the result of pointer arithmetic can be statically proven to be misaligned for the `pointee` type. For runtime offsets, the programmer must guarantee alignment via contract or manual assertion; violation results in undefined behavior.

Bounds checking is never performed for pointer arithmetic. Use array types (`[T; N]`) or slices (`&[T]`) for safe, bounds‑checked indexing.

**Minimum addressable unit**

References (`&T`, `&mut T`) and pointers (`*T`, `Ptr<pointee = T>`) require `T` to have a bit‑width of at least 8. Types narrower than one byte (e.g., `Int<3>`, `UInt<4>`) are value‑only types and cannot be the target of a reference or pointer. Bit‑fields in `@packed` structs are accessed through the enclosing struct, not by direct reference.

### Reference Coercion and Read-Only Borrows

By default, Posita does not allow a `&mut T` to be implicitly coerced
to `&T`. An explicit syntax is required to make the loss of mutability
visible to reviewers.

**Explicit Read-Only Borrow: `&ro`**

The `&ro` operator creates a read‑only (immutable) reference from a mutable
one:

```
def takes_shared(x: &Int<32>) -> Int<32> {
    return *x;
}

def main() -> Int<32> {
    set mut val = 42;
    let r: &mut Int<32> = &mut val;
    // Explicitly freeze the mutable reference to obtain an immutable one
    let result = takes_shared(&ro r);
    return result;
}
```

`&ro r` is a compile‑time operation; it produces a `&T` from a `&mut T`
with zero runtime overhead.

During the lifetime of the `&ro` borrow, the original mutable reference
(`r`) is frozen and cannot be used for mutation.

`&ro` is not a keyword; `ro` only has special meaning when immediately
following `&` in a borrow expression.

The borrow checker enforces the following semantics for `&ro`:

- Point‑level liveness (Q1‑B, committee 2026‑08‑05): the freeze lasts
  until the borrow variable's last use — at statement granularity.
  After the last use (even in the same block), the source becomes readable
  and writable again. An unused borrow does not freeze at all.
- Temporary views (a non‑bound `&ro` / `.freeze!()` — no borrow
  variable): the freeze lasts only for the enclosing expression — the
  temporary's "last use" is its own expression, so the source becomes
  writable again at the next statement.

In iterator chains and other fluent interfaces, `&ro` makes the transition
from mutable to read‑only iteration explicit:

```
def process_vec(v: &mut Vector<Int<32>>) -> Vector<Int<32>> {
    return v.iter_mut()
            .map(|x| *x * 2)        // operates on &mut Int<32>
            .collect();
}
```

If a subsequent adapter requires a read‑only view, the compiler will raise
an error asking for an explicit `&ro`:

```
def process_and_filter(v: &mut Vector<Int<32>>) -> Vector<Int<32>> {
    return v.iter_mut()
            .filter(|x| **x > 0)    // Error: cannot pass &mut T where &T is expected
            .map(|x| *x * 2)
            .collect();
}
```

The error can be resolved by inserting `.freeze!()` (the standard‑library
equivalent of `&ro`) at the point where mutability is surrendered:

```
def process_and_filter(v: &mut Vector<Int<32>>) -> Vector<Int<32>> {
    return v.iter_mut()
            .freeze!()              // explicit transition to read‑only view
            .filter(|x| *x > 0)
            .map(|x| *x * 2)
            .collect();
}
```

Note: The method `.freeze!()` is a convenience wrapper around `&ro`
that behaves identically and is preferred in method chains. Both forms
are compiler‑recognised and have zero runtime cost.

**Local Relaxation: `@auto_ro` and `@auto_coerce`**

For modules or functions where the explicitness of `&ro` becomes a
productivity burden, Posita provides opt‑in attributes that restore
implicit coercion within a limited scope.

`@auto_ro` – Allows `&mut T` to be implicitly coerced to `&T`
within the annotated function or module. Implicit coercions do not
freeze the source variable. Code requiring freeze guarantees should
use `&ro` explicitly. Not permitted in `@trusted` or Strict Mode.

```
@auto_ro
def flexible_function(r: &mut Int<32>) -> Int<32> {
    // No &ro required; implicit coercion is enabled
    return takes_shared(r);
}
```

Under the borrow checker, `@auto_ro` implicit coercions register a
temporary read-only loan scoped to the call expression. The source
variable is frozen during the call (not readable, not writable) and
restored to its original borrow state after the call returns. This
is narrower than the explicit `&ro` freeze (which lasts until the
borrow's last use).

`@auto_coerce` – Enables all safe implicit coercions within the
annotated scope (function, module, or file), including `&mut T` → `&T`
and deref coercions (e.g. `&Rc<T>` → `&T`). Not permitted in
`@trusted` functions or Strict Mode.

```
@auto_coerce
mod internal_utils {
    pub def helper(r: &mut Int<32>) -> Int<32> {
        return takes_shared(r); // implicit coercion works
    }
}
```

### Explicit Lifetime Parameters

When the compiler cannot infer lifetimes, you may annotate them explicitly:

```
def process<'a, T, E>(x: &'a mut Data, y: &'a Data) -> &'a mut Result<T, E> { ... }
```

Lifetime annotations are verified by the borrow checker; mismatches cause compile errors. They serve only for disambiguation.

### Reference‑Counted Type

`Rc<T>` provides shared ownership. The compiler verifies that every `clone()` is paired with a corresponding `drop()` along all control‑flow paths, preventing leaks. The net change in reference count across any function boundary can be specified in contracts.

### Graded Modality: `Lease<T, N>`

`Lease<T, N>` (standard library; GrTT's graded modality) packages a
value with exactly `N` rights to use it — the surface form of grades
greater than 1:

- Each use decrements the lease count (const-generic arithmetic,
  resolved at compile time).
- **Conservation is compiler-verified**: a `Lease` that does not reach
  zero before scope exit must be explicitly returned or passed to
  `forget` — the `@linear` discipline generalized from 1 to N.
- `Lease` is a type, so it appears wherever types appear: trait bounds,
  associated types, generic constraints — positions attributes cannot
  reach.

### Strings and Byte Slices

Posita distinguishes between UTF‑8 text and arbitrary bytes:

- `Str`: A built‑in UTF‑8 string slice type. It is an immutable view, guaranteed to contain valid UTF‑8.
- `String`: A standard‑library provided, mutable, owning UTF‑8 string type.
- `[Byte]`: A byte slice that makes no encoding guarantees.

Literals: `"hello"` has type `&Str`, `b"hello"` has type `&[Byte]`. Conversion requires explicit casts.

### Type‑level Default Values

Every type can declare a default value that is automatically assigned to any variable of that type that is not explicitly initialized. This eliminates all "uninitialized variable" bugs.

```
type MyInt = Int<8> with default = 1;
```

The default value must satisfy any type invariants; otherwise the compiler will reject the type definition.

**Semantically Sensitive Defaults and `no_default`:**

For types where a "blank" default is semantically dangerous, you can forbid implicit initialization with `with no_default`:

```
type OwnedFd = exists n: Int<32>
    invariant n >= 0
    with no_default;   // must explicitly initialize
```

Declaring `set fd: OwnedFd;` will then be a compile‑time error. The `with no_default` clause is the only way to forbid implicit initialization; there is no `@no_default` attribute.

GADT enums: If a generic enum contains any variant with a `when` constraint that involves its type parameters, the `with default` clause is prohibited. For non‑generic enums, `when` constraints that involve only global constants do not affect `with default` eligibility. This restriction may be relaxed in the future.

### Construction Validation

A type may specify a validation function that is automatically called after every explicit construction.

```
type Config = struct {
    timeout: UInt<32>,
} with validate = |c: &Config| -> Result<(), Error> {
    if c.timeout > 1000 {
        leave with Error::TimeoutTooLarge;
    }
    Ok(())
};
```

The compiler inserts a call to the `validate` closure immediately after any `Config { ... }` expression, and a compile‑time error is raised if the result is not handled.

### Self‑Referential Types

A type may refer to itself through `Self` inside its definition:

```
type ListNode = struct {
    value: Int<32>,
    next: Option<Ptr<size=UInt<64>, pointee=Self>>,
};
```

### Type Invariants

A type may define an `invariant` clause that all valid instances must satisfy. The compiler verifies or enforces the invariant at every construction point.

```
type NonZeroInt = exists n: Int<32>
    invariant n != 0;
```

The `exists` keyword introduces a name for the value being constrained. This name can be used in the invariant expression and is erased at runtime.

**Implicit invariant propagation:** All functions automatically inherit `requires` that their parameters satisfy type invariants, and `ensures` that their return values satisfy type invariants. This eliminates redundant contract repetition.

### Composite Types

Arrays: `[T; N]` (fixed size), `[T]` (slice, usually behind a reference).

Tuples: `(T1, T2, ...)`

**Empty Tuple (`()`)**: The empty tuple `()` is both a type and a value. It serves as Posita's unit type—a concrete, constructible value that carries no information. It is used as a generic placeholder when a type parameter must be instantiated but no meaningful value is needed (e.g., `Iterator<Item = ()>`). Unlike the `!` (never) type, `()` is a normal value that can be passed, returned, and stored. See also `!` vs `()` in Control Flow.

Structs:

```
type Point = struct {
    x: Int<32>,
    y: Int<32> with default = 0,
}
```

Enums (algebraic):

```
type Option<T> = enum {
    None,
    Some(T),
}
```

An enum can provide a custom error message that is emitted when a `match` does not cover all variants:

```
type State = enum {
    Init,
    Running,
    Stopped,
} with missing_match = "You must handle all three states: Init, Running, and Stopped.";
```

The `@exhaustive` attribute on an enum forces all `match`, `if let`, and `while let` sites to be exhaustive, preventing future variants from being silently ignored.

### Type Alias Impl Trait (TAIT)

A type alias can hide its concrete implementation by using `impl Trait`:

```
pub type MyIter = impl Iterator<Item = UInt<32>>;

pub def make_iter() -> MyIter {
    return 0..10;   // concrete type is Range<UInt<32>>, inferred by compiler
}
```

`MyIter` is an opaque type: external code only knows it implements
`Iterator<Item = UInt<32>>`. The exact type is determined by the defining
module and is not visible externally.

- All defining uses of a TAIT must resolve to the same concrete type
  (single-definition rule).
- TAIT is monomorphized at compile time; there is no dynamic dispatch.
- Contracts on `MyIter` can only reference properties guaranteed by
  the trait (e.g., `codomain.count() == 10`).
- TAIT cannot be used in enum set aliases.

### Enum Set Aliases

Posita supports combining existing enum types into a named alias using the `|` operator. This is particularly useful for error aggregation and state machine simplification.

```
type IoOrParseError = IoError | ParseError;

def read_and_parse() -> Result<Data, IoOrParseError> {
    let content = read_file("data.bin")?;  // may propagate IoError
    let data = parse(content)?;             // may propagate ParseError
    Ok(data)
}
```

**Restriction:** The `|` operator is only permitted in `type` alias declarations. Inline `A | B` in function signatures is not allowed; the combination must be given an explicit name.

**Variant uniqueness:** All variant names across the combined enums must be distinct. If two enums share a variant name (e.g., both define `Timeout`), the compiler reports an error. Disambiguation requires qualifying the variant name with its enum type (e.g., `IoError::Timeout` vs `NetError::Timeout`), which is permitted in `catch` and `match` patterns when the variant name is ambiguous, but not in type declarations.

**Transparency:** The alias is fully transparent to the type system. The compiler expands `IoOrParseError` to the full set of variants at compile time. `catch` branches, `@must_handle` annotations, and `match` expressions can reference individual variants by their original names without qualifying through the alias.

**Recursive sets:** An enum set alias may include other set aliases in its definition (e.g., `type AppError = IoOrParseError | DbError`). The compiler flattens the chain into a single variant set.

**No `impl Trait` or `dyn Trait` in set aliases:** The right‑hand side of
an enum set alias must consist solely of existing enum type names. `impl Trait` (TAIT) and `dyn Trait` are not permitted in set aliases, because
the variant set must be statically known at compile time for `catch`
pattern exhaustiveness and `@must_handle` resolution.

### Generalized Algebraic Data Types (GADTs)

Posita supports Generalized Algebraic Data Types (GADTs) to enable type‑safe embedding of domain‑specific languages and precise encoding of invariants directly into enum variants. A GADT is an enum where individual variants may impose additional constraints on the type parameters of the enum, allowing the compiler to refine types during pattern matching.

#### Motivation

Consider an expression language where the type of an expression is statically known. Without GADTs, a typical encoding requires a runtime type tag and fallible evaluation. With GADTs, the type parameter `T` in `Expr<T>` precisely tracks the expression's result type, making ill‑typed expressions unrepresentable.

```
type Expr<T> = enum {
    Lit(Int<32>) when T == Int<32>,
    IsZero(Expr<Int<32>>) when T == Bool,
    If(Expr<Bool>, Expr<T>, Expr<T>),
}
```

Here, `Lit` is only a valid variant of `Expr<Int<32>>`, `IsZero` only of `Expr<Bool>`, while `If` works for any `T`. Attempting to construct `Lit(42)` where an `Expr<Bool>` is expected is a compile‑time error.

#### Syntax

A GADT variant is declared like a regular enum variant, optionally followed by the keyword `when` and a compile‑time boolean expression. The expression may reference the enum's type parameters and may use equality constraints (`==`) with concrete types.

```
enum_variant ::= variant_name ( [ exists_var_decl ] field_type (, field_type)* )? [ when constraint_expr ]
exists_var_decl ::= 'exists' ident (, ident)* ':'
constraint_expr ::= boolean_expression   // must be evaluable at compile time
```

The `exists` clause introduces existentially quantified type variables scoped over the variant's fields and `when` constraint.

Currently, equality constraints of the form `T == ConcreteType` are supported, where `ConcreteType` may be a fully known type or may involve existentially quantified variables (e.g., `when T == [X]` where `X` is declared by `exists X`). Multiple constraints can be combined with `and`.

```
type KeyValue<K, V> = enum {
    IntKey(Int<32>, String) when K == Int<32> and V == String,
    StrKey(String, Int<32>) when K == String and V == Int<32>,
}
```

The `when` clause is part of the variant's "header" and must appear after the fields and before any trailing comma or closing brace of the enum body. It applies only to the variant it is attached to; multiple variants may carry independent constraints.

#### Semantics

A variant with a `when` clause is only considered a valid constructor for instances of the enum where the type arguments satisfy the constraint. For example, `Expr::Lit` can only produce values of type `Expr<Int<32>>`.

The compiler verifies at every construction site that the inferred or explicitly given type arguments satisfy the variant's constraints. Violation results in a compile‑time error.

Constraints are resolved purely at compile time and do not produce runtime checks. They are completely erased from the generated code.

#### Existential Quantification

Variants may declare existentially quantified type variables using `exists X` in the field list. These variables represent hidden types that are not exposed in the enum's type parameters but are constrained by the `when` clause.

```
type DynExpr<T> = enum {
    IntLit(Int<32>) when T == Int<32>,
    Slice(exists X: &[X]) when T == [X],
    Pair(exists A, B: (A, B)) when T == (A, B),
}
```

For `Slice(exists X: &[X]) when T == [X]`: there exists some type `X` such that the variant holds a `&[X]` and `T == [X]`.

When matching an existentially quantified variant, the compiler introduces fresh, abstract type variables for each `exists` parameter. These variables are kept distinct and cannot be unified with anything outside the branch. The branch body can use them, but they remain opaque except as dictated by the `when` constraint (e.g., from `T == [X]` we know the outer type `T` is a slice of some element type, but we cannot assume anything else about `X`).

The existentially quantified variables are scoped to the match arm and are prevented from leaking into the surrounding context by an occurs‑check at the branch boundary.

**Existential Variable Resolution**

- `when X == ConcreteType` (X directly constrained to a concrete type):
  X is resolved to `ConcreteType` within the branch body.
  X is removed from `GadtContext.facts`.
  Any use of X in the branch body is equivalent to using `ConcreteType`.

- `when T == Expr<X₁, X₂, ...>` (X appears inside a compound type expression):
  `T` is refined to `Expr<X₁, X₂, ...>`.
  `X₁, X₂, ...` remain opaque (still type variables in `GadtContext`).
  The branch body may exploit the structure of `Expr<X₁, X₂, ...>`
  (field access, indexing, passing as a function parameter type),
  but cannot assume the concrete identity of `X₁, X₂, ...`.
  `GadtContext` records `T ≡ Expr<X₁, X₂, ...>`.
  At the end of the branch body, the refinement of `T` and the existential
  variables `X₁, X₂, ...` are popped together.

- `when X == Y` (equality between two existential variables):
  `X` and `Y` are equivalent, both remain opaque.
  `GadtContext` records `X ≡ Y`.
  Within the branch body, any occurrence of `X` can be replaced by `Y` and
  vice versa.
  The equivalence is popped at the end of the branch body.

Existential quantification extends to index variables using the
existing `exists` syntax:

```
type List<T, ghost n: usize> = enum {
    Nil                                       when n == 0,
    Cons(exists m: usize, T, List<T, m>)      when n == m + 1,
}
```

Matching `Cons` on a `List<T, n>` introduces the existential `m`,
refines `n == m + 1` within the branch (both in the type system and
the SMT context), and the nested-refinement rules (equality
propagation, or-pattern intersection, unreachable alternatives) apply
unchanged to index equalities.

#### Pattern Matching and Type Refinement

When a GADT value is examined via `match`, `if let`, or `while let`, the compiler uses the variant's `when` constraints to refine the types in the corresponding branch. This enables writing type‑safe operations without redundant casts.

```
def eval<T>(e: Expr<T>) -> T {
    match e {
        Lit(n) => return n,                         // T refined to Int<32>, n: Int<32>
        IsZero(inner) => return eval(inner) == 0,  // T refined to Bool, inner: Expr<Int<32>>
        If(cond, then_expr, else_expr) => {
            if eval(cond) { return eval(then_expr) } else { return eval(else_expr) }
        },
    }
}
```

In the `Lit` branch, the compiler learns `T == Int<32>` and therefore the return type `T` can be satisfied by returning an `Int<32>`. Similarly, in the `IsZero` branch, `T == Bool` is known, and the recursive call `eval(inner)` returns `Int<32>`, allowing the comparison `== 0`. The `If` branch imposes no additional constraints, so `T` remains abstract.

The refinement applies to the entire branch body, including any variable bindings from patterns. For instance, in `Lit(n)`, the variable `n` has type `Int<32>`, as declared, and the type parameter `T` of the whole `match` expression is locally unified with `Int<32>`. This unification is consistent with the fact that `e` in that branch must be of type `Expr<Int<32>>`.

Refinement is also available in `if let` and `while let` expressions. For GADT enums, the compiler infers the same type equalities within the guarded block.

For existentially quantified variants, the compiler introduces fresh type variables for the existential parameters and records the equalities from the `when` clause. These fresh variables are available in the branch but are prevented from escaping.

#### Nested GADT Refinement

Type refinements introduced by a GADT pattern are propagated throughout the
branch body, including into nested patterns. The compiler maintains an
equality context during pattern traversal using the following deterministic
rules:

- Constructor patterns automatically introduce all equalities from
  the constructor's `when` clause into the current context.
- Equalities propagate inward to sub‑patterns, constraining their
  type parameters.
- Equalities propagate outward to the branch body, refining return
  types and variable bindings.
- Conflicting equalities (e.g., an outer context requires
  `T == Int<32>` but an inner constructor requires `T == Bool`) are
  compile‑time errors, not silently ignored.
- Wildcards (`_`) and variable bindings do not generate new
  equalities, but are constrained by the existing context.
- Or‑patterns (`|`) refine the context only when all alternatives
  produce identical equalities for every constrained type parameter:
  - Consistent constraints (e.g., `Lit(_) | Neg(_)` both require
    `T == Int<32>`) — the equality propagates into the branch body.
  - Conflicting constraints (e.g., `Add(_, _) | Eq(_, _)` where
    `Add` requires `T == Int<32>` and `Eq` requires `T == Bool`) —
    a compile‑time error is raised.
  - Partial constraints (e.g., `Lit(_) | If(_, _, _)` where `Lit`
    requires `T == Int<32>` but `If` imposes no constraint on `T`) —
    no refinement is propagated; `T` remains abstract in the
    branch body.

This conservative rule ensures that the type of every expression in
an OR‑pattern branch is statically known without relying on implicit
priority between alternatives. Variable bindings within OR‑patterns
are still subject to the usual compatibility check, which will
naturally reject incompatible payload types.

Unreachable alternatives within an or‑pattern (variants statically
proven impossible for the scrutinee type) are ignored for the purpose
of equality intersection. A compile‑time warning is emitted for each
unreachable alternative. The intersection is computed solely over
the reachable alternatives.

These rules are purely mechanical—the compiler performs deterministic
equality propagation without constraint solving, backtracking, or
implicit derivation.

**Interaction with trait resolution:** GADT refinements are visible to trait
obligation resolution within the branch body. Obligations are resolved eagerly
at the point of use, using the current GADT equality context. If the context
does not uniquely determine a trait instance, a compile‑time error is raised
immediately. Obligations are never deferred past the end of the branch body.

Example:

```
type Expr<T> = enum {
    Lit(Int<32>) when T == Int<32>,
    Neg(Expr<Int<32>>) when T == Int<32>,
    Add(Expr<Int<32>>, Expr<Int<32>>) when T == Int<32>,
    Eq(Expr<Int<32>>, Expr<Int<32>>) when T == Bool,
    If(Expr<Bool>, Expr<T>, Expr<T>),
}

def eval<T>(e: Expr<T>) -> T {
    return match e {
        Lit(n) => n,
        Neg(x) => -eval(x),
        Add(Lit(a), Lit(b)) => a + b,   // nested refinement works
        Eq(x, y) => eval(x) == eval(y),
        If(c, t, f) => if eval(c) { return eval(t) } else { return eval(f) },
    }
}
```

#### Exhaustiveness Checking

GADT constraints interact with exhaustiveness checking. If a particular variant is impossible for a given instantiation of the enum's type parameters, the compiler may omit it from exhaustiveness requirements.

Consider `Expr<Bool>`. The `Lit` variant requires `T == Int<32>`, which contradicts `T == Bool`; therefore, `Lit` is unreachable for `Expr<Bool>`. When matching on an `Expr<Bool>`, the compiler does not require a branch for `Lit`. Similarly, for `Expr<Int<32>>`, the `IsZero` variant is unreachable.

This dead‑variant elimination works automatically. When the `@exhaustive` attribute is applied to a GADT enum, the compiler enforces that all reachable variants are covered, and no warning is emitted for omitting unreachable ones. This eliminates the need for error‑prone wildcard `_` branches that might mask forgotten cases.

#### Interaction with `with default`

A type‑level default value declared with `with default` must be valid for all possible instances of the type. For a generic GADT enum whose variants carry `when` constraints involving the type parameters, a single default value cannot in general be well‑typed under all possible constraints.

Therefore, if a generic enum contains any variant with a `when` clause that involves its type parameters, the `with default` clause is prohibited. For non‑generic enums, `when` constraints that involve only global constants do not affect `with default` eligibility. This restriction may be relaxed in the future for monomorphic instances.

```
// Error: generic GADT enum cannot have a default value
type Expr<T> = enum {
    Lit(Int<32>) when T == Int<32>,
    ...
} with default = Lit(0);  // compile-time error
```

#### Construction and Validation

Constructors of GADT variants are subject to the same type‑checking and contract verification as any other constructor. If a variant has an invariant or a `validate` function, those apply after the `when` constraints have been satisfied.

Construction verification timing: For enum variants without `when`
constraints (e.g., `Option::Some(1)`), the compiler does not require
concrete type arguments at the construction site. Type inference proceeds
normally: (1) call-site constraints propagate back via suspended
constraints; (2) defaulting applies to unresolved variables (integer
literals → `Int<32>`); (3) `when` constraints are verified against the
fully resolved types. E060 is raised only if a `when` constraint fails
after solve + defaulting, not if type variables are merely unresolved at
the construction site.

#### Limitations and Future Directions

- Equality constraints of the form `T == Type` are supported, where `Type`
  may be a concrete type (e.g., `Int<32>`, `[Byte]`) or a type involving
  existentially quantified variables introduced with `exists` inside the
  variant (e.g., `when T == [X]` where `X` is declared as `exists X`).
- Constraints involving type parameters from the enum header on the
  right‑hand side (e.g., `T == [U]` where `U` is another enum type
  parameter) are not yet allowed.
- The constraint expressions are evaluated at compile time and cannot depend on runtime values.
- Type-equality constraints (`when T == Int<32>`) are entirely a type-system feature and do not interact with the SMT solver. Index-equality constraints over ghost value indices (`when n == m + 1`) are propagated into the SMT context of the match branch, where they combine with contracts and invariants.
- Default values for generic GADT enums are currently prohibited; a future version may allow default values for specific monomorphic instances using a syntax like `default for Expr<Bool> = ...` (planned).

#### Examples

A type‑safe abstract syntax tree for a simple language:

```
type Expr<T> = enum {
    Lit(Int<32>) when T == Int<32>,
    Neg(Expr<Int<32>>) when T == Int<32>,
    Add(Expr<Int<32>>, Expr<Int<32>>) when T == Int<32>,
    Eq(Expr<Int<32>>, Expr<Int<32>>) when T == Bool,
    If(Expr<Bool>, Expr<T>, Expr<T>),
}

def eval<T>(e: Expr<T>) -> T {
    return match e {
        Lit(n) => n,
        Neg(x) => -eval(x),
        Add(a, b) => eval(a) + eval(b),
        Eq(a, b) => eval(a) == eval(b),
        If(c, t, f) => if eval(c) { return eval(t) } else { return eval(f) },
    }
}
```

This demonstrates how GADTs eliminate the possibility of constructing ill‑typed ASTs and enable safe evaluation without runtime type checks.

Existential GADT example:

```
type DynExpr<T> = enum {
    IntLit(Int<32>) when T == Int<32>,
    Slice(exists X: &[X]) when T == [X],
    Pair(exists A, B: (A, B)) when T == (A, B),
}

def slice_length<T>(e: DynExpr<T>) -> usize
    requires T == [X] for some X   // conceptual requirement
{
    return match e {
        Slice(s) => s'len,   // s: &[X], length available
        _ => panic("not a slice"),
    }
}
```

In the `Slice` branch, the compiler knows `T == [X]` for the existential `X`, and `s` has type `&[X]`. The length operation is valid, and `X` remains abstract but does not need to be known further.

### Layout Control Attributes

Fine‑grained control for hardware registers and protocols:

- **Packing**: `@packed` removes all padding between fields, ensuring minimal memory footprint. Restriction: `@packed` cannot be applied to structs that contain reference types (`&T`, `&mut T`, `&[T]`, `&mut [T]`). If you need a compact layout with indirection, use `Ptr` instead.
- **Endianness:** `@endian(little)` or `@endian(big)`.
- **Bit order:** `@bit_order(lsb_to_msb)` or `@bit_order(msb_to_lsb)`. This attribute only applies to bit‑fields inside `@packed` structs.
- **Alignment:** `@align(N)` overrides natural alignment. `N` must be a power of two.
- **Padding:** `@pad(byte_count)` inserts explicit padding bytes.
- **C ABI compatibility:** `@layout(C)` forces the struct to follow the target platform's C ABI rules for field alignment, padding, and trailing size. This is required for any struct passed across an `extern "C"` boundary.
- **Newtype transparency:** `@transparent` guarantees that a single‑field struct has exactly the same layout as its field's type. This is essential for newtype wrappers that must be ABI‑compatible with the inner type.

Example:

```
@packed @endian(big) @bit_order(msb_to_lsb)
type IPv4Header = struct {
    version: UInt<4>,
    ihl:     UInt<4>,
    dscp:    UInt<6>,
    ecn:     UInt<2>,
};
```

Posita natively supports bit‑fields (`UInt<4>`, etc.) inside `@packed` structs. The `@bit_order` attribute controls the order in which bit‑fields fill bytes.

### Layout Aliases (`layout` keyword)

To avoid repetitive attribute lists, Posita allows defining named combinations of the above layout attributes using the `layout` keyword. A layout alias is a pure, declaration‑time shorthand for a set of built‑in layout attributes. It introduces no new semantics and cannot contain any executable logic.

**Syntax:**

```
layout Name {
    attribute1,
    attribute2,
    ...;
}
```

The body consists of a comma‑separated list of built‑in layout attributes (`packed`, `little_endian`, `big_endian`, `bit_order(lsb_to_msb)`, `bit_order(msb_to_lsb)`, `align(N)`, `pad(N)`, `C`, `transparent`). The list is terminated by a semicolon. The order of attributes is immaterial; the compiler normalizes their application.

**Restrictions:**

- A layout alias may only reference built‑in attributes; it cannot contain expressions, conditionals, loops, or calls to any function.
- A layout alias cannot reference other layout aliases (no nesting). This guarantees that any alias can be trivially inlined to its constituent built‑in attributes.
- The `@layout` attribute applied to a type accepts either the built‑in identifier `C` or a previously defined layout alias name.

Example:

```
layout MmioLayout {
    packed,
    little_endian,
    bit_order(lsb_to_msb);
}

@layout(MmioLayout)
type UartCtrl = struct {
    ctrl: UInt<8>,
    data: UInt<32>,
};

// Equivalent explicit form:
// @packed @endian(little) @bit_order(lsb_to_msb)
// type UartCtrl = struct { ... };
```

**Usage with `@layout(C)`:**

The `@layout` attribute on a type definition is used both for referencing a defined alias and for the special built‑in `C` layout. The compiler distinguishes based on whether the argument is the identifier `C` (a reserved built‑in) or a user‑defined layout name.

```
@layout(C)          // Built‑in C ABI layout
type CCompat = struct { x: Int<32>, y: Int<64> };

layout CompactIo {
    packed,
    little_endian;
}

@layout(CompactIo)  // User‑defined layout alias
type IoReg = struct { flags: UInt<8>, addr: UInt<32> };
```

**Auditability:** The compiler expands all layout aliases at their use sites. Tools like `capsa audit` and IDE hover information display the fully expanded attribute set, ensuring that the effective layout is always visible at the type definition.

## Language Attributes

The following attributes are not layout‑specific but affect language semantics or tooling behavior:

| Attribute | Applies to | Description |
|---|---|---|
| `@no_copy` | Type | Prevents automatic `Copy` derivation (sugar for an empty `Drop`). |
| `@deprecated("msg")` | Function, Type | Marks an API as deprecated |
| `@experimental` | Function, Type | Marks a feature as experimental |
| `@inline` | Function | Forces inlining at call sites; compile error if not possible |
| `@noinline` | Function | Prevents inlining of the function |
| `@auto_deref` | impl Deref | Allows method‑call receiver auto‑dereferencing for this `Deref` implementation. Without this attribute, a `Deref` impl requires explicit `(*x).method()` syntax for method calls through the wrapper type. `&T` / `&mut T` are exempt and always auto‑dereference. |
| `@auto_coerce` | Function, Module, File | Enables all safe implicit coercions within the annotated scope (function, module, or file), including `&mut T` → `&T` and deref coercions. Not permitted in `@trusted` functions or Strict Mode. |
| `@auto_ro` | Function, Module | Allows implicit `&mut T` → `&T` coercion within the annotated function or module. Implicit coercions do not freeze the source variable. Not permitted in `@trusted` functions or Strict Mode. |
| `@must_use` | Function, Type | Compiler warns if the return value is silently discarded. `Result` and `Option` are implicitly `@must_use`. |
| `@must_handle(Variant1, ...)` | Function (returning `Result`) | Compiler warns if the caller does not explicitly match or catch the listed error variants. Variant names are resolved against the error type `E` of `Result<_, E>`. If a variant name is ambiguous in the current scope, the compiler emits an error and requires explicit qualification using `EnumName::Variant`. |
| `@tailrec` | Function | Verifies that all recursive calls are in tail position and enforces tail‑call optimization; compile error if not possible |
| `@lemma` | Function (lemma provider) | Marks a `comptime` proof helper that returns assertions for the SMT solver |
| `@apply_lemma(...)` | Function (verification target) | Applies a lemma (by name) to supply the SMT solver with additional assertions during verification of the annotated function |
| `@trusted` | Function | Establishes a trust boundary for operations the compiler cannot fully verify, including `unsafe` blocks, raw pointer manipulation, `extern "C"` calls, and dynamic dispatch via `dyn Trait` (in strict mode). Requires `requires`/`ensures` contracts. |
| `@link_proof(path, hash)` | Function (`@trusted`) | References an external formal proof (e.g., Coq/ATS script) and records its SHA‑256 hash, supplementing the verification of `@trusted` code. In strict mode, every `@trusted` function must be accompanied by either `@link_proof` or at least one `@comptime_test` exercising its safety contract; otherwise compilation fails. `capsa audit` verifies the proof file exists and matches the hash. |
| `@pure` | Function | No side effects; result depends only on arguments |
| `@io(read)` | Function | May perform input operations |
| `@io(write)` | Function | May perform output operations |
| `@io` | Function | May perform any I/O (equivalent to `@io(read, write)`) |
| `@alloc` | Function | May perform dynamic memory allocation |
| `@no_alloc` | Function | Guarantees no dynamic allocation. This implies `@no_alloc_error`; redundant declaration of both is allowed but not required. |
| `@no_alloc_error` | Function | Guarantees no allocation on error paths, including `From` conversions. May coexist with `@alloc` (normal paths may allocate while error paths must not). |
| `@no_panic` | Function | Guarantees the function never panics; compiler verifies no `trap`, bounds checks, or calls to non-`@no_panic` functions. Verification failure is a compile‑time error in strict mode; in non‑strict mode, the compiler emits a warning and may instrument unproven checks with a runtime panic guard. |
| `@runtime_check` | Function | Defers contract checking to runtime, even if arguments are compile‑time known. Only allowed in non‑strict mode. |
| `@cfg(condition)` | Module, Function | Conditional compilation with `all`, `any`, `not` combinators. The condition may refer to target platform, features, etc. Only paths that compile under the configuration are permitted in strict mode. |
| `@hint(assertion)` | Function, Loop | Provides a hint to the SMT solver to guide proof search. Hints must be accompanied by a meta‑contract that proves the hint itself is valid. Example: `@hint(forall i in 0..len: arr[i] > 0)` asserts all elements are positive within the given function or loop. |
| `@exhaustive` | Enum | Requires all `match`, `if let`, and `while let` on the enum to be exhaustive. |
| `@debug_info` | Function, Module | Controls which symbols are emitted into debug information. Supports minimal exposure for safety‑critical deployments. |
| `@audit_log` | Function | Marks a function whose runtime contract violations must be written to an immutable audit log. The storage backend is defined by the standard library; tamper‑evident integrity (e.g., hash chains) is strongly recommended. |
| `@interrupt(irq, priority?)` | Function | Marks an interrupt handler. The compiler enforces that the function satisfies the constraints of `@no_alloc`, `@no_panic`, and return type `!`; violations are compile‑time errors. Redundant explicit `@no_alloc` or `@no_panic` annotations are allowed and produce no warning. See "Interrupts" for full constraints. |
| `@ieee_contracts` | Function | Interprets all floating‑point `requires` and `ensures` clauses on this function under IEEE 754 semantics instead of the default mathematical real domain. This attribute is not inherited by callees. |
| `@diverges` | Function | Declares that this function never returns normally. The function's return type may be any `T` (not just `!`), but all reachable paths in the body must diverge (e.g., `loop {}`, hardware halt). The compiler verifies divergence in strict mode. `@diverges` is incompatible with `panic` in the function body (divergence must be deterministic). Compatible with `@no_panic` (non‑panic divergence) and `@trusted`. Not compatible with `@runtime_check`. |
| `@consume(s)` / `@consume(≤s)` | Parameter | Declares the subject grade as part of the function type; propagated to callers by graded application. Affine params: ≤ 1. See §Graded Function Types. |
| `@grades(name)` | Module | Selects the usage semiring for the module. See §Usage Semirings. |

**Implicit relationships among attributes:**

- `@no_alloc` implies `@no_alloc_error`; a function that never allocates trivially satisfies the no‑allocation‑on‑error constraint.
- `@no_alloc_error` may coexist with `@alloc`; normal paths may allocate while error paths must not.
- Grades are compile-time only — compatible with `@pure` and Strict Mode; `@runtime_check` never applies to grades.

**Attribute compatibility and precedence:** When multiple attributes are combined, the compiler follows a strict ordering:

1. `@cfg` is evaluated first (determines existence of the item).
2. Effect annotations (`@pure`, `@io`, `@alloc`, `@no_alloc`, `@no_alloc_error`, `@no_panic`, `@diverges`) are checked for consistency.
3. Contract‑related attributes (`@trusted`, `@runtime_check`, `@link_proof`, `@lemma`, `@ieee_contracts`) are processed.
4. Code‑generation attributes (`@inline`, `@noinline`, `@tailrec`, `@auto_deref`) are applied last.

**Incompatible combinations rejected at compile time:**

- `@pure` + `@runtime_check`
- `@pure` + `@io`
- `@runtime_check` + `@lemma`
- `@trusted` + `@runtime_check`
- `@no_panic` + `@runtime_check`
- `@interrupt` + `@alloc`
- `@interrupt` + `@io`
- `@diverges` + `@runtime_check` (divergent functions do not return, runtime contract checks are meaningless)
- `@diverges` with a function body containing a reachable `panic` call

**Compatible combinations explicitly confirmed:**

- `@runtime_check` + `@ieee_contracts` (runtime checks use IEEE 754 semantics)
- `@pure` + `@ieee_contracts` (only changes contract interpretation domain, not side effects)
- `@no_alloc_error` + `@alloc` (normal paths may allocate, error paths must not)
- `@diverges` + `@no_panic` (non‑panic divergence such as infinite loops or hardware halts)
- `@diverges` + `@trusted` (divergence may depend on external guarantees)
- `@consume` + `@pure` (grades are compile-time analysis, orthogonal to side effects)
- `@consume` + Strict Mode (grade checking is fully static)

## Type Attributes

Access compile‑time properties using `'`:

- `x'len` – length of array/slice
- `x'size` – bit width
- `x'align` – alignment
- `x'first`, `x'last` – first/last index (for arrays)
- `T'default` – the default value of type `T` (usable in `comptime`)

### Compile‑Time Layout Reflection

The built‑in function `layout_of!(T)` returns a compile‑time `LayoutDescriptor` for type `T`, describing the exact size, alignment, and field offsets. This is essential for `comptime` code that verifies or generates hardware‑specific memory layouts.

```
comptime {
    set desc = layout_of!(IPv4Header);
    // desc.size, desc.align, desc.fields[0].offset, etc.
}
```

`layout_of!` is a `comptime`‑only operation and thus requires `!`.

## Type Inference and Capture

`set a = 42;` infers `a` as `Int<32>` (default integer width).

Type capture:

```
set auto<T> = expression;          // capture type only
set auto<T, N> = expression;       // capture type and compile‑time constant
set auto<T, N, L> = expression;    // multiple captures (types and/or constants)
```

Binds the compile‑time entities (types or compile‑time constant values) of `expression` to the names inside the angle brackets, making them available for reflection, assertion, or type factory usage in subsequent `comptime` blocks. Capture names are immutable and scoped to the enclosing block. The comma‑separated list follows the same syntax as generic type parameters. For a concrete usage example, see `make_employee_report` in the Complete Example section.

## Ghost Variables

Ghost variables are declared with the `ghost` keyword and exist only at compile time. They participate in contracts and invariants but are completely erased from runtime code. Their scope follows normal block scoping; they cannot affect runtime control flow (e.g., `if ghost_var` is illegal outside contracts).

Ghost variables exist for compile‑time proof, not for compile‑time computation. Unlike `comptime` variables, which are temporaries within a `comptime` block and vanish after its execution, ghost variables persist throughout the entire verification of a function. They carry proof‑relevant state that is never materialized at runtime, allowing the SMT solver to reason about invariants that span multiple points in the control flow.

Ghost variables follow the same mutability rules as regular variables. `ghost set mut x = false;` declares a mutable ghost variable that can be reassigned in any statement, but these mutations are erased from the final binary.

```
def get_adult_names(users: &[User]) -> Vector<String>
    requires users'len > 0
    requires exists user in users: user.age >= 18
    ensures codomain'len >= 1
{
    set mut names = Vector<String>::new();
    ghost set mut found_adult: Bool = false;

    for i in 0..users'len
        invariant if found_adult { names'len >= 1 } else { true }
        invariant (exists u in users[0..i]: u.age >= 18) implies found_adult
    {
        set user = users[i];
        if user.age >= 18 {
            names.push(user.name.clone());
            found_adult = true;
        }
    }
    return names;
}
```

Note: When iterating over arrays/slices, prefer the explicit index form (`for i in 0..arr'len`) if you need to refer to the index in contracts or invariants.

## Safety Guarantees, Effect Annotations, and `unsafe`

### The "No Undefined Behavior" Promise

Posita eliminates language‑level undefined behavior in safe code: no uninitialized variables, no integer overflow UB, no null pointer dereference, no out‑of‑bounds access, no data races, no invalid enum values, no misaligned or invalid pointer casts.

### Effect Annotations

Functions may be annotated with fine‑grained effect markers to describe their side effects. These annotations are checked by the compiler and serve as auditable documentation for reviewers.

- `@pure`: The function has no side effects and its result depends only on its arguments.
- `@io(read)`: The function may perform input operations (reading files, network, etc.).
- `@io(write)`: The function may perform output operations.
- `@io(read, write)` or simply `@io`: The function may perform any I/O.

**I/O effect aliases:** Projects can define named I/O effect aliases to
group related resource tags, following the same pattern as `layout` aliases:

```
io FileIO {
    read: file;
    write: file;
}

io NetIO {
    read: net;
    write: net;
}

io ConsoleIO {
    write: console;
}

// Usage: combine aliases with commas (union semantics)
@io(FileIO, ConsoleIO)
def log_to_console(msg: &Str) { ... }

// The underlying tag syntax is also available for inline use:
@io(read: file | net, write: console)
def fetch_and_display() { ... }
```

An `io` alias block lists `read` and/or `write` directions, each followed
by a semicolon-separated list of resource tags.

Aliases are combined with commas in `@io(...)` annotations, producing the
union of their effects. This is consistent with `@io(read, write)` syntax
for combining directions.

The inline tag syntax (`read: file | net`) is syntactic sugar for an
anonymous alias and is fully equivalent.

Unqualified `@io(read)` remains valid and grants access to all I/O sources.

- `@alloc`: The function may perform dynamic memory allocation. May coexist with `@no_alloc_error`.
- `@no_alloc`: The function guarantees no dynamic allocation. This implies `@no_alloc_error`.
- `@no_alloc_error`: The function guarantees no allocation on any error path. All `From` conversions reachable via `?` in error paths must also be `@no_alloc`. May coexist with `@alloc`.
- `@no_panic`: The function never panics. The compiler statically verifies the absence of overflow traps, bounds‑check failures, or calls to non‑`@no_panic` functions. Verification failure is a compile‑time error in strict mode; in non‑strict mode, the compiler emits a warning and may instrument unproven checks with a runtime panic guard. Callers are not required to be `@no_panic` themselves; the guarantee is internal to the function body.
- `@diverges`: The function never returns normally. The return type may be any `T`, but all reachable paths must diverge deterministically (e.g., `loop {}`, hardware halt). Functions marked `@diverges` must not contain reachable `panic` calls. Compatible with `@no_panic` (non‑panic divergence) and `@trusted`.
- `@trusted`: The function contains operations the compiler cannot fully verify (`unsafe` blocks, raw pointer manipulation, `extern "C"` calls, or dynamic dispatch via `dyn Trait` in strict mode) and establishes a trust boundary; it must carry `requires`/`ensures` contracts.
- `@audit_log`: The function must write any runtime contract violation to an immutable audit log. The log storage backend is provided by the standard library; tamper‑evident integrity (e.g., hash chains) is strongly recommended.
- `@ieee_contracts`: Interprets all floating‑point contracts (both `requires` and `ensures`) on this function under IEEE 754 semantics rather than the mathematical real domain. This attribute is scoped to the annotated function only and does not affect the contract semantics of any callees.

In strict mode the compiler verifies that effect annotations are consistent across call chains. A function calling an `@io(write)` function must itself be annotated at least `@io(write)`.

**Closure effect inference:** A closure's effect annotation is the union of the effects of its captured variables and its body. The compiler automatically derives this union and annotates the closure type.

### The `unsafe` Keyword

`unsafe` allows escape hatches (inline assembly, raw pointer manipulation, C FFI). All `unsafe` operations must be encapsulated in an `@trusted` function with `requires`/`ensures` contracts. The compiler trusts these contracts as axioms, and they are auditable via `capsa audit`.

In Strict Mode, `unsafe` blocks are completely forbidden, guaranteeing UB‑free by construction.

### `@trusted` Nesting

`@trusted` functions may call other `@trusted` functions, but the entire call chain is tracked by `capsa audit`. In strict mode, every `@trusted` function must be accompanied by either `@link_proof` referencing an external formal proof, or at least one `@comptime_test` exercising its safety contract. If neither is present, compilation fails. If neither is possible, the dependency must be explicitly marked as `trusted = true` in `posita.toml`, indicating that it has been manually audited.

### Dynamic Dispatch in Strict Mode

In strict mode, the construction and invocation of `dyn Trait` objects require a `@trusted` context. Because the concrete implementation type is resolved at runtime, the compiler cannot statically verify contracts or perform exhaustive control‑flow analysis through the dispatch site. By confining dynamic dispatch to `@trusted` functions, the trust boundary is made explicit, and the programmer must provide `requires`/`ensures` contracts that cover all possible implementations.

Outside of `@trusted` code, `dyn Trait` is rejected in strict mode. In non‑strict mode, dynamic dispatch is permitted but flagged during audit for mandatory review.

### `extern "C"` ABI Rules

All `extern "C"` function declarations are inherently unsafe and can only be called inside `unsafe` blocks or `@trusted` functions. When a reference type (`&T`, `&[T]`, `&mut T`, `&mut [T]`) appears in the signature of an `extern "C"` function, the compiler automatically converts it to the corresponding raw C pointer for the call:

- `&T` and `&[T]` become `*const T`
- `&mut T` and `&mut [T]` become `*mut T`

The length component of slices is discarded. The caller is responsible for ensuring that the data satisfies any additional C‑side requirements (e.g., null termination for `puts`). This conversion is deterministic and does not constitute an implicit runtime check.

## Traits and Implementations

Posita uses traits to define shared behavior, similar to Rust traits or Haskell typeclasses. Traits may contain function signatures, associated types (with optional defaults), and default method implementations.

### Defining a Trait

```
trait Add<Rhs = Self> {
    type Output;
    def add(self: &Self, rhs: &Rhs) -> Self::Output;
}

trait Iterator {
    type Item;
    type Distance = usize;  // associated type with default value
    def next(&mut self) -> Option<Self::Item>;
}

trait Default {
    def default() -> Self;
}

trait Drop {
    def drop(&mut self);
}

trait Copy: Clone { }  // marker trait, no methods

trait Clone {
    def clone(&self) -> Self;
}

trait Deref {
    type Target;
    def deref(&self) -> &Self::Target;
}

trait Fn<Args...> {
    type Output;
    def call(&self, args: Args) -> Self::Output;
}
```

Within a trait definition, `type Name = DefaultType;` declares an associated type with a default value. Implementations may override the default or use it as‑is.

### Method‑Call Auto‑Dereferencing

Posita provides a controlled form of auto‑dereferencing for method calls. A method call `receiver.method(args)` automatically inserts dereference steps under the following conditions:

- `&T` / `&mut T`: These built‑in reference types always auto‑dereference to `T`. The call `r.method()` where `r: &Point` is equivalent to `(*r).method()`.
- `@auto_deref` on `Deref` implementations: A type that implements `Deref` may annotate its `impl` block with `@auto_deref`. This grants method‑call auto‑dereferencing through that specific `Deref` implementation.

A `Deref` implementation without `@auto_deref` does not participate in auto‑dereferencing, and method calls through the wrapper type require explicit `(*x).method()` syntax.

```
// Standard library Rc<T> – auto‑deref is enabled by the type author
@auto_deref
impl<T> Deref for Rc<T> {
    type Target = T;
    def deref(&self) -> &T { /* … */ }
}

// Opaque wrapper – auto‑deref is intentionally disabled
impl<T> Deref for OpaquePtr<T> {
    type Target = T;
    // no @auto_deref → method calls on OpaquePtr<T> do NOT auto‑deref
    def deref(&self) -> &T { /* … */ }
}
```

This design keeps auto‑dereferencing explicit at the point where it is granted—the `Deref` implementation—while providing ergonomic method calls for smart pointers and other wrapper types that are explicitly designed to be transparent.

### Implementing a Trait

```
impl Add for Int<32> {
    type Output = Int<32>;
    def add(self: &Int<32>, rhs: &Int<32>) -> Int<32> { /* … */ }
}

impl Drop for UniqueToken {
    def drop(&mut self) { /* release token */ }
}
```

### Automatic Clone for Copy Types

When `Copy` is automatically derived (or manually implemented), the compiler also automatically derives `Clone` with `def clone(&self) -> Self { return *self; }`.

### Associated Types and Projections

Associated types are accessed via the `::` operator: `T::Output`. In `where` clauses they are used to constrain relationships between types.

### Generic Associated Types (GAT)

An associated type in a trait may itself be parameterized by compile‑time
lifetimes or values. This enables the expression of type constructors at
the trait level.

#### Declaration

```
trait StreamingIterator {
    type Item<'a> where Self: 'a;
    def next<'a>(&'a mut self) -> Option<Self::Item<'a>>;
}
```

A GAT parameter list `<'a>` (lifetimes) or `<const N: usize>` (compile‑time
constants) follows the associated type name.

An optional `where` clause on the GAT constrains the relationship between the
trait's implementing type and the GAT's parameters. The constraint `Self: 'a`
is typical when the GAT borrows from `self`.

Multiple GAT parameters may appear, separated by commas, e.g.:
`type Map<'a, const N: usize> where Self: 'a;`

#### Implementation

```
struct BytesStream { data: &[Byte], pos: usize }

impl StreamingIterator for BytesStream {
    type Item<'a> = &'a Byte where Self: 'a;
    def next<'a>(&'a mut self) -> Option<&'a Byte> {
        if self.pos < self.data'len {
            let b = &self.data[self.pos];
            self.pos += 1;
            return Some(b);
        } else {
            return None;
        }
    }
}
```

Each GAT declared in the trait must be given a concrete type in every `impl`.
The concrete type may also be generic over the same (or fewer) parameters,
and must satisfy any `where` constraints declared in the trait.

#### Interaction with HRTB

GATs naturally combine with higher‑ranked trait bounds to express constraints
that hold for all possible lifetimes:

```
def drain_all<I>(iter: I) -> Vector<I::Item<'_>>
    where I: for<'a> StreamingIterator<Item<'a> = &'a Int<32>>
{
    // ...
}
```

Here `for<'a>` quantifies over the GAT's lifetime parameter, allowing the
function to accept any `StreamingIterator` whose items are references.

#### Compile‑time Value Parameters

GATs may also be parameterized by compile‑time constants, using the same
`const` syntax as regular generics:

```
trait FixedBuffer {
    type Slice<const N: usize> where N <= Self::CAPACITY;
    const CAPACITY: usize;
}
```

This enables type‑level functions over sizes, e.g., returning a fixed‑size
array type that depends on a const parameter.

#### Restrictions

- GAT parameters are not inferred from usage; they must be explicitly
  provided at the use site (e.g., `Self::Item<'a>`).
- A GAT cannot introduce new type parameters; only lifetime or const parameters
  are permitted.
- The `where` clause on a GAT may only constrain the relationship between the
  implementing type and the GAT's parameters; it cannot impose arbitrary trait
  bounds on external types.

### Higher-Ranked Trait Bounds (HRTB)

In `where` clauses, a trait bound can quantify over all possible lifetimes
using the `for<...>` syntax:

```
def apply_to_refs<F>(f: F, x: &Int<32>) -> &Int<32>
    where F: for<'a> Fn(&'a Int<32>) -> &'a Int<32>
{
    return f(x);
}
```

- `for<'a>` introduces one or more lifetime parameters scoped over the
  subsequent trait bound.
- The bound `F: for<'a> Fn(&'a T) -> &'a T` means: for any lifetime `'a`,
  `F` implements the `Fn` trait with that lifetime.
- HRTB can also appear in `constraint` blocks and `@lemma` contracts.
- The compiler translates `for`-quantified lifetimes into universally
  quantified variables in the SMT solver when verifying contracts.

### Operator Desugaring

User‑defined operator overloading is achieved by implementing the corresponding built‑in trait. The compiler desugars operators according to the following table:

| Expression | Desugars to | Trait |
|---|---|---|
| `a + b` | `Add::add(&a, &b)` | `Add` |
| `a - b` | `Sub::sub(&a, &b)` | `Sub` |
| `a * b` | `Mul::mul(&a, &b)` | `Mul` |
| `a / b` | `Div::div(&a, &b)` | `Div` |
| `a % b` | `Rem::rem(&a, &b)` | `Rem` |
| `-a` | `Neg::neg(&a)` | `Neg` |
| `a == b` | `Eq::eq(&a, &b)` | `Eq` |
| `a != b` | `not(Eq::eq(&a, &b))` | `Eq` |
| `a < b` | `Ord::lt(&a, &b)` | `Ord` |
| `a > b` | `Ord::gt(&a, &b)` | `Ord` |
| `a <= b` | `Ord::le(&a, &b)` | `Ord` |
| `a >= b` | `Ord::ge(&a, &b)` | `Ord` |
| `a[i]` | `Index::index(&a, i)` | `Index` |
| `a[i] = v` | `IndexMut::index_mut(&mut a, i)` | `IndexMut` |

Overload resolution follows method lookup rules: the compiler searches for an `impl` of the corresponding trait in the current scope and in the defining modules of the operand types. No implicit type coercion is performed to satisfy operator resolution; all operand types must match exactly.

Overflow‑suffixed operators (`+%`, `+?`, `+!`) are not overloadable; they are compiler intrinsics that apply the overflow policy after the underlying addition, for both integer and floating-point types.

The error propagation operator `?`, the compile‑time call marker `!`, and the `as`/`as!` casts are not part of the trait system; they are compiler‑defined operations.

### Dynamic Dispatch

When static dispatch is not possible (e.g., heterogeneous collections), the `dyn` keyword creates a trait object: `dyn Trait`. Trait objects use dynamic dispatch via a vtable and may incur a heap allocation. Their use is explicit to ensure reviewers can identify runtime dispatch points.

```
set handlers: [dyn Fn(Int<32>) -> Int<32>; 10] = default;
```

**Restrictions in strict mode:** In strict mode, constructing or calling through a `dyn Trait` object is only allowed inside `@trusted` functions, because the compiler cannot statically determine the concrete implementation and therefore cannot perform contract verification or exhaustive control‑flow analysis across the dispatch boundary. See the Safety Guarantees section for details.

### Built‑in Traits

The following traits are defined by the language and automatically implemented for applicable types:

- `Add`, `Sub`, `Mul`, `Div`, `Rem` – arithmetic operators
- `Eq`, `Ord` – comparison operators
- `Copy` – bitwise copy semantics
- `Clone` – explicit duplication
- `Default` – default value construction
- `Drop` – destructor
- `Deref` – explicit dereferencing; `@auto_deref` may be attached to grant method‑call auto‑deref
- `Fn` – callable objects (functions and closures)
- `Display` – formatting
- `Serialize`, `Write` – I/O traits (standard library)

## Concurrency Model

### Tasks

Basic concurrency unit:

```
set worker = task { /* code */ };
```

### Channels

Typed, bounded, synchronous channels:

```
let (sender, receiver) = Channel<Int<32>>::new(16);
```

### `async`/`await`

For non‑blocking operations, functions return `Future<T>`:

```
async def fetch_data(url: &[Byte]) -> Result<Data, Error> { ... }

def main() -> Result<(), Error> {
    set data = await fetch_data(b"...")?;
}
```

The `await` keyword marks a suspension point that must be visible to reviewers.

### Interrupts

Interrupt handlers are modeled as special, strongly constrained tasks:

```
@interrupt(irq = 14)
def timer_handler() -> ! {
    // must satisfy @no_alloc, @no_panic constraints; cannot have custom parameters or return a value
}
```

- Interrupt handler functions must have the return type `!` (never return).
- The compiler enforces that interrupt handlers satisfy the constraints of `@no_alloc` and `@no_panic`; violations are compile‑time errors. Redundant explicit `@no_alloc` or `@no_panic` annotations are allowed and produce no warning.
- Interrupt handlers cannot have custom parameters.
- The compiler automatically generates the interrupt vector table from all `@interrupt` annotations, respecting platform ABI and optional `@layout` attributes.

### Task Isolation

The `isolate` block guarantees that the enclosed code does not access any external mutable state. The compiler verifies this property statically, enabling safe concurrent access patterns.

```
isolate {
    // only local variables and immutable external references are accessible
    set x = compute_safe();
}
```

`isolate` blocks can be called from multiple interrupt contexts without data‑race risks.

## Modules and Imports

### Visibility

All symbols are private by default. Use `pub` to export.

### Importing

- `import std::io;` — qualified access `io::puts(...)`.
- `import std::io as my_io;` — alias.
- `import std::{io, fs};` — nested paths.
- Wildcard import is prohibited (`import *` is illegal).

### Package Layout

Project defined by `posita.toml`. Compiler resolves full dependency graph.

### Audit Rules

Projects may declare module‑level effect restrictions in `posita.toml`
under the `[audit.rules]` section:

```
[grades]
default = "usage"        # usage | bounded | relevance | sec
exact = ["safety::**"]   # modules locked to exact grades

[audit.rules]
deny_effects = ["io:net", "io:serial"]
require_grades = ["crypto::**"]
```

Each entry in `deny_effects` is an effect tag. If any function in the
module (transitively) exhibits a denied effect, compilation fails with
a diagnostic identifying the function and the prohibited effect.

The `require_grades` entry specifies modules where all public function
parameters must carry explicit `@consume` annotations; ungraded public
binders in these modules are compile‑time errors.

This mechanism enforces security and safety boundaries at the module
level, ensuring that critical code cannot accidentally perform
restricted I/O.

## Language Versioning and Deprecation

### Editions

Each file declares its edition:

```
edition = "2026";
```

### Deprecation

```
@deprecated("Use `new_method` instead")
def old_method() { ... }
```

### Experimental Features

```
@experimental
type NewInteger = Int<128>;
```

Requires `--enable-experimental` flag. When enabled, `Int<bits>` may accept bit widths up to 128.

### Conditional Compilation

Modules, functions, or type definitions can be conditionally compiled using the `@cfg` attribute with the `all`, `any`, `not` combinators:

```
@cfg(all(target_os = "linux", target_arch = "x86_64"))
def platform_specific() { ... }

@cfg(any(feature = "logging", debug))
def maybe_log(msg: &Str) { ... }

@cfg(not(target_os = "windows"))
def unix_only() { ... }
```

In strict mode, all compilation paths must be provably reachable under at least one configuration.

## Compile‑Time Execution

Posita does not support procedural macros or any form of syntax‑level code generation outside the `comptime` system. All compile‑time code generation is performed by `comptime` functions, which are type‑checked, sandboxed, and subject to resource limits. This ensures that all code—whether executed at runtime or at compile time—is visible, auditable, and subject to the same safety guarantees.

**Important:** `comptime` blocks and functions operate at the expression and statement level. They can produce values, type aliases, and type metadata, but they cannot generate top‑level declarations such as `impl`, `trait`, `def`, or `type` definitions. This is a deliberate design separation: `comptime` handles value‑level compile‑time computation, while declarative code generation (deriving traits, generating functions for types) is the role of `generate` blocks (see below).

**Safety restrictions:** `comptime` code runs in a sandboxed environment. It is prohibited from performing any of the following:

- File system writes
- Network access
- Spawning processes
- Calling `@io(write)` functions or any `@trusted` functions that perform I/O

Violations are detected at compile time and cause a hard error. This guarantees that `comptime` evaluation is purely a deterministic, side‑effect‑free transformation of the source program.

**Proof obligations for generated code:** Code generated by `generate` blocks is subject to the same verification standards as hand‑written code. Any `@trusted` function produced by `generate` must include corresponding `@link_proof` or `@comptime_test` evidence; otherwise, compilation fails in strict mode.

### comptime Blocks

```
comptime { /* compile‑time code */ }
```

`comptime` blocks are generic compile‑time computation engines. They may contain statements (including loops, conditionals, and calls to `comptime` functions), but they may not contain item‑level declarations such as `impl`, `type`, or `def`. This restriction ensures that `comptime` blocks cannot silently inject new bindings into the enclosing scope, preserving the principle that a type's complete behavior is statically known from its definition site.

For declarative code generation that produces module‑level declarations, see `generate` blocks (planned).

#### Capturing Outer Constants

By default, a `comptime` block cannot refer to variables from the enclosing runtime
scope. To make compile‑time‑known values accessible, provide an explicit capture
list after the `comptime` keyword:

```
set answer = 42;

comptime [answer] {
    // answer is a compile‑time constant here
    @compile_error!("The answer is {}.", answer);
}
```

Each captured variable must be an immutable binding (`set` or `let`) whose
initializer is a compile‑time constant expression—a literal, a pure
`comptime` function call, or a combination of other captured compile‑time
constants. The compiler evaluates the initializer at compile time and makes the
value available inside the block. Attempting to capture a variable whose value
cannot be determined at compile time results in a compile‑time error.

#### Comptime Blocks as Expressions

A `comptime` block may be used as an expression. In that case, the block's
value is computed at compile time and treated as a compile‑time constant. The
resulting value can be assigned to an immutable variable with `set` and later
captured into other `comptime` blocks via the explicit capture syntax.

A `comptime` block cannot access variables declared in another `comptime` block
directly; all compile‑time data sharing must go through the explicit capture
mechanism.

#### Trust Boundaries in comptime Blocks

A `comptime` block may optionally be annotated `@trusted`:

```
comptime @trusted [captures] { ... }
```

An `@trusted` comptime block is allowed to:

- Call `@trusted` functions and methods
- Access `unsafe` operations

The block's author assumes full responsibility for satisfying the contracts
of all `@trusted` operations within it. The compiler will not verify these
contracts at compile time. Non‑`@trusted` comptime blocks cannot call
`@trusted` functions or contain `unsafe` code.

This mechanism mirrors the runtime `@trusted` function annotation and ensures
that trust boundaries remain explicit and auditable even during compile‑time
execution.

### comptime Functions

```
comptime def eval_polynomial(...) -> Int<64> { ... }
```

Calls to `comptime` functions must be marked with `!` to make the compile‑time nature explicit at the call site:

```
set val = eval_polynomial!(coeffs, x);
```

This ensures that reviewers can immediately identify where compile‑time evaluation occurs.

### Comptime Tests

`@comptime_test` blocks execute at compile time and are used to validate `@trusted` code against concrete inputs:

```
@comptime_test
def test_trusted_function() {
    set val = some_trusted_function(test_input);
    assert(val == expected_output);
}
```

Test failures cause a compile error. `@comptime_test` functions are stripped from the final binary. Contract‑verification counterexamples can be automatically converted into such tests.

### Type Factories

`comptime` functions can return `type`:

```
comptime def make_vector(Elem: type, N: usize) -> type {
    return [Elem; N];
}

type Vec4f = make_vector!(Float<32>, 4);
```

`type` is a first‑class value in `comptime` contexts and can be passed, returned, and stored.

Execution is bounded by step/memory limits to prevent non‑termination.

### Compile‑Time Reflection (`@typeInfo`)

```
comptime {
    set info = @typeInfo!(MyStruct);
    for field in info.fields { /* generate code */ }
}
```

Public reflection (`@typeInfo!`) only exposes `pub` items. Internal reflection (`@typeInfo!` with full access) is available only inside `@trusted` comptime blocks.

### Built‑in Compile‑Time Utilities

The following utilities are available in `comptime` contexts:

- `assert(condition)`: Evaluates `condition` at compile time. If the condition is `false`, compilation halts with an error message. `assert` is stripped from the final binary and cannot be used for runtime checks.
- `@compile_error("msg")` and `@compile_error!("msg")`: Both forms are legal. The built‑in `compile_error` is descriptive enough to be recognized as a compile‑time operation, so the `!` marker is optional. Using `!` explicitly remains consistent with other `comptime` call sites and may be preferred for clarity. Both unconditionally halt compilation with the given message. Typically employed in `comptime` to reject invalid code generation or type combinations.

### Optimization Hooks (advanced)

```
comptime {
    set plan = optimize(target = my_function, strategy = q_learn(config));
    plan.apply();
}
```

### Lemma Functions

Lemma functions are `comptime` functions that return auxiliary assertions to assist the SMT solver:

```
@lemma
comptime def pow2_induction_hint(n: Int<32>) -> [Assertion; 2] { ... }
```

They are invoked by placing `@apply_lemma(pow2_induction_hint)` on the target function's definition, which causes the compiler to inject the returned assertions into the SMT solver's context during verification of that function.

### `generate` Blocks (Planned)

`generate` blocks provide a declarative, auditable mechanism for compile‑time code generation. Unlike `comptime` blocks, which are general‑purpose computation engines that cannot directly inject top‑level declarations, `generate` blocks are the only mechanism for producing module‑level declarations (`impl`, `def`, `type`, `const`) in a controlled and reviewable way.

This feature is planned for v0.2 and has been prioritized in response to community feedback. The `comptime` system deliberately excludes declaration‑level generation to maintain a clean separation: `comptime` handles value‑level computation, while `generate` handles trait derivation and declaration synthesis.

- **Attachment:** A `generate` block must be explicitly attached to a type definition or module using `generate for <TypeOrModule>`.
- **Declarative generation:** It may contain any module‑level declaration, such as `impl` blocks, function definitions, type aliases, or compile‑time constants. These declarations are expanded and injected into the enclosing scope at compile time.
- **Declarative name mapping (no implicit identifier splicing):** Posita rejects traditional implicit identifier concatenation (e.g., `concat_idents!` or `#`‑based splicing) because it undermines static auditability, breaks determinism, and obscures the connection between generated code and its source. Instead, `generate` blocks will adopt a declarative name mapping approach. Within a `generate` block, reusable templates can be defined with placeholder slots (e.g., `[field]`). During expansion, the compiler instantiates each template for the appropriate compile‑time entities (such as struct fields), mapping the placeholder to a concrete name derived directly from the source. This ensures every generated identifier has a clear, searchable origin in the source template.
- **Declarative iteration:** `generate` blocks support iterating over compile‑time known sequences, such as the fields of a struct obtained via `@typeInfo!(T).fields`. These loops are fully unrolled at compile time and must be side‑effect free. They enable per‑field code generation without sacrificing auditability.
- **Pure and deterministic:** `generate` blocks are side‑effect free. They may use conditionals (`if`) and field‑wise iteration based on compile‑time constants (e.g., `@typeInfo`), but they cannot call functions with `@io` effects or rely on non‑deterministic input. The transformation from type information to generated declarations must be entirely deterministic.
- **Auditability:** All code generated for a type is visible directly below its definition. Reviewers can understand the full interface of a type without searching the entire codebase for scattered `comptime` blocks that might inject implementations.
- **Error diagnostics:** Errors inside a `generate` block point to the original source location within the block, preserving the correspondence between the generator and the generated code. Contextual information (e.g., "in expansion of `impl Serialize for MyStruct`") is provided for semantic errors.

Example (draft syntax, subject to change):

```
type MyStruct = struct {
    x: Int<32>,
    y: Int<32>,
}

generate for MyStruct {
    if @typeInfo!(MyStruct).fields'len <= 4 {
        impl Copy for MyStruct { }
    }

    // Generate a getter for each field via a name‑mapped template
    def [field] get(self: &MyStruct) -> field.type {
        return self.[field];
    }

    impl Serialize for MyStruct {
        // ...
    }
}
```

Note: This feature is planned for a future release and is not yet implemented. The exact syntax for name mapping, iteration, and template instantiation remains under design.

## Statements and Expressions

**Statement termination:** Every statement in Posita must be terminated with a semicolon. There is no implicit semicolon insertion, and the compiler will reject any statement that lacks a terminating `;`. The only exception is the final expression in a block that is used as an expression (e.g., the tail expression of an `if` expression or `match` arm), which may omit the semicolon to denote that its value is yielded to the enclosing expression. Function return values must always be produced via an explicit `return` statement; a trailing expression without a semicolon at the end of a function body does not constitute an implicit return.

### Variable Declaration

```
set identifier : Type = expression;   // full form
set identifier = expression;          // type inference
set identifier : Type;                // uses type's default (unless no_default)
```

Variables are immutable by default. Use `mut` for mutability:

```
set mut x = 0;
x = x + 1;
```

**Variable shadowing:** A variable declared in an inner scope may shadow
a variable of the same name from an outer scope. This is allowed and follows
lexical scoping rules. However, the compiler will reject shadowing of
reserved contract keywords (`codomain`, `old`, etc.). Redeclaring an
immutable variable with the same name in the same scope is also an error.

### `let` Bindings

`let` is a restricted, immutable‑only variant of `set` that additionally supports pattern destructuring. A `let` binding is always immutable; there is no `let mut`. It must always be explicitly initialized and cannot rely on a type's default value. A `let` binding that specifies a type annotation without an explicit initializer (`let x: Type;`) is a compile‑time error; use `set` instead.

- Simple binding: `let x = expr;` (equivalent to `set x = expr;`)
- Tuple destructuring: `let (x, y) = tuple_expr;`
- Struct destructuring: `let Point { x, y } = point_expr;`
- Enum variant destructuring with mandatory `else`:

```
let Some(value) = opt_expr else { leave with Error::None; };
```

The `else` block must diverge (via `return`, `leave with`, `panic`, etc.).

### Assignment

`x = value` (requires `mut`). Compound assignments: `+=`, `-=`, `*=`, etc.

## Control Flow

Conditional: `if condition { ... } else { ... }`

`if` as an expression: `if` is an expression that returns a value. All branches must return the same type, and the `else` branch is mandatory when used as an expression:

```
set msg = if x > 5 { b"big" } else { b"small" };
```

`if let`: `if let Some(val) = opt { ... }` `if let` tests a single pattern and executes the body if it matches. Unlike `let ... else`, which requires the pattern to succeed and forces a divergent `else` block on failure, `if let` is designed for optional destructuring where the non‑matching case simply does nothing (when used as a statement) or falls through to an optional `else` expression. `if let` may be used as an expression. When used as an expression, an `else` branch is mandatory, and both branches must produce values of the same type. This is consistent with the requirement for `if` expressions. When `if let` is used as a statement, the `else` branch is optional.

`while let`: `while let Some(item) = iter.next() { ... }`

Loops: `for item in iterable { ... }`, `while condition { ... }`, `loop { ... }`

Leaving loops: `leave;` (exits `for`, `while`, or `loop`), `leave 'label;`, `continue;`

**Never type (`!`)**: The `!` type represents a computation that never returns (e.g., `panic`, infinite loops, or divergent control flow). It can be used as the return type of functions that do not terminate normally, enabling the compiler to perform control‑flow analysis. `!` vs `()`: The `!` type represents the absence of any value and is uninhabited—no expression can produce a value of type `!`. The `()` type is a unit value, representing an empty but existing value. Use `!` for functions that never return or whose return value must never be used; use `()` when a generic instantiation requires a concrete, ignorable value. This distinction eliminates the need for a separate `unit` keyword. See also "Empty Tuple" in Composite Types.

## Functions

```
def function_name(param1: Type1, param2: Type2) -> ReturnType { ... }
```

Default parameter values: `def f(x: Int<32> = 0) { ... }`

A function body must use an explicit `return` statement to produce its result; the final expression in a body is never implicitly returned.

### Two-Phase Borrows

When a method call `receiver.method(args)` or a function call where a
mutable borrow `&mut x` and another reference to `x` appear as arguments,
the compiler implicitly defers the activation of the mutable borrow
until after all arguments have been evaluated. During argument
evaluation, `x` remains readable (its mutable borrow is in a `reserved`
state, not yet `active`). Once all arguments are fully evaluated and any
shared borrows of `x` within them have reached their last use, the
mutable borrow activates and `x` becomes exclusive.

This relies on point‑level liveness: shared borrows of `x`
within the arguments die at their last use point (the argument-passing
expression), not at the end of a lexical block. The mutable borrow
activates only after all shared borrows are dead, so the two borrows
never overlap.

This enables:

```
v.push(v.len());   // len() reads v (reserved), then push borrows &mut v (active)
```

The explicit `&ro` form remains available:

```
v.push((&ro v).len());  // explicit read-only borrow, scoped to the sub-expression
```

Two-phase borrows are a compiler-internal rule and do not require opt-in.
They do not interact with `@auto_ro` or `@auto_coerce` attributes.

### Return Value Semantics

When a function returns a value of a non‑`Copy` type, the compiler guarantees that the return value is constructed directly in the caller's stack frame. This eliminates the intermediate `move` and any associated `Drop` call. Named return value optimization (NRVO) is guaranteed when all return paths return the same local variable that has not had its address taken.

For `Copy` types, no such guarantee is made; the value is returned via standard register or stack conventions.

### Move Elision for Non‑Copy Types

For types that are not `Copy`, the compiler guarantees that `move` operations are elided in the following contexts:

- **Return value optimization (RVO):** A returned value is constructed directly in the caller's return slot.
- **Named return value optimization (NRVO):** A local variable that is returned on all paths, and has not had its address taken, is constructed directly in the return slot.
- **Temporary materialization:** A temporary value (e.g., the result of a function call or constructor expression) that is immediately used to initialize a variable, field, or parameter is constructed directly in the target location.
- **Argument passing:** An argument of non‑`Copy` type is passed directly to the callee's parameter slot without an intermediate `move`.

In all these cases, no intermediate `move` is performed, and no corresponding `Drop` call occurs on the source location.

These guarantees ensure that factory functions, builder patterns, and value‑oriented APIs have predictable, zero‑overhead semantics for non‑`Copy` types.

## Pattern Matching

**Pattern refutability**: `set` does not support pattern destructuring;
it binds a simple identifier only. `let` is Posita's sole pattern
destructuring construct. When the pattern is irrefutable (e.g. tuples,
structs), `let` behaves like an immutable binding. When the pattern is
refutable (e.g. enum variants), `let` requires an `else` block to handle
the non‑matching case. `if let` and `while let` accept both refutable and
irrefutable patterns, but the compiler warns when an irrefutable pattern
is used (since the conditional is unnecessary).

```
match value {
    pattern1 | pattern2 if guard_condition => expression1,
    pattern3 => { statements },
    _ => default,
}
```

Or patterns: `pattern1 | pattern2` matches if either pattern matches. Both patterns must bind the same set of variables with compatible types.

Guards: The `if guard_condition` clause filters matches. Guard expressions must be `@pure` and free of side effects.

**GADT Refinement:** When matching on a GADT enum, the compiler automatically refines the enum's type parameters according to the variant's `when` clauses. Within each branch, the type equalities implied by the matched variant become available, enabling type‑safe operations without casts. See the GADT section for details.

### Slice Patterns

Slice patterns allow destructuring of slices (`&[T]`) and arrays (`[T; N]`) without copying elements. They are usable in `match`, `if let`, `while let`, `for`, and `loop` contexts. All element and sub‑slice bindings are views onto the original data; no allocation or element‑wise copy occurs.

Supported forms (where `T` is the element type, and `view` is of type `&[T]`):

| Pattern | Matches | Bindings |
|---|---|---|
| `[]` | Empty slice | – |
| `[_]` | Exactly one element | – (element ignored) |
| `[x]` | Exactly one element | `x: &T` (immutable reference to element) |
| `[x, ..rest]` | At least one element | `x: &T`, `rest: &[T]` (remaining sub‑slice) |
| `[..init, x]` | At least one element | `init: &[T]`, `x: &T` |
| `[x, ..mid, y]` | At least two elements | `x: &T`, `mid: &[T]`, `y: &T` |
| `_` | Any slice | – |

**Binding rules:**

- Element bindings (`x`, `y` etc.) are always of type `&T`, regardless of whether `T` implements `Copy`. This avoids any ownership transfer and keeps the original slice intact.
- Sub‑slice bindings (`rest`, `mid`, `init`) are of type `&[T]`.
- To obtain an owned copy of an element when `T: Copy`, use a separate `let` statement (e.g., `let val = *x;`).
- Patterns are refutable unless they cover all possible lengths. For example, `[]` is irrefutable only for an empty slice; `[x, ..rest]` is irrefutable for any non‑empty slice but refutable for a general `&[T]`.

Examples:

```
def sum(xs: &[Int<32>]) -> Int<32> {
    return match xs {
        [] => 0,
        [head, ..tail] => *head + sum(tail),
    };
}

def is_palindrome(xs: &[Int<32>]) -> Bool {
    set mut remaining = xs;
    loop {
        match remaining {
            [] | [_] => return true,
            [first, ..middle, last] if *first == *last => {
                remaining = middle;
                continue;
            },
            _ => return false,
        }
    }
}
```

**Interaction with borrowing:**

A slice pattern binds references derived from the original slice. The original data remains borrowed (immutably) until all bindings go out of scope. This is enforced by the existing borrow checker.

## Structured Resource Cleanup

Posita provides three complementary mechanisms for resource cleanup, each with distinct guarantees. The following table summarizes their roles:

| Mechanism | Failure handling | Execution timing | Typical use case |
|---|---|---|---|
| RAII (`Drop` trait) | Infallible | Automatic at scope exit (declaration order reversed) | File handles, locks, memory |
| `finally` block | Infallible | At scope exit on all paths, after `scope_cleanup` actions | Simple, always‑run cleanup |
| `scope_cleanup` | Fallible (opt‑in via `propagates`) | Deferred to scope exit (LIFO), with explicit early trigger via `trigger @name` | Multi‑exit fallible cleanup, non‑ownership actions |

### RAII (Resource Acquisition Is Initialization)

RAII is the foundational resource management strategy. Types that implement the `Drop` trait automatically run their destructor when the value goes out of scope. The compiler guarantees that destructors execute in reverse order of declaration and that they cannot fail. See the `Drop` trait in the Traits section for details.

### `finally` Blocks

```
def process() -> Result<(), Error> {
    set file = File::open("data.bin")?;
    return Ok(());
} finally {
    file.release_buffer();  // infallible cleanup: no ?, leave with, return, or panic allowed
}
```

`finally` blocks are reserved for infallible cleanup that must execute on every exit path. They are the simplest mechanism after RAII. For fallible cleanup, use `scope_cleanup` and handle errors explicitly.

### Named Scope Cleanup (`scope_cleanup`)

`scope_cleanup` registers a named, deferred code block that is guaranteed to execute when the enclosing scope is exited, unless it is explicitly triggered earlier via `trigger @name`. It is designed for cleanup actions that may fail and that must occur regardless of which exit path a function takes, including early returns, `leave with`, and `?` propagation.

**Syntax:**

```
scope_cleanup @name {
    // cleanup code; may contain explicit error handling (catch, match, etc.)
}
```

or, to allow error propagation:

```
scope_cleanup @name propagates {
    // cleanup code; '?' is permitted to propagate errors
}
```

**Conditional cleanup:** A `scope_cleanup` block may specify a compile‑time guard
via the optional `when` clause:

```
scope_cleanup @name when condition {
    // only executed when `condition` is true at the point of scope exit
}
```

The `condition` must be a compile‑time predicate—it may reference only `ghost`
variables, `const` generic parameters, and other compile‑time‑constant
expressions. The compiler evaluates it at each exit point and omits the
cleanup block on paths where the condition is `false`. This preserves the
"erased at runtime" guarantee for ghost state while allowing conditional
resource management without runtime overhead.

The block captures variables from the enclosing scope immutably or via `&mut` (subject to borrow rules). It is not a first‑class closure; it cannot escape the scope.

Multiple `scope_cleanup` blocks in the same scope execute in LIFO (last‑in, first‑out) order when the scope is exited.

An explicit `trigger @name;` statement executes the cleanup block immediately and removes it from the deferred list. It will not execute again at scope exit. `trigger` is a statement, not an expression.

Early exits (`return`, `leave with`, `leave`, `continue` to outer labels) are compile‑time errors inside a `scope_cleanup` block. This ensures the block is non‑escaping and preserves the LIFO execution guarantee.

**Error handling:**

The behaviour of error propagation within a `scope_cleanup` block is governed by the presence of the `propagates` and `overrides` modifiers, as summarised below:

| Modifiers | `?` operator | `catch` implicit propagation |
|---|---|---|
| (none, default) | Forbidden (compile error) | Forbidden (must be exhaustive; `|_|` allowed) |
| `propagates` | Allowed (error added to signature) | Allowed (equivalent to `?`) |
| `propagates overrides` | Allowed (error overrides prior errors) | Allowed (error overrides prior errors) |

By default, the cleanup block must not contain unhandled `Result` values. The `?` operator is forbidden, and any fallible operation must be explicitly handled (e.g., with `catch`, `match`, or `let _ = ...`). This prevents cleanup from silently altering a function's error signature.

When the `propagates` modifier is present, the block may use `?` to propagate errors. The compiler's may‑be analysis automatically includes the propagated error variants in the enclosing function's `E` type. The programmer is responsible for ensuring that the error is meaningful to callers.

**Error precedence:** If a function is already exiting with an error (via `leave with`), and a `scope_cleanup` with `propagates` also fails, the original error is preserved and the cleanup error is recorded in the audit log (if `@audit_log` is present) but does not replace the original. If the function would exit normally (`return Ok`), a cleanup error becomes the function's return value.

**`overrides` modifier:** In rare cases where a cleanup error is more critical than the original error (e.g., data loss from a failed buffer flush), the programmer may explicitly mark a `scope_cleanup` block with `overrides`. This must be used together with `propagates`. When `overrides` is present and the cleanup block fails while the function is already in an error state, the cleanup error replaces the original error as the function's final error return. Only one `scope_cleanup` block per scope may use `overrides`. This decision point is flagged by `capsa audit`.

```
scope_cleanup @flush_critical propagates overrides {
    file.flush()?;  // this error will override any prior business error
}
```

**Example — conditional rollback using `when`:**

```
def process() -> Result<(), DbError> {
    let tx = db.begin()?;
    ghost set mut committed = false;

    scope_cleanup @rollback when not committed propagates {
        db.rollback()?;
    }

    // ... business logic ...
    db.commit()?;
    ghost set committed = true;
    return Ok(());
}
```

Because `committed` is a ghost variable and the `when` clause is evaluated at
compile time, the runtime code contains no dynamic branching for the cleanup
guard. Paths where `committed == true` simply omit the `scope_cleanup` block
entirely.

**Asynchronous functions:**

`scope_cleanup` blocks are always synchronous. They cannot contain `await` expressions or call `async` functions without immediately blocking (which is strongly discouraged). In an `async` function, use `scope_cleanup` only for synchronous cleanup (e.g., releasing memory, sending a synchronous message). For asynchronous resource closure, use an RAII type with an async `close` method for the graceful path, and an infallible synchronous `Drop` as a best‑effort safety net.

**Borrowing implications:**

Variables borrowed (especially `&mut`) by a `scope_cleanup` block remain borrowed until the scope exits, even after a `trigger` has executed the block. This is a conservative behaviour of the borrow checker. To allow the variable to be used again after the trigger point, introduce a new scope:

```
{
    scope_cleanup @flush { file.flush()?; }
    // ... use file ...
    trigger @flush;
}
// file is no longer borrowed here
```

The compiler will emit diagnostics identifying the `scope_cleanup` block that extends the borrow, and suggest this pattern where appropriate.

**Design rationale:** `scope_cleanup` fills the gap between infallible RAII/`finally` and manual error handling. It provides single‑point auditability for cleanup in functions with multiple exit paths, while keeping error propagation explicit and controlled.

## Expressions

Arithmetic: `+`, `-`, `*`, `/`, `%`. The operators `+`, `-`, `*` accept optional overflow suffixes (`%`, `?`, `!`).

Bitwise: `&`, `|`, `^`, `<<`, `>>`, `~`

Logical: `and`, `or`, `not`

Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`

Dereference: `*ptr`

Address‑of: `&var`

Cast: `value as NewType` (safe), `value as! NewType` (bitcast)

Rounding suffixes for float‑to‑int conversion appear after the target type in an `as` cast:

```
value as Int<32> round   // round to nearest, ties to even
value as Int<32> trunc   // truncate toward zero (default)
value as Int<32> ceil    // ceiling
value as Int<32> floor   // floor
```

**Operator precedence (highest to lowest):**

| Precedence | Operators | Associativity |
|---|---|---|
| 1 (highest) | `*`, `/`, `%` | left‑to‑right |
| 2 | `+`, `-` | left‑to‑right |
| 3 | `<<`, `>>` | left‑to‑right |
| 4 | `&` | left‑to‑right |
| 5 | `^` | left‑to‑right |
| 6 | `|` | left‑to‑right |
| 7 | `==`, `!=`, `<`, `>`, `<=`, `>=` | left‑to‑right |
| 8 | `and` | left‑to‑right |
| – | `implies` | right‑to‑left (only in contracts) |
| 9 | `or` | left‑to‑right |
| 10 | `not` | right‑to‑left (prefix) |
| 11 (lowest) | `..`, `..=` | left‑to‑right |

**`as!` layout compatibility:** The compiler verifies that the source and target types have the same size and alignment via `layout_of!`, or, in the case of truncation, that the truncated value does not violate the target type's `invariant`. All uses of `as!` are flagged by `capsa audit` for mandatory human review.

**Alignment compatibility:** The compiler also verifies that the target
type's alignment is not stricter than the source type's alignment, unless
the source is a raw pointer (`*T` or `Ptr<...>`) and the entire cast
plus the subsequent dereference are contained within an `unsafe` block.
In `unsafe` blocks, the programmer assumes full responsibility for
ensuring that the resulting pointer is correctly aligned for its type;
violation may result in undefined behavior (including hardware faults on
architectures that trap on misaligned access).

For safe code that must read or write misaligned data, the standard
library provides `from_unaligned` / `write_unaligned` functions that
perform byte‑wise copies without alignment requirements.

### Move Semantics

The `move` keyword explicitly transfers ownership of a non‑`Copy` value. It may be used in:

- Assignments: `set target = move source;`
- Function arguments: `consume(move value);`
- Closure captures: `let closure = |...| capture value by move { ... };`

After a move, the source variable is invalidated and any subsequent use is a compile‑time error. The compiler will not insert a `drop` call for the moved‑from variable.

### First-Class Polymorphism (`poly` / `unbox`)

Posita supports first‑class polymorphic values through the `poly` and `unbox` keywords. This allows generic functions (whose types carry `Forall` quantifiers) to be boxed into first‑class values, passed around, stored in data structures, and instantiated at different types at different use sites — without losing the ability to be re‑instantiated.

- `poly(expr)` — Boxes a polymorphic expression into a first‑class polytype value. The expression must have a `Forall` type (i.e., a generic function, or a value whose type is universally quantified). The result type is `Poly { quantifiers, body }`, a first‑class type that can be stored, passed, and returned.
- `unbox(expr)` — Unboxes a `Poly`-typed value by instantiating each of its quantifiers with a fresh type variable. The result is a monotype (or still‑polymorphic, depending on the body) that can be applied or used further. Calling `unbox` on the same `poly` value multiple times produces fresh instantiations each time.

**Let‑Generalization**

When the result of an `unbox` expression is bound by `set` or `let`, the compiler generalizes any remaining inference variables that are not constrained by the enclosing context. This allows the bound identifier to be used polymorphically at multiple types, exactly like a regular generic function. The underlying mechanism is Posita's implementation of let‑generalization (from ML‑family languages), which ensures that a single `unbox` followed by a binding yields a fully polymorphic value without requiring repeated `unbox` calls.

**Example — boxing and unboxing the identity function:**

```
def id<T>(x: T) -> T { return x; }

def main() -> Int<32> {
    set p = poly(id);         // box id into a first-class polytype
    set f = unbox(p);         // unbox → let-generalization makes f: ∀T. T → T
    set x = f(42);            // instantiated for Int<32>
    let _ = f(true);          // instantiated for Bool — same poly, different types
    return x;
}
```

**Multiple instantiations via let‑generalization:**

```
def main() -> Int<32> {
    set p = poly(id);
    set f = unbox(p);         // f is polymorphic
    set x = f(42);            // uses Int<32>
    set y = f(true);          // uses Bool
    return x;
}
```

**Multiple quantifiers:**

```
def pair<T, U>(a: T, b: U) -> T { return a; }

def main() -> Int<32> {
    set f = unbox(poly(pair));  // f: ∀T U. T × U → T (let‑generalized)
    return f(42, true);         // f(42, true): Int<32>
}
```

**Inline boxing and unboxing:**

```
def main() -> Int<32> {
    return unbox(poly(id))(42);
}
```

**Error cases:**

- `poly(42)` is rejected at compile time: `poly(...)` requires a polymorphic expression, but `42` is a concrete `Int<32>`.
- `unbox(x)` where `x : Int<32>` is rejected: `unbox(...)` requires a polytype value; the compiler will report an error when the concrete non‑poly type is resolved.

**Design note:** `poly`/`unbox` follow the same design as higher‑rank polymorphism in ML dialects (e.g., the `[∀α.τ]` boxed type in OmniML §3.1). The `Poly` type is covariant in its body and participates in unification with α‑conversion of quantifier indices. Posita's built‑in let‑generalization ensures that values obtained via `unbox` are fully polymorphic when bound, retaining the ergonomics of parametric polymorphism. This enables type‑safe passing of polymorphic functions without requiring a separate `dyn`‑style vtable dispatch mechanism.

## Error Handling

**Error handling philosophy:** Posita's error handling rests on three pillars designed to make every failure path explicit and auditable:

1. `Result<T, E>` — the only carrier of recoverable errors, providing type‑level separation of success and failure.
2. `?` — the propagation operator, providing concise, zero‑cost forwarding of errors with full type visibility.
3. `catch` / `leave with` — structured control flow for local error handling, conversion, and exit.

The compiler accepts the error value directly (`leave with ErrorVariant`). The `Err(...)` wrapping is a semantic operation performed during type checking—the error value is type‑checked against the function's error type `E` and recorded as an `ErrorExit` in the control‑flow graph. This is not a syntactic rewrite into `return Err(...)`; `leave with` retains its distinct identity throughout compilation for audit, contract verification, and control‑flow analysis.

### The `Result` Type

`Result<T, E>` is a built‑in enum:

```
type Result<T, E> = enum { Ok(T), Err(E) }
```

### Propagating Errors with `?`

```
def read_file(path: &[Byte]) -> Result<String, FileError> {
    set file = File.open(path)?;   // propagates FileError
    // ...
}
```

No type‑erased errors; fully monomorphized, zero overhead.

When a function is annotated `@no_alloc_error`, the compiler verifies that every `?` propagation path uses only `From` implementations that are themselves `@no_alloc`. If any conversion in the error path could allocate, the `?` operator is rejected. Simple enum‑to‑enum conversions without payload transformations automatically satisfy this requirement.

### Handling Errors Locally with `catch`

A `catch` expression has the type `T` where the preceding expression has type `Result<T, E>`. Each branch of `catch` must either diverge (via `leave with`, `panic`, etc.) or produce a value of type `T`. The patterns in `catch` branches are the enum variant names directly (e.g., `|IoError| { ... }`), not qualified paths, because the error type is already known from the expression. If a variant name is ambiguous in the current scope, the compiler requires explicit qualification using `EnumName::Variant`.

If the enclosing function does not return a `Result` whose error type can carry unmatched variants, a `catch` expression must be exhaustive over `E`, or must include a wildcard branch that produces a value of type `T`. Otherwise, any unmatched variant is implicitly propagated—equivalent to `leave with Err(unmatched_variant)`—which is only valid when the function returns a `Result<_, E>`.

Example with divergence:

```
def process() -> Result<(), ProcessError> {
    set data = fetch() catch {
        |NetworkError as e| { log(e); leave with ProcessError::NetworkFail; }
        |ParseError => { leave with ProcessError::BadData; }
    };
    return Ok(());
}
```

- `as` binding: Binds the error value to a local variable for inspection or logging.
- Arrow shorthand (`=>`): When a branch body is a single expression (e.g., a return value), the `=>` syntax can replace curly braces for brevity.

Example with a fallback value (non‑diverging, exhaustive):

```
def fetch_or_default() -> Data {
    set data = fetch() catch {
        |NetworkError| { cached_default }
        |_| { cached_default }   // catch‑all required because function does not return Result
    };
    return data;
}
```

**Wildcard pattern (`|_|`):** A wildcard pattern `|_|` matches any error variant not explicitly listed in preceding branches. It can be used as a catch‑all for uniform handling of remaining errors, or to explicitly ignore them. When used, `|_|` must appear as the last branch.

```
def process() -> Result<(), ProcessError> {
    set data = fetch() catch {
        |NetworkError as e| { log(e); leave with ProcessError::NetworkFail; }
        |_| { log("unhandled error"); cached_default }
    };
    return Ok(());
}
```

The compiler verifies that `|_|` covers all remaining variants, and `capsa audit` flags wildcard branches for mandatory review, as they may silently absorb unexpected errors.

**Non‑exhaustiveness**: Unlike `match`, `catch` does not require exhaustiveness when the enclosing function returns a `Result`. Any error variant that is not explicitly matched is implicitly propagated—equivalent to `leave with Err(unmatched_variant)`. This is the fundamental difference between `catch` and `match`: `catch` says "handle these specific errors here, let everything else pass through"; `match` says "handle all possibilities here and now."

This non‑exhaustive design works together with `@must_handle`: a library author can mark specific error variants as `@must_handle`, forcing callers to write a `catch` branch for those variants even if they use a wildcard for the rest.

**When to use `match` instead:** If you need to exhaustively handle every error variant (e.g., in a top‑level error handler that must not let any error escape), use `match`. `catch` is designed for selective interception with implicit propagation of the rest, while `match` provides exhaustive case analysis.

### Early Return with `leave with`

`leave with` is a structured, non‑local exit that returns an error from the current function. It is the only error exit mechanism for `Result`‑returning functions.

The compiler accepts the error value directly:

```
def example() -> Result<Int<32>, MyError> {
    set x = dangerous_op() catch {
        |e| leave with e;          // binds the whole error value
    };
    // ...
}
```

The error value after `leave with` must be of type `E` where the enclosing function returns `Result<_, E>`. The compiler automatically wraps it in `Err(...)`. Using `return Err(e)` in place of `leave with` is a compile‑time error.

### Explicit Error Paths and `From` Conversions

`From` implementations allow automatic error conversion via `?`. All conversions are statically known.

### Fatal Errors: `panic`

Only within `unsafe` contexts or `comptime` blocks.

### Fine‑Grained Error Accountability (`@must_handle`)

By default, `@must_use` ensures that a `Result` is not silently discarded. The `@must_handle` attribute provides finer control, requiring callers to explicitly handle specific error variants:

```
@must_handle(NetworkError, ParseError)
def fetch() -> Result<Data, Error> { ... }

// Caller that ignores NetworkError or ParseError will get a compiler warning
let data = fetch() catch {
    |NetworkError as e| { log(e); cached_default }  // handled
    |ParseError => { leave with ... }                // handled
    // other errors can still fall through
};
```

In strict mode, the warning becomes an error. **Interaction with `?`:** Propagating an error via `?` does not count as handling for the purposes of `@must_handle`. The caller must explicitly match the variant or, on the enclosing function, declare `@delegates_must_handle(NetworkError, ParseError)` to pass the responsibility upstream.

### Static Error Tracking: May‑Be and Must‑Be Analysis

Posita employs two complementary static analyses to enforce accountability for every error path.

**May‑Be Analysis (over‑approximation):** Determines which error variants may reach a given program point. This is used to check that every function signature accurately declares all errors it can propagate. If a `catch` block does not intercept a particular variant, may‑be analysis ensures that variant appears in the enclosing function's return type. Failure to include it is a compile‑time error.

**Must‑Be Analysis (under‑approximation):** Determines which error variants must occur at a specific call site—i.e., when all possible execution paths lead to that error and no successful return is possible. Must‑be analysis is more precise and is employed in strict mode together with `@must_handle`. If the compiler can prove that a particular error variant is unavoidable at a call site, and that variant is marked `@must_handle`, the caller is forced to handle it locally rather than propagate it further. This prevents the indefinite deferral of critical error handling.

Together, these analyses implement a chain of responsibility:

- May‑be ensures errors are never silently dropped from the type signature.
- Must‑be, when combined with `@must_handle`, forces certain errors to be resolved near their origin, upholding the principle that truly critical failures must not be endlessly propagated.

## Contracts and Constraints

### Function Contracts

- `requires`: a precondition.
- `ensures`: a postcondition. Applies to all exit paths unless explicitly qualified with `on Ok(...)` or `on Err(...)`.
- `terminates`: a termination measure. For recursive functions, the argument specified must strictly decrease on each recursive call and have a lower bound, ensuring the recursion always terminates.

Contracts are evaluated in the mathematical integer domain (ideal precision) by default. To switch a function to IEEE 754 floating‑point semantics for all its `requires` and `ensures` clauses, annotate it with `@ieee_contracts`. This is a function‑level attribute; it does not affect the contract semantics of any callees.

```
@ieee_contracts
def sqrt_approx(x: Float<64>) -> Float<64>
    requires x >= 0.0
    ensures abs(codomain * codomain - x) <= 1e-10
{ ... }
```

### Return Value and Path Labels

Within an `ensures` clause, the reserved word `codomain` refers to the value returned by the function. It is implicitly bound and may not be used as an ordinary variable or parameter name anywhere in the program.

To write postconditions that apply only to specific return paths, Posita supports path labels. A path label is an identifier prefixed with `@` that is attached to a `return` statement and then referenced in `ensures` clauses.

**Label on return:**

```
return @label expression
```

**Multiple labels (space‑separated):**

```
return @label1 @label2 expression
```

**Label in ensures:**

```
ensures @label property
```

Here `@label` acts as a placeholder for the value returned on every path marked with that label. The property is a Boolean expression written in terms of `@label` and any visible parameters or global constants. This asserts that each such return value satisfies the property.

A label is visible only within the function where it is used. A label referenced in `ensures` but not present on any `return` causes a compile‑time error. Labels and `codomain` may be freely combined; `ensures codomain > 0` applies to all return paths regardless of labels.

Example (multi‑path constraints):

```
def categorize(x: Int<32>) -> Int<32>
    ensures @even % 2 == 0
    ensures @big > 100
{
    if x < 10 {
        return @even 4;
    } else if x < 20 {
        return @even @big 200;
    } else {
        return @big 300;
    }
}
```

Example (using `codomain` and labels together):

```
def compute(x: Int<32>) -> Int<32>
    ensures codomain >= 0
    ensures @fast < 100
{
    if x < 10 {
        return @fast x * 2;
    } else {
        return x * 10;
    }
}
```

### Contract Qualifiers

Postconditions (`ensures`) can be specialized to apply only to specific exit paths using qualifiers:

- `ensures on Ok(val) => ...`: The postcondition holds only when the function returns `Ok`. The name `val` is bound to the success value.
- `ensures on Err(error) => ...`: The postcondition holds only when the function returns `Err`. The name `error` is bound to the error value.
- `ensures on_timeout => ...` (async only): Holds when the async operation times out.
- `ensures on_cancel => ...` (async only): Holds when the async operation is cancelled before completion.

**Asynchronous contracts:** For `async` functions, the postcondition may be further qualified with `ensures on_timeout => ...` and `ensures on_cancel => ...`. The `on_timeout` clause describes what holds when the operation times out (the runtime scheduler is responsible for triggering this path). The `on_cancel` clause describes what holds when the operation is cancelled before completion. The compiler verifies that the function body respects these guarantees along each corresponding control‑flow path.

To refer to the value of an argument at function entry, use the `old()` function:

```
def increment(x: &mut Int<32>)
    ensures *x == old(*x) + 1
{ *x += 1; }

def divide(a: Int<32>, b: Int<32>) -> Int<32>
    requires b != 0
    ensures a == codomain * b + a % b
{ return a / b; }

def factorial(n: Int<32>) -> Int<32>
    requires n >= 0
    ensures codomain > 0
    terminates n
{
    if n == 0 { return 1; } else { return n * factorial(n - 1); }
}
```

### Deferred Contract Checking

When the `@runtime_check` attribute is applied to a function, its `requires` and `ensures` checks are performed at runtime, even if the arguments are compile‑time constants. This is useful during development or when interfacing with untrusted inputs.

In strict mode, `@runtime_check` is treated as an error: all contracts must be statically proven. The `@runtime_check` attribute is only permitted in non‑strict mode, where it serves as an explicit marker that the developer has chosen to defer verification to runtime.

### Loop Invariants and Termination

Loops may specify an `invariant` and a `decreases` clause. The `decreases` expression must be a non‑negative integer that strictly decreases on each iteration, providing a proof that the loop always terminates.

```
def sum(arr: &[Int<32>]) -> Int<32>
    ensures codomain == fold(arr, 0, +)
{
    set mut total = 0;
    for i in 0..arr'len
        invariant total == fold(arr[0..i], 0, +)
        decreases arr'len - i
    { total += arr[i]; }
    return total;
}
```

Invariants and decreases clauses must appear immediately after the loop header, before the opening brace `{`.

### Built‑in Contract Functions

- `fold(arr, init, op)`: represents the left fold of array `arr` starting from `init` using binary operator `op`. It is a pure function used only in contracts and invariants. Its semantics are: `fold(arr, init, op) == op(...op(op(init, arr[0]), arr[1]), ..., arr[arr'len-1])`.
- `abs(x)`: absolute value of integer `x`.
- `old(expr)`: captures the value of `expr` at function entry, usable in `ensures` clauses.
- `A implies B`: logical implication, equivalent to `not A or B`. The
  expression `A implies B` is true if either `A` is false or `B` is true.
  It is available in all contract contexts and is evaluated in the
  mathematical (ideal) domain, without short‑circuit semantics.
- `forall` and `exists` quantifiers may appear in contracts. They use the colon syntax:

```
requires forall i in 0..arr'len: arr[i] > 0
requires exists i in 0..arr'len: arr[i] == target
```

### Type Invariants (see Type System)

### Ghost Variables (see Type System)

### Generic Constraints with `where`

```
def serialize<T, S>(value: &T, stream: &mut S) -> Result<(), Error>
    where T: Serialize, S: Write, T::Format: Display
{ ... }
```

A trait bound may also be written inline in the parameter list:

```
def identity<T: Copy + Default>(x: T) -> T { return T::default(); }
```

This is syntactic sugar for `where T: Copy + Default`.

A `where` clause may also express type equality constraints. When two type
parameters are equated (`where T == U`), both parameters are treated as
identical within the function body and are exempt from the generality
check (`E104`). Redundant constraints (where the parameters are already
known to be equivalent) produce a compiler warning (`redundant_where_eq`),
not an error.

```
def f<T, U>(x: T, y: U) -> U where T == U { return x; }
```

The `where` clause also supports tuple syntax to apply a multi‑type constraint to a tuple of type parameters:

```
where (T, U): MyConstraint<T, U>
```

This form is equivalent to `where T: MyConstraint<...>, U: MyConstraint<...>` for constraints defined with multiple type parameters. It provides a compact notation when a constraint relationship involves several type parameters simultaneously.

### Reusable Constraint Blocks (`constraint`)

```
constraint SortableContainer<C> { C: Container, C::Item: Ord, C::Item: Default }

def sort<C>(container: &mut C) where C: SortableContainer { ... }
```

Constraints may be combined with `+`: `def f<T>() where T: A + B { ... }`. Constraints may also reference other constraints, forming a hierarchy.

### Generic Parameter Generality

The body of a generic function must type‑check for all possible
instantiations of its type parameters. A type parameter may not be unified
with a concrete type within the function body (unless explicitly constrained
by a `where` clause). Attempting to do so results in compile‑time error
`E104`.

Similarly, two distinct type parameters (e.g., `T` and `U`) may not be
unified with each other inside the function body.

This rule enforces parametricity: the implementation must not depend on the
identity of any type parameter. Type equalities introduced by GADT pattern
matching, which are scoped to a branch and enforced by the GADT context, are
exempt from this check.

Const generic parameters are inherently exempt, as they are monomorphized
for each concrete value.

## Standard Library Initialization Philosophy

Posita avoids implicit conversions between literals and dynamic containers. Compile‑time functions provide ergonomic initialization. The `...args: T` syntax declares a variadic parameter; inside the function `args` is accessible as a `&[T]` slice.

```
comptime def vec<T>(...args: T) -> Vector<T> {
    set mut v = Vector::with_capacity(args'len);
    for a in args { v.push(a); }
    return v;
}

set v = vec!(1, 2, 3);  // Creates Vector<Int<32>> via explicit comptime function
```

## Compiler Diagnostics

The compiler provides three diagnostic levels for contract verification failures:

| Level | Name | Purpose | Output |
|---|---|---|---|
| L1 | Locate | Quickly find the problem | Source file, line, column, violated clause |
| L2 | Explain | Understand the cause | Counterexample in Posita syntax, data‑flow trace |
| L3 | Debug | Deep analysis | SMT‑LIB script, solver statistics, unsat core |

Developers select the level via `capsa build --diagnostic-level=N`. When a contract fails, the compiler attempts to produce a minimal counterexample and a specific suggestion for how to fix it.

**Grade and index diagnostics:** Grade mismatch and index mismatch errors
belong to the E11x error family. At diagnostic level L2, grade errors
include a complete usage-point listing (source location + count per use
site) — this information is already computed during inference and incurs
zero additional cost. Index mismatch errors include the conflicting
equalities from the GADT context or ghost index constraints.

## Undefined Behavior Prevention

| UB Category | Prevention Mechanism |
|---|---|
| Uninitialized variables | Type‑level defaults; `no_default` forces explicit initialization where needed. |
| Integer overflow (signed) | Default `trap`; configurable per‑type/operator; compiler range analysis can eliminate checks. |
| Division by zero | Contract `requires b != 0` enforced at compile time or via runtime panic. |
| Null pointer dereference | References are non‑null; `Option<&T>` enforces checking before use. |
| Dangling pointers | Borrow checker; default copy semantics reduce accidental moves. |
| Data races | `&mut T` is exclusive, `&T` is shared; compile‑time borrow rules. |
| Buffer overflow | Array access checked via contract or runtime bounds; pointer arithmetic constrained to explicit `Ptr` types. |
| Invalid enum values | Type invariants. |
| Misaligned pointers | `Ptr` type carries alignment; safe casts check alignment; `as!` demands proof of alignment. In `unsafe` blocks, alignment is the programmer's responsibility. The standard library provides `from_unaligned`/`write_unaligned` for safe misaligned access. |
| Type punning / transmute | `as!` requires compile‑time verification of layout compatibility. |
| Non‑terminating loops | `decreases` clause on loops proves strict decrease of a non‑negative measure. |
| Non‑terminating recursion | `terminates` clause on functions proves strict decrease of the argument. |
| Infinite loops at compile time | `comptime` execution bounded by step and memory limits. |
| Static initialization order | Compile‑time evaluation ensures constant initializers; module‑level variables use zero‑init or type defaults. |

Strict Mode forbids `unsafe` entirely; all contracts must be statically proven.

## Complete Example

```
edition = "2026";

type AppError = enum { IoError, ValidationError(&[Byte]) }

extern "C" def puts(s: &[Byte]) -> Int<32>;

@trusted
def safe_puts(msg: &[Byte]) -> Result<(), AppError>
    requires msg'len > 0 and msg[msg'len - 1] == 0
    ensures codomain == Ok(())
{
    unsafe { puts(msg); }
    return Ok(());
}

type EmployeeId = UInt<16> with default = 0;
type Salary = Int<32> with default = 0;
type Age = exists n: UInt<8> invariant n >= 18 and n <= 120 with default = 25;
type OwnedFileDescriptor = exists n: Int<32> invariant n >= 0 with no_default;

type UniqueToken = struct { id: EmployeeId }
impl Drop for UniqueToken { def drop(&mut self) { } }

type Department = enum { Engineering, Sales, HumanResources }

type Employee = struct {
    id: EmployeeId, age: Age, salary: Salary, dept: Department,
    name: &[Byte],
}

type PositiveInt = exists n: Int<32> invariant n > 0 with default = 1;

constraint AddableDefault<T> { T: Add<T, Output = T>, T: Default, T: Copy }

def sum_list<T>(items: &[T]) -> T where T: AddableDefault {
    set mut total: T = T::default();
    for item in items { total = total + *item; }
    return total;
}

def calculate_bonus(salary: Salary, multiplier: Int<32>) -> Salary
    requires salary >= 0
    requires multiplier > 0
    ensures codomain >= salary
{
    set mut total = salary;
    set mut i: Int<32> = 0;
    while i < multiplier
        invariant total == salary + i * salary
        invariant i >= 0
        decreases multiplier - i
    { total = total + salary; i = i + 1; }
    return total;
}

def process_employee(emp: &mut Employee, bonus_mult: PositiveInt) -> Result<(), AppError>
    requires emp.salary >= 0
{
    if emp.age > 100 { leave with AppError::ValidationError(b"Employee too old\0               "); }

    safe_puts(b"Processing employee...\0") catch {
        |IoError as e| { log(e); leave with AppError::IoError; }
        |ValidationError(_)| { leave with AppError::IoError; }
    };

    emp.salary = calculate_bonus(emp.salary, bonus_mult);
    return Ok(());
} finally { }

def make_employee_report(emp: &Employee) -> &[Byte] {
    set auto<BonusType> = calculate_bonus(emp.salary, 1);
    comptime {
        set info = @typeInfo!(BonusType);
        match info {
            TypeInfo::Int { bits, .. } => { if bits > 32 { @compile_error!("Bonus type too large"); } }
            _ => {}
        };
    }
    return b"Report generated";
}

def sum_manual<T: Add<T, Output = T> + Default + Copy>(arr: &[T]) -> T {
    set mut total: T = T::default();
    set mut idx: usize = 0;
    while idx < arr'len {
        total = total + *arr[idx];
        idx = idx + 1:usize;
    }
    return total;
}

def main() -> Result<(), AppError> {
    set fd: OwnedFileDescriptor = 3;

    set e1 = Employee { id = 1, age = 25, salary = 50000, dept = Department::Engineering, name = b"Alice" };
    set mut e2 = Employee { id = 2, age = 45, salary = 75000, dept = Department::Sales, name = b"Bob" };

    set bonus_mult: PositiveInt = 1: PositiveInt;
    process_employee(&mut e2, bonus_mult) catch { /* ... */ };

    set salaries: [Salary; 3] = [e1.salary, e2.salary, 60000];
    set total_salary = sum_manual(&salaries);

    safe_puts(b"=== Employee Summary ===\0") catch { |_| {} };

    set e3 = e1;

    set token1 = UniqueToken { id = 10 };
    set token2 = move token1;

    return Ok(());
}
```

## Relationship to Other Languages

- **From Ada:** explicit representation control, attribute syntax, contract‑based verification, default initialization, traceability.
- **From Rust:** `Result`‑based error handling (without type erasure), `if let`, `match`, trait‑like generics, borrow checker.
- **From Zig:** The `comptime` mechanism and the philosophy of moving work to compile time are direct inspirations. Posita adds the `!` call marker and integrates `comptime` with SMT‑based contract verification, going beyond what Zig's comptime offers.
- **From ATS:** The ambition to eliminate runtime errors through static proofs and the practice of encoding invariants in types. ATS2's template system and its removal of GC demonstrate the viability of advanced type systems in resource‑constrained, no‑runtime environments. Posita diverges by separating compile‑time computation (`comptime`) from declarative code generation (`generate`) and replacing explicit proof terms with SMT‑based automation, trading some expressive power for a lower annotation burden and stronger auditability.
- **From F\*, DML, ATS:** erased value indices — types may mention runtime values with all type-level dependency discharged at compile time (F\*/DML) and invariants encoded in types (ATS).

**Unique to Posita**: bit‑width parameterized integers with explicit overflow control, orthogonal pointer sizes, type‑level defaults with invariants and `no_default`, `leave` / `leave with`, type capture, fully static error monomorphization, compile‑time type factories, reflection, structured `finally` blocks, systematic UB elimination, optional strict mode, ghost variables, specification tags, named scope cleanup with compile‑time guards, construction validation, lemma functions, fine‑grained effect annotations, deferred contract checking (`@runtime_check`), layout reflection (`layout_of!`), layout aliases (`layout`), proof hints (`@hint`), fine‑grained error accountability (`@must_handle`), tiered diagnostics, implicit invariant propagation, `old()` expressions, fixed‑precision rationals, MMIO types, interrupt vector generation, `@diverges` for deterministic non‑returning functions, first‑class polymorphism (`poly`/`unbox`) with let‑generalization, GADTs with `when` constraints and existential quantification, affine and linear types (`@linear`), codomain keyword with path labels (`@label`), slice patterns, const generics, TAIT, HRTB, generic associated types (GAT), explicit read‑only borrows (`&ro`), nested GADT refinement rules, graded function types with mechanical propagation, Lease modality, ghost value indices, unified grade lattice (value / ghost / const / contract as four channels), I/O effect aliases, audit rules, floating-point overflow control (trap/ieee/saturate), compile-time floating-point exception detection, Two-Phase Borrows, Point-level liveness, Temporary view freeze, Place-level move semantics (partial move with Full/Partial/Moved tracking for structs, arrays, and enums), `where` type equality constraints, generic parameter generality, and more.

## Design Q&A

**Q: Why "ultra‑static typing"? How is it different from dependent types or refinement types?**

A: Ultra‑static typing resolves all representation details and behavioral guarantees at compile time. Types never require runtime evidence: every type-level dependency — refinement predicates, ghost indices, grades — is discharged or erased at compile time. Types may mention runtime values (as erased indices); they never carry runtime proofs. All checks are provable at compile time in strict mode, leaving zero runtime validation overhead.

**Q: Why copy semantics by default? Doesn't it harm performance?**

A: Copy semantics eliminate moved‑from invalid states, crucial for safety‑critical reasoning. Small types copy at register speed. Large types should use references. Explicit `move` is available as an optimization. For detailed rationale, see the "Move vs. Copy" design note in DESIGN.md.

**Q: How does `with default` differ from Rust's `Default`?**

A: Rust's `Default` is a trait requiring explicit call; Posita's is a type‑level attribute that automatically initializes. Defaults must satisfy type invariants. `with no_default` forces explicit initialization.

**Q: What is the role of `unsafe`? Is Posita "pure safe"?**

A: `unsafe` is confined, requiring `@trusted` with contracts. Strict Mode disallows `unsafe` entirely, making the program pure safe and UB‑free.

**Q: How do you handle C ABI?**

A: `extern "C"` functions are inherently unsafe and require `unsafe` blocks or `@trusted` wrappers with contracts. Standard library provides safe wrappers. Strict Mode can forbid them entirely.

**Q: How do you prevent array out‑of‑bounds?**

A: Compiler proves bounds statically using contracts and invariants; failing proof, it either reports an error (strict mode) or inserts explicit runtime checks (never UB).

**Q: What is the compile‑time performance model?**

A: Fast mode (default) skips SMT proofs; strict mode enables them with time budgets. Incremental compilation and caching keep build times manageable.

**Q: How does the module system work?**

A: File‑based, private by default, no wildcard imports. Dependencies are explicitly declared in `posita.toml`, enabling whole‑program analysis.

**Q: How does Posita support formal verification?**

A: Contracts (`requires`/`ensures`), type invariants, loop invariants are verified by SMT solver. In strict mode, all contracts must be proven.

**Q: What about linear types and usage counts?**

A: Posita provides three complementary mechanisms: `@linear` types forbid implicit discard (see §Affine and Linear Types); `@consume(s)` declares usage grades as part of the function type, propagated mechanically to callers (see §Graded Function Types); and `Lease<T, N>` packages N use-rights for grades greater than 1 (see §Graded Modality: Lease). The usage semiring is selectable per module (see §Usage Semirings).

**Q: Can I hand‑write proofs in Posita?**

A: Posita provides `@lemma` functions to supply auxiliary assertions that assist the SMT solver, and `@comptime_test` blocks to validate `@trusted` code against concrete inputs at compile time. For external formal proofs, `@link_proof` can reference Coq/ATS files that are distributed with the package and verified by `capsa`.

**Q: What are ghost variables?**

A: Ghost variables (`ghost set mut x = ...`) exist only at compile time and are erased from the final binary. They enable complex proofs without any runtime overhead. Ghost variables may appear in `when` guards of `scope_cleanup` blocks to conditionally control cleanup at compile time, but they never affect runtime control flow directly.

**Q: What happens when data comes from an untrusted runtime source (e.g., a JSON file)?**

A: Posita requires explicit parsing and validation at the boundary. Once the data is converted into a strongly‑typed struct, all subsequent code enjoys full static guarantees.

**Q: Does Posita support runtime contract checking?**

A: Yes, the global `--runtime-contracts` flag and the per‑function `@runtime_check` attribute allow deferring contract checks to runtime. This is useful for debugging or when interfacing with untrusted inputs. In strict mode, `@runtime_check` is not permitted.

**Q: How does Posita interact with WCET analysis?**

A: Posita generates highly deterministic code and exports detailed metadata (CFG, loop bounds, etc.) via `--emit-timing-info`. Professional WCET tools like aiT consume this metadata to produce precise timing results.

**Q: How does Posita compare to Moonbit's formal verification?**

A: Moonbit's verification is lighter, targeting cloud/Wasm safety. Posita provides hardware‑level precision (bit‑widths, endianness, interrupt safety) and a complete audit chain from requirements (`@spec`) to object code, suitable for DO‑178C and ISO 26262 certification.

**Q: Does Posita have `if` expressions?**

A: Yes. `if` is an expression that returns a value. All branches must return the same type, and the `else` branch is mandatory.

**Q: Does Posita support ternary operators (`?:`)?**

A: No. The `if` expression already serves this purpose, and `?` is reserved for error propagation.

**Q: What is the `!` symbol used for?**

A: `!` marks calls to `comptime` functions, making compile‑time evaluation explicit at the call site. This is consistent with `?` for error propagation and `await` for asynchronous suspension.

**Q: Can `@trusted` functions call each other?**

A: Yes. The entire trust chain is tracked by `capsa audit`. In strict mode, every `@trusted` function must have either `@link_proof` or `@comptime_test`; otherwise compilation fails.

**Q: How are specification and requirements traced?**

A: Documentation comments with `@spec`, `@requirement`, `@rationale`, and `@trace` link code directly to system requirements. The `capsa spec` command generates a traceability matrix suitable for certification.

**Q: Does Posita support procedural macros?**

A: No. All compile‑time code generation is performed by `comptime` functions, which are type‑checked and sandboxed. This ensures that all code is visible, auditable, and subject to the same safety guarantees.

**Q: How are layout attributes used?**

A: `@packed` removes padding, `@endian` controls byte order, `@bit_order` controls bit field order, `@align` overrides alignment, `@pad` inserts explicit padding, `@layout(C)` forces C ABI compatibility, and `@transparent` guarantees newtype layout identity. Additionally, the `layout` keyword defines named aliases for combinations of these attributes, enabling reuse without hiding detail.

**Q: How do layout aliases work?**

A: A `layout` definition, e.g., `layout Mmio { packed, little_endian; }`, creates a named combination of built‑in layout attributes. It can be applied via `@layout(Mmio)` on a type. The compiler expands the alias at the use site, preserving full auditability. Layout aliases may only contain built‑in attributes; they cannot contain executable logic or reference other aliases.

**Q: Is `@trusted` marked at the call site?**

A: No. `@trusted` is a declaration‑site attribute on function definitions. Call sites remain clean; the trust boundary is established once, at the function definition, and tracked by `capsa audit`.

**Q: What is the difference between `set` and `let`?**

A: `set` is the general variable declaration, defaulting to immutability but allowing `set mut`. `let` is a restricted, always‑immutable form that additionally supports pattern destructuring (tuples, structs, enum variants with mandatory `else`). `let` always requires an explicit initializer and cannot use a type's default value. Use `set` for general purposes; use `let` when you need destructuring or want to enforce immutability at a glance.

**Q: What are the fine‑grained effect annotations?**

A: `@pure`, `@io(read)`, `@io(write)`, `@io`, `@alloc`, `@no_alloc`, `@no_alloc_error`, `@no_panic`, `@diverges`, `@audit_log`, and `@trusted` describe a function's side effects. The compiler checks these annotations for consistency, giving reviewers a precise summary of what a function can do without reading its body.

**Q: How are slices passed to `extern "C"` functions?**

A: `&[T]` and `&mut [T]` are automatically converted to `*const T` and `*mut T` at the ABI boundary, with the length component discarded. This is a deterministic, compiler‑enforced rule; the programmer must ensure the data meets the C function's expectations (e.g., null termination).

**Q: What is `usize`?**

A: `usize` is a built‑in type alias for `UInt<N>` where `N` is the target platform's pointer width. It exists for convenience when writing code that deals with array indices and pointer‑sized values, but explicit `UInt<32>` or `UInt<64>` may always be used instead.

**Q: What happens with `MIN / -1` for signed integers?**

A: This case always traps regardless of the type's overflow policy. It is a representability issue, not a standard overflow. The compiler attempts to statically prove this case unreachable when possible.

**Q: How do I define and implement a trait?**

A: Use `trait TraitName { ... }` to define methods and associated types. Use `impl TraitName for MyType { ... }` to provide implementations. See the "Traits and Implementations" section for details.

**Q: What does `exists` mean in a type definition?**

A: `exists n: BaseType invariant P(n)` introduces a name for the value being constrained. It is required when the invariant expression refers to the value itself. The bound name is erased at runtime.

**Q: How do I use `@apply_lemma`?**

A: Place `@apply_lemma(lemma_fn_name)` on a function definition. The compiler will call the `comptime` lemma function and inject its returned assertions into the SMT solver during verification of that function.

**Q: What are `...` parameters?**

A: The `...args: T` syntax declares a variadic parameter. Inside the function body, `args` is accessible as a `&[T]` slice containing all supplied arguments. This is typically used in `comptime` functions for container initializers.

**Q: How does `move` work?**

A: `move` explicitly transfers ownership of a non‑`Copy` value in assignments, function arguments, or closure captures. After a move, the source variable is invalidated and any subsequent use is a compile‑time error. The compiler will not call `drop` on the moved‑from variable.

**Q: Are `extern "C"` functions safe to call?**

A: No. All `extern "C"` functions are inherently unsafe and can only be called inside `unsafe` blocks or `@trusted` functions. This ensures all FFI calls are auditable.

**Q: How should `catch` patterns be written?**

A: `catch` branches use the enum variant names directly (e.g., `|IoError| { ... }`), without qualifying them with the enum type. The error type is already known from the expression being caught. The `as` keyword can bind the error value to a local variable. A wildcard `|_|` can be used to match all remaining variants. For branches consisting of a single expression, the arrow shorthand `=>` can replace curly braces: `|ParseError => leave with ...`.

**Q: How do I enforce that callers handle specific errors?**

A: Annotate the function with `@must_handle(Variant1, ...)`. The compiler will warn if a caller does not explicitly match or catch those variants. This keeps critical errors visible without forcing exhaustive matching of all variants.

**Q: What is dynamic dispatch and how do I use it?**

A: Dynamic dispatch is available via `dyn Trait` objects, which use a vtable for method calls. Use them explicitly when static dispatch is not possible, such as storing heterogeneous types in a collection. In strict mode, constructing or calling through `dyn Trait` requires a `@trusted` context, because the compiler cannot statically verify the target implementation. This makes runtime dispatch visible to reviewers and ensures it is covered by explicit contracts.

**Q: How do `decreases` and `terminates` work?**

A: `decreases expr` on a loop guarantees termination by requiring `expr` to be a non‑negative integer that strictly decreases each iteration. `terminates arg` on a recursive function requires the specified argument to strictly decrease on each recursive call with a lower bound. Both are used by the compiler to prove termination, which is essential for safety‑critical systems and WCET analysis.

**Q: How do I write integer literals with a specific bit width?**

A: Use the `42i32` suffix for `Int<32>` or `0xFFu8` for `UInt<8>`. This is syntactic sugar for `42: Int<32>` and `0xFF: UInt<8>`, respectively. The type is fully checked by the compiler. For refined types like `PositiveInt`, you can either rely on type inference from the declaration (`set x: PositiveInt = 1;`) or annotate the literal directly (`1: PositiveInt`). Both forms are checked at compile time.

**Q: What is the `isolate` block?**

A: An `isolate` block guarantees that the enclosed code does not access any external mutable state. The compiler verifies this statically, enabling safe concurrent execution from multiple interrupt or task contexts.

**Q: What is conditional compilation (`@cfg`)?**

A: The `@cfg(condition)` attribute allows modules, functions, or type definitions to be conditionally included based on target platform, features, or other compile‑time predicates. Conditions can be combined with `all`, `any`, `not`. In strict mode, all compilation paths must be reachable under some configuration.

**Q: What is `layout_of!`?**

A: `layout_of!(T)` is a compile‑time reflection function that returns a `LayoutDescriptor` detailing the exact size, alignment, and field offsets of type `T`. It is essential for verifying hardware‑facing layouts in `comptime` code.

**Q: How do `@hint` attributes help with proofs?**

A: `@hint(assertion)` provides a suggestion to the SMT solver during contract verification. It guides the solver's search without affecting runtime behavior or safety guarantees. The hint itself must be proven valid (meta‑contract) before being injected.

**Q: What are `@must_handle` and how is it different from `@must_use`?**

A: `@must_use` warns if the entire return value is ignored. `@must_handle` is more specific: it lets a library author name particular error variants that the caller must explicitly handle (via `catch` or `match`), even if other variants are handled by a wildcard. This ensures critical failure modes never slip through silently.

**Q: What is `@exhaustive`?**

A: `@exhaustive` on an enum definition forces all `match`, `if let`, and `while let` on that enum to be exhaustive. This prevents new variants added during evolution from being silently ignored in existing code.

**Q: How do contracts interact with error paths?**

A: By default, `ensures` applies to all exit paths. You can specialize with `ensures on Ok(val) => ...` and `ensures on Err(error) => ...` to make guarantees specific to success or failure returns.

**Q: How does the compiler help me debug contract failures?**

A: The compiler provides three diagnostic levels (L1: locate, L2: explain with counterexamples, L3: full SMT‑LIB dump). Use `capsa build --diagnostic-level=N` to select the depth of information.

**Q: When is `as!` safe to use?**

A: The compiler verifies that source and target types have the same size and alignment, or that a truncation does not violate the target type's `invariant`. All uses of `as!` are flagged for human review by `capsa audit`. For alignment‑unsafe casts involving raw pointers, the operation must be placed in an `unsafe` block.

**Q: How is audit logging handled?**

A: Functions marked `@audit_log` must write contract violations to an immutable audit log. The storage backend is provided by the standard library; tamper‑evident mechanisms (e.g., hash chains) are strongly recommended.

**Q: How are `Rational` values converted to `Float`?**

A: Conversion is explicit (`as Float<64>`) and uses round‑to‑nearest ties‑to‑even by default. This prevents silent precision loss.

**Q: Why can't I use `Regex` types in contracts?**

A: SMT solvers have limited string theory capabilities, making automatic proof unreliable. `Regex` types are for runtime use only; contracts should use simpler predicates.

**Q: What effects do MMIO operations have?**

A: MMIO reads and writes are implicitly `@io(read)` and `@io(write)`, respectively. The compiler ensures they are only used in appropriate effect contexts.

**Q: How are interrupt handlers constrained?**

A: The compiler enforces that interrupt handlers satisfy the constraints of `@no_alloc` and `@no_panic`, and have the return type `!`. They cannot have custom parameters. The compiler generates the interrupt vector table from `@interrupt` annotations.

**Q: Why does Posita have `!` but no `unit` type?**

A: Posita reuses the empty tuple `()` as its unit type. `()` is a regular value that can be constructed and passed around, satisfying generic placeholders. The `!` type is reserved for the true absence of a value—it is uninhabited and signals that a computation never completes normally. This separation keeps the type system orthogonal (no special `unit` keyword) while making the semantics of "no value" explicit.

**Q: Why does Posita have `leave with` and not allow `return Err(...)`?**

A: `leave with` is the only valid error exit in Posita. It provides a dedicated syntactic marker for error exits, making failure paths immediately visible to reviewers. Unlike `return Err(e)`, which can be confused with successful returns during code review, `leave with` cannot be mistaken for anything other than an error exit. This aligns with Posita's commitment to explicit, auditable control flow.

**Q: How does `@ieee_contracts` differ from the old `ensures with ieee_precision` syntax?**

A: The old syntax appeared at the contract level and read like a single postcondition, creating ambiguity about whether it was a clause or a global modifier. The new `@ieee_contracts` attribute is a function‑level annotation that unambiguously switches all `requires` and `ensures` on that function to IEEE 754 semantics. It also cleanly separates semantics control from contract content, and the attribute form makes it visually consistent with other function modifiers like `@pure` and `@no_panic`. It does not affect the contract semantics of callees, keeping the scope clear.

**Q: What is the relationship between `@no_alloc` and `@no_alloc_error`?**

A: `@no_alloc` implies `@no_alloc_error`; a function that never allocates trivially satisfies the no‑allocation‑on‑error constraint. Redundant declaration of both is allowed but not required.

**Q: Can `@no_alloc_error` and `@alloc` be used together?**

A: Yes. This combination means normal execution paths may perform dynamic allocation, but error paths (including all `From` conversions reachable via `?`) must not allocate.

**Q: What happens when `@no_panic` verification fails?**

A: In strict mode, it is a compile‑time error. In non‑strict mode, the compiler emits a warning and may instrument unproven checks with a runtime panic guard.

**Q: Are `@link_proof` and `@comptime_test` required for all `@trusted` functions?**

A: In strict mode, every `@trusted` function must have either `@link_proof` (referencing an external formal proof) or at least one `@comptime_test` exercising its safety contract. If neither is present, compilation fails.

**Q: How are ambiguous variant names in `@must_handle` resolved?**

A: Variant names are resolved against the error type `E` of the function's `Result<_, E>` return type. If a variant name is ambiguous in the current scope, the compiler emits an error and requires explicit qualification using `EnumName::Variant`.

**Q: Does `@interrupt` implicitly add `@no_alloc` and `@no_panic`?**

A: No. The compiler enforces that interrupt handlers satisfy these constraints; it does not inject attributes. Redundant explicit `@no_alloc` or `@no_panic` annotations on an `@interrupt` function are allowed and produce no warning.

**Q: What is `@diverges` and when should I use it?**

A: `@diverges` marks a function that never returns normally, even though its return type is a concrete `T` (not `!`). This is useful for stub implementations that must match a trait signature, eternal watchdogs, or hardware halt routines. The compiler verifies that all paths in the function body diverge deterministically (e.g., `loop {}`). `@diverges` must not be combined with `panic` (use `@no_panic` for non‑panic divergence). It is incompatible with `@runtime_check`.

**Q: Is `leave with` just syntax sugar for `return Err(...)`?**

A: No. `leave with` is a distinct semantic construct that retains its identity throughout the compilation pipeline. Unlike operator desugaring (where `a + b` is rewritten into `Add::add(&a, &b)` in HIR), `leave with` remains as an `ErrorExit` terminator in the control‑flow graph. This distinction enables precise auditing (all error exit points are enumerable without pattern‑matching against `return`), contract verification (`ensures on Err` binds directly to `ErrorExit` nodes), and WCET analysis (error paths and success paths are analyzed separately).

**Q: How does `scope_cleanup` differ from `defer` in other languages?**

A: Unlike anonymous `defer`, Posita's `scope_cleanup` is a named, non‑escaping deferred block. It supports explicit early triggering via `trigger @name`, compile‑time conditional execution via `when`, and its default error‑handling mode forbids `?` to prevent silent error injection. The `propagates` modifier must be explicitly used to allow error propagation, and `overrides` can be used to let cleanup errors take precedence over original errors. Early exits (`return`, `leave with`, etc.) are forbidden inside the block to preserve the LIFO execution guarantee. This design provides both flexibility for fallible cleanup and strong auditability through single‑point declaration.

**Q: How can I conditionally skip a `scope_cleanup` block (e.g., "only on failure")?**

A: Use a ghost variable together with the `when` clause. Declare a ghost variable before the `scope_cleanup`, update it after successful operations, and reference it in the `when` condition. The compiler evaluates the condition at compile time and omits the cleanup block on paths where it is false. Because ghost variables are erased at runtime, this has zero overhead. See the "Structured Resource Cleanup" section for an example.

**Q: How do I combine multiple error types into a reusable signature?**

A: Use an enum set alias with the `|` operator in a `type` declaration. For example, `type AppError = IoError | DbError | ParseError;` creates a named combination that can be used in multiple function signatures. The compiler checks for variant name uniqueness across the combined enums and reports an error if any names conflict. See the "Enum Set Aliases" section for details.

**Q: What is `@auto_deref` and when should I use it?**

A: `@auto_deref` is an attribute placed on a `Deref` implementation that allows method‑call receivers to auto‑dereference through that implementation. Without it, wrapper types require explicit `(*x).method()` syntax. Use `@auto_deref` when your type is designed to be a transparent wrapper (e.g., `Rc<T>`, `Box<T>`). Omit it when the dereference should be explicit (e.g., opaque pointers, newtypes with semantic boundaries). Built‑in references (`&T` / `&mut T`) always auto‑dereference without requiring the attribute.

**Q: How do I use `poly` and `unbox` for first‑class polymorphism?**

A: `poly(expr)` boxes a polymorphic expression (like a generic function) into a `Poly` value. `unbox(poly_value)` instantiates that polytype with fresh type variables. When the result is bound with `set` or `let`, Posita's let‑generalization mechanism automatically generalizes the remaining type variables, making the bound identifier fully polymorphic (e.g., `∀T. T → T`). You can then apply it at multiple different types without additional `unbox` calls. See the "First‑Class Polymorphism" section for examples.

**Q: Does Posita support GADTs?**

A: Yes. Posita supports Generalized Algebraic Data Types (GADTs) where enum variants can carry type equality constraints using the `when` keyword, including existentially quantified type variables introduced with `exists`. This enables type‑safe embedded DSLs and precise type refinement during pattern matching. See the "Generalized Algebraic Data Types" section for full details.

**Q: How do GADTs interact with `with default`?**

A: For generic enums whose variants have `when` constraints involving the type parameters, `with default` is prohibited. For non‑generic enums, `when` constraints using only global constants are allowed. See the GADT section for more.

**Q: What is `codomain` and how does it differ from the old `result`?**

A: `codomain` is the reserved keyword used in `ensures` clauses to refer to the function's return value, replacing an earlier, undocumented use of `result` in contracts. The change was driven not just by the risk of variable‑name collisions but, more importantly, by readability: `result` is an extremely common identifier in business logic, which could cause reviewers to mistake it for a local variable rather than a contract keyword, whereas the mathematical term codomain is rarely used as a variable name and immediately signals "this is the function's output." `codomain` cannot be used as an ordinary variable or parameter name anywhere in the program. For postconditions that need to distinguish among different return paths, `@label` annotations on `return` statements can be combined with `ensures @label property`; these work alongside `codomain` and do not replace it.

**Q: How do I write different postconditions for different return paths without using an enum?**

A: Use path labels. Attach `@label` to a `return` statement (e.g., `return @fast x * 2;`) and write `ensures @fast < 100` in the contract. Multiple labels can be assigned to a single return, and a label can be used on several returns. Labels are scoped to the function and checked for consistency. See the "Return Value and Path Labels" section for examples.

**Q: What are affine and linear types in Posita?**

A: All non‑`Copy` types are affine: they can be moved or discarded, but not duplicated. `@linear` types are a stricter subset that also forbid implicit discarding—they must be explicitly consumed (e.g., by passing to a consumer function or calling `forget`). This provides fine‑grained control over resources that must never be silently dropped. See the "Affine and Linear Types" section for details.

**Q: How do I pass a `&mut T` to a function expecting `&T`?**

A: Use `&ro r` to explicitly create a read‑only reference, or apply `@auto_ro` to the enclosing function or module. In method chains, prefer `.freeze!()`. See the "Reference Coercion and Read-Only Borrows" section for details.

**Q: Can I use `impl Trait` or `dyn Trait` in an enum set alias?**

A: No. Enum set aliases (`A | B`) must be composed from concrete enum type names so that the variant set is statically known. `impl Trait` and `dyn Trait` are rejected in this context.

**Q: How does `as!` handle alignment mismatches?**

A: The compiler rejects `as!` if the target alignment is stricter than the source and the compiler cannot prove the pointer is correctly aligned. If the source is a raw pointer, the entire operation can be placed inside an `unsafe` block, where the programmer assumes responsibility for alignment. Safe alternatives like `from_unaligned` are available for misaligned data.

**Q: How do grades differ from contracts?**

A: Grades express *counts* as semiring algebra on function types and propagate to callers mechanically. Contracts express resource *semantics* (state before/after) and are proved by SMT. Contracts must not restate what a grade already expresses.

**Q: Does Posita have dependent types?**

A: Posita has *erased* dependent indexing (F\*, DML, ATS lineage): types may mention runtime values as ghost indices; all type-level dependency is erased, never shapes layout, and never requires runtime evidence.

**Q: How do `const N` and `ghost n` differ?**

A: `const` parameters are compile-time known, shape layout, and monomorphize. Ghost indices are runtime values used only for reasoning: no layout impact, one shared code body.

**Q: Is grade inference "implicit" in the bad sense?**

A: No. Inference is analysis (facts, never entering signatures); declaration is commitment (enters the type, constrains callers). Same relation as type inference vs. type annotations.
