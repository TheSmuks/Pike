# Pike Test DSL Reference

## Test Macros (m4-based)

### test_eq(expr1, expr2)
Assert two expressions produce equal values.

```pike
test_eq(1+1, 2)
test_eq("hello"*2, "hellohello")
test_eq(sizeof(({1,2,3})), 3)
```

### test_true(expr)
Assert expression evaluates to a truthy value.

```pike
test_true(1)
test_true(zero_type((["a":0])["b"]))
test_true(file_stat("/bin/sh"))
```

### test_false(expr)
Assert expression evaluates to a falsy value (0, Val.null, or absent mapping entry with zero_type 1 — though zero_type returning 1 is truthy; use test_false on the mapping lookup itself for the "key present, value is 0" case).

```pike
test_false(0)
test_false(zero_type((["a":0])["a"]))
```

### test_any([[code]], expected)
Evaluate a Pike code block (must `return` a value) and compare against expected.

```pike
test_any([[return ({1,2,3})[-1];]], 3)
test_any([[string s = "abc"; return s[1];]], 'b')
test_any([[mapping m = (["x":1, "y":2]); return m["x"] + m["y"];]], 3)
```

### Deep Equality Macros

```pike
// test_equal — structural/deep equality (uses equal())
test_equal(({1,2,3}), ({1,2,3}))  // PASS — same structure
test_equal(([1:2]), ([1:2]))      // PASS — same structure

// test_any_equal — evaluate code and compare structurally
test_any_equal([[array a = ({1,2}); a += ({3}); return a;]], ({1,2,3}))

// Key difference:
// test_eq uses == (identity comparison)
// test_equal uses equal() (structural/deep comparison)
// Use test_equal for arrays, mappings, multisets, objects
```

### test_do([[code]])
Execute code with no result check. Fails only on compilation or runtime error.

```pike
test_do([[int x = 42; x += 8;]])
test_do([[object buf = String.Buffer(); buf->add("data");]])
test_do([[add_constant("TEST_VAL", 99);]])
```

### test_compile([[code]])
Assert code compiles without errors.

```pike
test_compile([[class Foo { int x; void create(int y) { x=y; } }]])
test_compile([[int(0..255) x = 128;]])
```

### test_compile_error([[code]])
Assert code fails to compile (type errors, syntax errors, etc.).

```pike
test_compile_error([[int x = "string";]])
test_compile_error([[void f(int x) { return x; }]])
```

### test_eval_error([[code]])
Assert code throws a runtime error.

```pike
test_eval_error([[ error("boom"); ]])
test_eval_error([[ ({})[100] ]])
test_eval_error([[ 1/0; ]])
```

### cond([[condition]], [[test ...]])
Conditionally run tests based on a Pike expression (e.g., feature availability).

```pike
cond([[constant Thread.Mutex]], [[
  test_do([[object m = Thread.Mutex();]])
  test_true(functionp(Thread.Mutex()->lock))
]])
```

## Test File Structure

- Tests live in `testsuite.in` files throughout the source tree.
- These are m4 preprocessed into `testsuite` files.
- Module-specific test directories: look under each module's source directory.
- `bin/mktestsuite` generates test suites from `.in` files.
- The build system collects and runs them via `make verify`.

## Running Tests

| Command | Purpose |
|---|---|
| `make verify` | Full test suite from repo root |
| `make verbose_verify` | Full suite with test output |
| `make gdb_verify` | Full suite under GDB |
| `make valgrind_verify` | Full suite under valgrind |
| `make testsuites` | Build test files without running |
| `pike testsuite` | Run a single preprocessed suite |

## Common Patterns

### Array and mapping operations
```pike
test_eq(sizeof(({1,2,3})), 3)
test_any([[return ({1,2,3})[-1];]], 3)
test_eq(indices((["a":1,"b":2])) | sort, ({"a","b"}) | sort)
```

### String operations
```pike
test_eq("hello"*2, "hellohello")
test_eq(sizeof("Pike"), 4)
test_any([[return String.Buffer("ab")->get();]], "ab")
```

### Error handling
```pike
test_eval_error([[ error("fail"); ]])
test_eval_error([[ ({})[42] ]])
test_compile_error([[ int x = "nope"; ]])
```

### Zero-type / mapping key detection
```pike
test_true(zero_type((["a":0])["b"]))   // key missing: zero_type == 1
test_false(zero_type((["a":0])["a"]))  // key present: zero_type == 0
```

### Conditional tests
```pike
cond([[constant Sql.Sql]], [[
  test_do([[object db = Sql.Sql("sqlite://:memory:");]])
]])
```
