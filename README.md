# 🔷 Nova DSL Compiler

> **A complete, production-quality compiler for a custom Domain-Specific Language — written entirely in C, targeting native binaries via LLVM.**

[![Build](https://img.shields.io/badge/build-passing-brightgreen)](#)
[![Language](https://img.shields.io/badge/language-C11-blue)](#)
[![Backend](https://img.shields.io/badge/backend-LLVM-orange)](#)
[![License](https://img.shields.io/badge/license-MIT-green)](#)

---

## Architecture Overview

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Source     │    │    Lexer    │    │   Parser    │    │   Sema      │    │  CodeGen    │
│  .nova file  │───▶│  lexer.c   │───▶│  parser.c  │───▶│  sema.c    │───▶│ codegen.c   │
│             │    │  Token stream│    │  AST nodes  │    │  Type check │    │  LLVM IR    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                                                                     │
                                                                              ┌──────▼──────┐
                                                                              │    llc      │
                                                                              │  Assembly   │
                                                                              └──────┬──────┘
                                                                                     │
                                                                              ┌──────▼──────┐
                                                                              │ clang/gcc   │
                                                                              │ Native ELF  │
                                                                              └─────────────┘
```

## Pipeline Stages

| Stage | File | Description |
|-------|------|-------------|
| **Lexer** | `src/lexer.c` | Tokenises source into a stream of `Token` structs. Handles hex/binary literals, escape sequences, block comments, keyword recognition. |
| **Parser** | `src/parser.c` | Recursive-descent parser building a typed AST. Implements Pratt/precedence-climbing for expressions. |
| **AST** | `src/ast.c` | All node types + arena allocator. Zero heap fragmentation — entire compilation uses a single arena. |
| **Sema** | `src/sema.c` | Two-pass semantic analyser: first pass registers all top-level symbols; second pass type-checks all function bodies. Hash-table scopes. |
| **CodeGen** | `src/codegen.c` | Walks annotated AST and emits textual LLVM IR ready for `llc`. Handles SSA registers, stack allocation, control flow, type coercion. |
| **Diag** | `src/diag.c` | Colour-coded, location-aware error/warning messages. Auto-disables colour when not a TTY. |

---

## Nova Language Reference

### Types

| Nova   | LLVM IR  | Description             |
|--------|----------|-------------------------|
| `i8`   | `i8`     | 8-bit signed integer    |
| `i16`  | `i16`    | 16-bit signed integer   |
| `i32`  | `i32`    | 32-bit signed integer   |
| `i64`  | `i64`    | 64-bit signed integer   |
| `u8`   | `i8`     | 8-bit unsigned integer  |
| `u32`  | `i32`    | 32-bit unsigned integer |
| `u64`  | `i64`    | 64-bit unsigned integer |
| `f32`  | `float`  | 32-bit float            |
| `f64`  | `double` | 64-bit float            |
| `bool` | `i1`     | Boolean                 |
| `str`  | `i8*`    | String literal pointer  |
| `void` | `void`   | No value                |
| `*T`   | `T*`     | Pointer to T            |
| `[N]T` | `[N x T]`| Fixed-size array        |

### Syntax at a Glance

```nova
// Functions
fn add(a: i64, b: i64) -> i64 {
    return a + b;
}

// Variables
let x: i64 = 42;
let mut counter: i64 = 0;

// Control flow
if x > 0 {
    puts("positive");
} else if x == 0 {
    puts("zero");
} else {
    puts("negative");
}

// Loops
while counter < 10 {
    counter = counter + 1;
}

for i in 0..100 {
    // i goes from 0 to 99
}

// Extern functions (call into libc)
extern fn printf(fmt: str, val: i64) -> i32;

// Structs
struct Point {
    x: i64,
    y: i64,
}

// Enums
enum Color {
    Red,
    Green,
    Blue,
}

// Type casting
let f: f64 = 3;
let n: i64 = f as i64;
```

### Operators

| Category   | Operators                          |
|------------|------------------------------------|
| Arithmetic | `+ - * / %`                        |
| Bitwise    | `& \| ^ ~ << >>`                   |
| Logical    | `&& \|\| !`                        |
| Comparison | `== != < > <= >=`                  |
| Assignment | `= += -= *= /=`                    |
| Other      | `as` (cast), `..` (range), `->` (return type) |

---

## Quick Start

### Prerequisites

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install gcc llvm clang make

# macOS
brew install llvm gcc
```

### Build

```bash
git clone https://github.com/yourname/nova-lang
cd nova-lang
make
```

### Compile a Nova Program

```bash
# Full pipeline: .nova → native binary
./novac examples/showcase.nova -o showcase
./showcase

# Emit LLVM IR only
./novac --emit-ir examples/showcase.nova -o showcase.ll
cat showcase.ll

# Emit assembly only
./novac --emit-asm examples/showcase.nova -o showcase.s

# Dump AST
./novac --ast examples/showcase.nova

# Dump token stream
./novac --tokens examples/showcase.nova

# With optimization
./novac -O2 examples/showcase.nova -o showcase_fast

# Verbose mode
./novac -v examples/showcase.nova
```

### Run Tests

```bash
make test
```

Expected output:
```
══════════════════════════════════════════
  Nova Compiler — Test Suite
══════════════════════════════════════════
  hello                PASS (IR)
  arith                PASS (IR)
  control              PASS (IR)
  fib                  PASS (IR)
  structs              PASS (IR)
  loops                PASS (IR)
  strings              PASS (IR)

  Passed: 7  Failed: 0
══════════════════════════════════════════
```

---

## VS Code Setup

Install these extensions:
- **C/C++** (Microsoft) — IntelliSense, debugging
- **clangd** — Code completion
- **CodeLLDB** — LLDB debugger integration

Build & debug config is in `.vscode/` (included in the repo).

---

## Project Structure

```
nova-lang/
├── include/          # Public headers
│   ├── lexer.h       # Token types, Lexer state
│   ├── ast.h         # ASTNode, Type, Arena
│   ├── parser.h      # Parser state
│   ├── sema.h        # Sema, Scope, Symbol
│   ├── codegen.h     # CodeGen state
│   └── diag.h        # Diagnostic helpers
├── src/              # Implementation
│   ├── lexer.c
│   ├── ast.c
│   ├── parser.c
│   ├── sema.c
│   ├── codegen.c
│   ├── diag.c
│   └── main.c        # Compiler driver
├── tests/            # .nova test programs
├── examples/         # Showcase programs
├── docs/             # Extended documentation
├── scripts/          # Helper scripts
├── Makefile
└── README.md
```

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Add tests in `tests/`
4. Run `make test`
5. Submit a pull request

### Roadmap

- [ ] LLVM C API integration (replace textual IR)
- [ ] Closures / first-class functions
- [ ] Generics / parametric types
- [ ] Module system
- [ ] Standard library (nova-std)
- [ ] Self-hosting (compile Nova with Nova)
- [ ] WASM backend
- [ ] LSP server for IDE integration

---

## License

MIT License — see [LICENSE](LICENSE).

---

*Built with ❤️ in C — because the best compilers are written close to the metal.*
