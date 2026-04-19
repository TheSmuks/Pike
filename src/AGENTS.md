# Pike C Source — AGENTS.md

## Overview
This directory contains the Pike interpreter core written in C:
- Bytecode compiler (compiler.c, program.c, las.c, tree.c)
- Stack-based VM (interpret.c, interpret_functions.h)
- Garbage collector (gc.c, gc_header.h)
- Built-in types (stralloc.c for strings, array.c, mapping.c, multiset.c, svalue.c)
- Bignum support (bignum.c)
- Module system (module.c, dynamic_module.c)
- Error handling (error.c)
- Preprocessor (preprocessor.h)
- Backend/event loop (backend.c)
- Threading support (threads.c, threadlib.h)

## Build
- Autoconf-based: `./run_autoconfig . && ./configure && make`
- Build happens in repo root via meta-Makefile
- The configure script is in `src/configure.in`

## C Code Style
- Tab indentation throughout
- PMOD_EXPORT for public API functions
- PIKE_MODULE_INIT / PIKE_MODULE_EXIT for module lifecycle
- Pike_sp stack: push results, pop args
  - Arguments at Pike_sp[-args] through Pike_sp[-1]
  - Push results: push_text(), push_int(), push_array(), etc.
  - RETURN macro for PIKEFUN
- String handling: make_shared_string(), free_string()
- Reference counting: add_ref(), free_svalue(), EXIT_SVALUE(),
- Type macros: PIKE_T_INT, PIKE_T_STRING, PIKE_T_ARRAY, etc.
- Memory: dmalloc debugging available (--with-dmalloc configure option)

## Key Source Files
| File | Purpose |
|---|---|
| interpret.c | Bytecode interpreter main loop |
| interpret_functions.h | OPCODE definitions (F_ADD, F_CALL, etc.) |
| compiler.c | Pike-to-bytecode compiler |
| program.c | Program/class management |
| object.c | Object instances |
| svalue.c | Svalue (tagged union) operations |
| stralloc.c | String allocation and manipulation |
| array.c | Array type implementation |
| mapping.c | Mapping (hash table) implementation |
| gc.c | Garbage collector |
| error.c | Error/exception handling |
| backend.c | Event loop (file I/O, signals, callbacks) |
| threads.c | POSIX thread wrapper |
| las.c | Low-level AST for compiler |
| tree.c | Syntax tree nodes |
| preprocessor.h | Macro preprocessor |
| opcodes.c | Bytecode opcode definitions |
| module.c | Dynamic module loading |
| bignum.c | Bignum auto-promotion |

## Svalue Type
The svalue (struct svalue_s) is Pike's universal value type:
- .type: PIKE_T_INT, PIKE_T_STRING, PIKE_T_ARRAY, PIKE_T_MAPPING, etc.
- .u: union holding the actual value pointer
- .subtype: used for PIKE_T_INT (NUMBER_UNDEFINED, NUMBER_NUMBER)
- Integers stored inline in .u.integer (not boxed)

## GC Rules
- All reference-counted types: strings, arrays, mappings, objects, programs, functions
- Use add_ref() when storing a reference
- Use free_svalue() when releasing a reference
- GC cycles are detected by the mark-and-sweep collector
- THREADS_ALLOW / THREADS_DISALLOW for releasing interpreter lock

## Testing
- `make verify` from repo root
- Individual module tests: look for testsuite.in

## Security
- Never use unchecked memcpy/sprintf — use Pike's safe alternatives
- Respect reference counting — every add_ref needs a matching free
- Buffer operations must check bounds
