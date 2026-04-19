---
name: pike-debugging
description: Debugging Pike programs and interpreter using built-in tools, error handling, and GC introspection
globs:
  - "**/*.pike"
  - "**/*.pmod"
metadata:
  version: "1.0.0"
  organization: pike-lang
---

# Pike Debugging

## Overview
Debugging Pike programs using built-in error handling, logging, introspection, and GC tuning tools. Covers both Pike-level and C-level debugging scenarios.

## Key Concepts

- Pike uses `catch` (not try/catch) for error handling — it returns the error or 0
- `werror()` writes to stderr; `write()` writes to stdout
- `%O` format specifier dumps a readable representation of any value
- `describe_backtrace()` formats stack traces from caught errors
- Use conditional compilation (`constant DEBUG = (int)getenv("DEBUG")`) for debug output
- `Pike.gc_parameters()` provides GC tuning and statistics
- `_refs()` inspects reference counts for leak detection
- `gc()` forces garbage collection

## Rules

### 1. Use `catch` for error handling

Pike uses `catch { ... }` — not try/catch — to intercept errors. It returns the error object on failure, or 0 on success.

```pike
// WRONG
try {
  do_something();
} catch (error e) {
  // Pike does not have try/catch
}
```

```pike
// CORRECT
mixed err = catch {
  do_something();
};
if (err) {
  werror("Failed: %s\n", describe_error(err));
}
```

### 2. Use `werror()` for debug logging

`werror()` writes to stderr. Use it for debug and diagnostic output; reserve `write()` for program output.

```pike
// WRONG
write("Debug: value is " + value + "\n");
```

```pike
// CORRECT
werror("Debug: value is %O\n", value);
```

### 3. Use `%O` format for inspecting values

`sprintf("%O", x)` produces a readable dump of any value — type, structure, and content. Use it instead of string concatenation for diagnostics.

```pike
// WRONG
werror("Got array: " + (string)my_array + "\n");
```

```pike
// CORRECT
werror("Got array: %O\n", my_array);
```

### 4. Use conditional compilation for debug output

Evaluate debug flags once at load time with `constant`. This avoids repeated string comparisons and lets the compiler optimize away dead branches.

```pike
// WRONG
if (getenv("DEBUG")) {
  // Ad-hoc, string comparison every call
  werror("Verbose output...\n");
}
```

```pike
// CORRECT
constant DEBUG = (int)getenv("DEBUG");
// Evaluated once at load time; dead code eliminated
if (DEBUG) werror("Verbose output...\n");
```
### 5. Use `Pike.gc_parameters()` for GC tuning

`Pike.gc_parameters()` returns and optionally sets garbage collection parameters. Use it to inspect GC state and tune collection behavior.

```pike
// WRONG
// Guessing about GC behavior without data
gc();
```

```pike
// CORRECT
mapping params = Pike.gc_parameters();
werror("GC stats: %O\n", params);
gc(); // Force collection, then re-check
werror("After GC: %O\n", Pike.gc_parameters());
```

### 6. Use `destruct()` for cleanup debugging

Force destruction of objects to test cleanup paths. `destruct(obj)` calls the object's `destroy()` method immediately.

```pike
// WRONG
obj = 0; // Relies on GC to collect, timing is nondeterministic
gc();
```

```pike
// CORRECT
destruct(obj); // Immediately triggers destroy(), deterministic
```

### 7. Use `describe_backtrace()` for stack traces

Catch errors and format the full stack trace with `describe_backtrace()`. It works on the error object returned by `catch`.

```pike
// WRONG
mixed err = catch { failing_function(); };
if (err) werror("Error: %O\n", err);
```

```pike
// CORRECT
mixed err = catch { failing_function(); };
if (err) {
  werror("Error: %s\n%s\n",
         describe_error(err),
         describe_backtrace(err));
}
```

### 8. Use `master()->handle_error()` for global error handling

Override `handle_error` in the master object to intercept all unhandled errors globally — useful for logging frameworks and crash reporters.

```pike
// WRONG
// Patching every call site individually
```

```pike
// CORRECT
// In a custom master or override:
void handle_error(mixed err) {
  werror("Unhandled: %s\n%s\n",
         describe_error(err),
         describe_backtrace(err));
  ::handle_error(err);
}
```
