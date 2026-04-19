# Pike Programming Language

> This file guides AI coding agents working on the Pike codebase.

## Project Overview

Pike is a dynamic, interpreted, object-oriented language with C-like syntax. Version 8.0.1116, triple-licensed under GPL, LGPL, and MPL.

- C interpreter/compiler in `src/` (bytecode VM, garbage collector, compiler frontend/backend)
- Pure Pike standard library in `lib/modules/` (`.pmod` files)
- Native C modules in `src/modules/`
- Optional post-build modules in `src/post_modules/`

## Build Commands

From repo root — the meta-Makefile handles everything.

```bash
# First time (generates configure)
cd src && ./run_autoconfig . && cd .. && make

# Configure with options
make CONFIGUREARGS="--prefix=/usr/local"

# Build (output goes to build/<os-arch>/)
make

# Parallel build
make MAKE_PARALLEL="-j$(nproc)"

# Clean
make spotless

# Install
make install
```

The built binary is at `build/<os-arch>/pike` or via the `bin/pike` wrapper.

## Running Pike

```bash
bin/pike                  # Interactive REPL (hilfe mode)
bin/pike --version        # Print version
bin/pike script.pike      # Run a script
bin/pike -c script.pike   # Syntax check only
```

## Running Tests

```bash
make verify               # Full test suite (from repo root)
make check                # Alternative target
make testsuites           # Build module testsuites only
```

Test DSL uses m4 macros: `test_eq`, `test_any`, `test_do`, `test_compile`, `test_eval_error`. Test files are named `testsuite.in` and live in module directories.

## Code Style — C (`src/`)

- Tab indentation
- Public API: `PMOD_EXPORT` prefix
- Module init/exit: `PIKE_MODULE_INIT` / `PIKE_MODULE_EXIT`
- Value construction: `push_text()`, `push_int()`, `push_array()`, etc.
- Svalue stack: args at `Pike_sp[-args]` through `Pike_sp[-1]`
- String handling: `make_shared_string()`, `free_string()`, `BEGIN_MANIP` / `END_MANIP`
- GC safety: `add_ref()` / `free_svalue()` correctly
- Module files use `.cmod` extension with `PIKECLASS` / `PIKEFUN` macros

## Code Style — Pike (`lib/modules/`)

- Tab indentation
- `.pmod` files are modules (analogous to `.py` in Python)
- `.pmod` directories form module trees with `module.pmod` or direct `.pmod` files
- Autodoc: `//!` prefix for documentation comments
- Types encouraged: `int|string`, `array(string)`, `mapping(string:mixed)`
- No semicolons after class/method bodies (C-like brace style)
- Constructor: `create()`, destructor: `destroy()`
- Inheritance: `inherit ParentClass;`, parent call: `::create(args);`

## Architecture Overview

```
Pike/
├── src/              # C interpreter core (compiler, VM, GC, builtins)
├── src/modules/      # C native modules (Stdio, Image, Mysql, Parser, etc.)
├── src/post_modules/ # Optional native modules (GL, GTK, SQLite, JSON, etc.)
├── lib/modules/      # Pure Pike stdlib (ADT, Calendar, Crypto, Protocols, etc.)
├── lib/master.pike.in # Master object (module resolver, autoloader)
├── refdoc/           # Reference documentation sources
├── bin/              # Build wrappers (bin/pike)
└── build/            # Build output directory (per-OS-arch)
```

## Standard Library Coverage

Detailed patterns in `.agents/skills/pike-language-reference/references/stdlib-patterns.md` (1200+ lines). Key modules covered:

- **File I/O**: Stdio.File, Stdio.read_file, Stdio.Buffer, Stdio.Port
- **HTTP**: Protocols.HTTP (sync/async), Protocols.HTTP.Server
- **Database**: Sql.Sql (MySQL, PostgreSQL, SQLite)
- **Async**: Concurrent.Future/Promise
- **Crypto**: SHA256/SHA512/AES/RSA/HMAC/Password
- **TLS**: SSL.File, SSL.Context, SSL.Port
- **Parsing**: Parser.HTML (callback-driven), Regexp
- **Data Structures**: ADT.Table, ADT.Stack, ADT.Queue, ADT.Heap, ADT.CritBit
- **Serialization**: encode_value/decode_value, Serializer.Encodeable, Standards.JSON
- **Math**: Math.inf, Math.nan, Math.Angle
- **Error Handling**: Error.Generic and subclasses
- **CLI**: Getopt.find_option/find_all_options
- **Process**: Process.run, Process.create_process, Process.spawn
- **Image**: Image.decode, Image.load, Image.Image
- **More**: Filesystem, MIME, Colors, Standards (URI, UUID, BASE64), Debug


## Critical Pike Semantics (for Code Generation)

These are the most common traps when generating Pike code. Violating any of these produces code that compiles but behaves incorrectly.

| Pike idiom | Do NOT write |
|---|---|
| `({1, 2, 3})` | `[1, 2, 3]` |
| `(["key": "val"])` | `{"key": "val"}` |
| `(< "a", "b" >)` | `{"a", "b"}` |

- Strings are **immutable** — use `String.Buffer` for mutability
- Array `==` checks **identity** not value — use `equal()` for structural comparison
- Int division rounds toward minus infinity: `-8/3 == -3`
- Constructor is `create()`, **not** `__init__` or `new`
- Inheritance keyword: `inherit`, **not** `extends`
- Parent access: `::method()`, **not** `super`
- `foreach(arr; idx; val)` gets both index and value
- `switch` cases fall through like C/Java — use `break` to prevent
- `sprintf` supports `%O` for inspect (readable representation of any value)
- `sscanf` returns match count; values assigned via lvalue references
- `zero_type(m[key]) == 1` distinguishes missing key from key mapping to `0`
- Bignum auto-promotion on overflow is transparent
- `mixed` accepts everything; `void` means no return value or optional argument
- `Val.true`, `Val.false`, `Val.null` are singleton objects

## Commit Guidelines

- Prefix with module name: `[Stdio] Fix buffer overflow in read()`
- Reference issues: `Fixes #12345` or `References #12345`
- One logical change per commit

## Security

- No unchecked buffer operations in C code
- Respect svalue reference counting
- Module code runs with interpreter privileges
- Use `add_ref()` / `free_svalue()` correctly for GC safety
