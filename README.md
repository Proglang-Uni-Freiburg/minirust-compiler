# Mini Rust Compiler

A compiler for **MiniRust** (`.mrs` files) — a small subset of Rust — targeting
**RISC-V 64-bit assembly**. Built for the Compiler Construction course at the
University of Freiburg, following Appel's *Modern Compiler Implementation in
ML*.

It builds on course codebases:
- The [Compiler Construction template repository](https://github.com/Compiler-Construction-Uni-Freiburg/compiler-construction-uni-freiburg-2025-compiler-2025-template-2025) — project structure, build system, RISC-V/QEMU environment.
- The [MiniRust reference implementation](https://github.com/Proglang-Uni-Freiburg/minirust) — initial parser and type-checking infrastructure.

The compiler extends these foundations with a full pipeline including IR lowering, canonicalization, and RISC-V code generation.

---

## Requirements

Only Docker is required. The container provides everything else:
- Rust (nightly toolchain, required for `box_patterns`)
- RISC-V GCC cross-compiler
- QEMU for running generated binaries

---

## Usage

All commands run through the `./do` helper script (Docker by default; add `--no-docker` / `-nd` to run locally):

```bash
./do compile examples/fib.mrs          # compile a .mrs file to RISC-V assembly
./do run examples/fib.mrs              # compile, link with GCC, and run via QEMU
./do test                              # run the integration test suite
./do --release run examples/fib.mrs   # release mode
./do clean                             # clean output
./do shell                             # open a shell in the Docker container
./do docker-rebuild                    # rebuild the Docker image
```

Compiled assembly and binaries are placed in `tmp/`.

Under the hood:
- **Compile**: `cargo run -- -i <src> -o tmp/<name>.S`
- **Run**: compile → `riscv64-linux-gnu-gcc -static <asm> runtime/runtime.c -o <bin>` → `qemu-riscv64-static <bin>`
- **Test**: `cargo test --test test -- --nocapture`

Binary flags: `-i/--src` (required), `-o/--tgt` (defaults to stdout), `-v/--verbose` (prints output of every pass).

---

## The MiniRust Language

MiniRust is **fully expression-based**: every construct evaluates to a value; there are no statements.

**Types:** `()`, `bool`, `i64`, `fn(T...) -> T`

**Expressions** (`expr` rule): integer/bool/unit literals, binary ops (`+ - * / > >= < <= == !=`), `if/else` (both branches required unless the type is `()`), function calls, block expressions `{ body }`, identifiers.

**Body context** (`body` rule — inside `{}`): extends `expr` with sequencing constructs that require a continuation:
- `let x = e1; e2` — binds `x` in scope of `e2`; evaluates to `e2`. The semicolon is mandatory. Missing continuation defaults to `()`.
- `fn name(...) { ... }` followed by a newline and continuation — inline function definition; evaluates to the continuation. Missing continuation defaults to `()`.
- `e1; e2` — evaluates `e1` for side effects, then evaluates to `e2`.

**Top-level:** only `fn` declarations. Every program must have exactly one `main() -> ()`.

**I/O:** `println!("{}", expr)` is recognized as a literal token in the grammar (not a real macro) and compiled directly to a `print_int64` call into `runtime/runtime.c`. This is a deliberate hack to keep MiniRust a valid Rust subset.

---

## Architecture

The pipeline is strictly sequential with no optimization passes. `lower()` in `main.rs` calls `semant::check`, then passes the resulting fragments to `codegen::select` and `codegen::emit`.

```
Source text (.mrs)
    │
    ▼ [PARSE]  src/parse/grammar.rs — PEG grammar (peg crate)
AST: Vec<Tag<Top>>  (src/ast/)
    │
    ▼ [SEMANT + IR LOWER + CANONICALIZE]  src/semant/mod.rs
    │  Type-checking, IR tree construction, and canonicalization are one combined
    │  pass. Canonicalization (src/ir/canonical.rs) is triggered implicitly inside
    │  FunContext::exit in src/ir/translate.rs at the end of each function.
IR Program: Vec<Fragment>  — Fragment::Proc { label, body: Vec<Stmt>, frame }
    │
    ▼ [INSTRUCTION SELECTION]  src/codegen/lower.rs — maximal munch Muncher
Vec<Instr>
    │
    ▼ [EMIT]  codegen::emit() — render Instr to text
RISC-V assembly (.S)
    │
    ▼ [EXTERNAL]  GCC cross-compiler + runtime/runtime.c → ELF → QEMU
```

### Key Design Decisions

**No register allocator.** Every IR `Temp` is immediately spilled to a frame-relative stack slot via `Frame::get_offset`. ABI registers (FP=0, SP=1, RV=2, ARG0–ARG7=3–10) are pre-defined Temp IDs; user temps start at 11.

**Translate tri-form (Appel).** `translate::Expr` has three variants: `Ex(ir::Expr)` (value), `Nx(ir::Stmt)` (no value), `Cx(closure → CJump)` (conditional). `un_ex/un_nx/un_cx` coerce between forms. `if` expressions and boolean conditions use the `Cx` form.

**Type-check and IR-lower are the same pass.** `semant::check_expr` simultaneously verifies types and builds IR trees, returning `TypedExpr { ir: translate::Expr, ty: Tag<Type> }`.

**AST uses `Tag<T>` pervasively.** `Tag<T>` wraps any value with a byte-offset span, enabling rustc-style annotated error messages via `annotate-snippets`.

### Module Map

| Path | Responsibility |
|---|---|
| `src/ast/` | AST node types, `Tag<T>` span wrapper, error types, pretty-printer |
| `src/parse/grammar.rs` | Full PEG grammar; handles precedence, comments, `println!` |
| `src/semant/mod.rs` | `check()`: walks AST, type-checks, emits IR; `check_expr` is the core |
| `src/semant/env.rs` | Two-layer environment (`globals` + `locals`); `Binding` is `Var(Access)` or `Fun(Label)` |
| `src/ir/mod.rs` | IR tree types: `Expr`, `Stmt`, `Fragment`, `Program` |
| `src/ir/symbols.rs` | `Label` (string-based) and `Temp` (integer ID) with auto-generation |
| `src/ir/frame.rs` | `Frame`: stack layout, FP-relative slot assignment, ABI parameter mapping |
| `src/ir/translate.rs` | IR builder helpers + `FunContext` (function entry/exit, canonicalization trigger) |
| `src/ir/canonical.rs` | Appel canonicalization: hoist `ESeq`, linearize `Seq` nesting |
| `src/codegen/lower.rs` | `Muncher`: maximal-munch instruction selection over IR trees |
| `src/codegen/mod.rs` | `Instr` enum, `Register` enum, `select()`, `emit()`, prologue/epilogue |
| `runtime/runtime.c` | C runtime: `print_int64(int64_t)` linked into every compiled program |

---

## Tests

`tests/test.rs` runs a differential test suite against real `rustc` as the oracle:
1. Discovers all `.mrs` files in `tests/suite/`.
2. For each: compiles with real `rustc`, captures expected output.
3. Compiles with `minirust_compiler` → GCC (RISC-V) → QEMU.
4. Asserts actual output matches the oracle.
