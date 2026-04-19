---
name: pike-testing
description: Writing and running Pike tests using the m4-based test DSL and test infrastructure

globs:
  - "**/testsuite.in"
  - "**/test_*"
metadata:
  version: "1.0.0"
  organization: pike-lang
---

# Pike Testing

## Overview
Pike uses an m4-based test DSL for its test suite. Tests are written in `testsuite.in` files, preprocessed by m4, and executed by the Pike test harness. This skill covers the test macros, file structure, and common patterns.

## Key Concepts

- Tests are m4 macros, not Pike code directly — they expand into Pike test harness calls.
- `[[...]]` delimiters are m4 quoting; they prevent premature expansion of Pike syntax that collides with m4.
- Test files live alongside module source and are collected into a unified testsuite at build time.
- `make verify` from the repo root runs the full suite.

## Rules

### 1. Use test_eq for equality assertions

`test_eq(expr1, expr2)` asserts two expressions produce equal values.

```pike
// WRONG — manual assert with verbose output
if (1+1 != 2) { werror("failed: 1+1 != 2\n"); errors++; }

// CORRECT — concise equality assertion
test_eq(1+1, 2)
test_eq("hello"*2, "hellohello")
test_eq(({1,2,3})[-1], 3)
```

### 2. Use test_any for expression evaluation

`test_any([[code]], expected)` evaluates a Pike code block and compares its return value against expected.

```pike
// WRONG — using test_do and manual comparison
test_do([[if (sizeof(({1,2,3})) != 3) error("bad size");]])

// CORRECT — return a value, let the macro assert it
test_any([[return ({1,2,3})[-1];]], 3)
test_any([[string s = "abc"; return s[1];]], 'b')
```

### 3. Use test_do for statement execution

`test_do([[code]])` executes code without checking a result. Use it for side-effect validation or setup.

```pike
// WRONG — wrapping a statement in test_eq with a dummy value
test_eq(({int x = 42; x += 8;}), 0)

// CORRECT — execute statements, rely on runtime errors for failures
test_do([[int x = 42; x += 8;]])
test_do([[object buf = String.Buffer(); buf->add("data");]])
```

### 4. Use test_compile for compile-time checks

`test_compile([[code]])` asserts the code compiles without errors. Use `test_compile_error` for the inverse.

```pike
// WRONG — using test_do to check compilability (catches errors too late)
test_do([[class Foo { int x; void create(int y) { x=y; } }]])

// CORRECT — verify compilation explicitly
test_compile([[class Foo { int x; void create(int y) { x=y; } }]])
test_compile_error([[int x = "string";]])
```

### 5. Use test_eval_error for expected errors

`test_eval_error([[code]])` asserts the code throws a runtime error.

```pike
// WRONG — manually catching and checking
test_do([[mixed err = catch { error("boom"); }; if (!err) error("expected error");]])

// CORRECT — let the macro assert that an error is thrown
test_eval_error([[ error("boom"); ]])
test_eval_error([[ ({})[100] ]])
```

### 6. Use test_true/test_false for boolean checks

`test_true(expr)` asserts the expression is truthy. `test_false(expr)` asserts falsy.

```pike
// WRONG — using test_eq against 1 or 0
test_eq(zero_type((["a":0])["b"]), 1)

// CORRECT — use boolean assertions for clarity
test_true(zero_type((["a":0])["b"]))
test_false(zero_type((["a":0])["a"]))
test_true(file_stat("/usr/bin/env"))
test_false(0)
```

### 7. Use [[...]] delimiters for code blocks in tests

Pike syntax (commas, parentheses, semicolons) collides with m4 argument parsing. Wrap multi-statement or complex code in `[[...]]`.

```pike
// WRONG — unquoted code breaks m4 argument parsing
test_any(return ({1,2,3})[-1];, 3)

// CORRECT — [[...]] prevents m4 from splitting on commas inside Pike aggregates
test_any([[return ({1,2,3})[-1];]], 3)
test_do([[array a = ({1,2,3}); a += ({4});]])
```

### 8. Run tests via make verify from repo root

The standard entry point for the full test suite is `make verify` from the Pike repository root.

```bash
# WRONG — running individual test files without preprocessing
pike testsuite.in

# CORRECT — use the make target (preprocesses, builds testsuites, runs them)
make verify

# Also available:
make verbose_verify    # with test output
make gdb_verify        # under GDB
make valgrind_verify   # under valgrind
make testsuites        # build test files without running
```
