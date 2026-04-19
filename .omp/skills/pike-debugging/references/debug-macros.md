# Debugging Tools and Macros Reference

## Pike-Level Debugging

### Error Handling
- `catch { ... }` — Catch any error. Returns error object on failure, 0 on success.
- `describe_backtrace(error)` — Format a stack trace from a caught error.
- `describe_error(error)` — Extract just the error message string.
- `master()->handle_error(err)` — Global unhandled error hook. Override in master for centralized logging.

### Logging and Output
- `werror(format, args...)` — Write formatted output to stderr. Use for debug/diagnostic messages.
- `write(format, args...)` — Write formatted output to stdout. Use for program output.
- `sprintf("%O", value)` — Inspect any value. `%O` produces a readable dump including type and structure.

### Object Introspection
- `destruct(object)` — Force immediate destruction of an object. Calls `destroy()` synchronously.
- `_refs(obj)` — Return reference count of an object. Useful for leak detection.
- `objectp(x)`, `programp(x)`, `stringp(x)`, `intp(x)`, `mappingp(x)`, `arrayp(x)`, `multisetp(x)` — Runtime type predicates.
- `typeof(x)` — Return the type of a value at runtime.

### Debug Framework
- `Debug.Profiling` — Profiling support for performance analysis.

### Garbage Collection
- `Pike.gc_parameters()` — Return (or set) GC parameters and statistics. Returns a mapping with fields like `gc_interval`, `garbage_ratio`, etc.
- `gc()` — Force a garbage collection cycle.

## C-Level Debugging

### Compile-Time Flags
- `--with-dmalloc` — Configure flag to enable dmalloc memory debugging.
- `PIKE_DEBUG` — Preprocessor define that enables extra runtime checks in the interpreter.
- `DO_IF_DEBUG(code)` — Macro that expands to `code` when `PIKE_DEBUG` is defined, nothing otherwise.

### C API Macros
- `PMOD_EXPORT` — Marks a function or variable as part of the public module API.
- `PIKE_FATAL(msg)` / `Pike_fatal(msg)` — Fatal error. Prints message and aborts. Use for unrecoverable C-level assertions.
- `Pike_error(msg)` — Throw a Pike-level error from C code. The Pike `catch` mechanism will intercept it.
- `push_undefined()` — Push an undefined value onto the Pike value stack.

### Value Stack (C Level)
- `Pike_sp` — Pointer to top of the evaluator stack. Arguments to a C-level function are at `Pike_sp[-args]` through `Pike_sp[-1]`.
- `Pike_fp` — Current call frame pointer. Local variables accessible via `Pike_fp->locals`.

## GDB with Pike

### Useful Commands
- `info threads` — List all threads in the Pike process.
- `print Pike_sp` — Inspect the evaluator stack pointer.
- `print Pike_fp` — Inspect the current frame pointer.
- `print Pike_fp->locals[i]` — Access local variables in the current frame.

### Breakpoint Targets
- `interpret_functions.h` — Set breakpoints on specific opcodes (e.g., `f_add`, `f_call_function`).
- `gc.c` — Debug marker macros for tracing GC behavior.
- `module.c` — Break on module loading failures.

## Common Debug Scenarios

### Memory Leak
Track reference counts and force collection:
```pike
werror("Refs before: %d\n", _refs(obj));
obj = 0;
gc();
// Re-check surviving objects with _refs()
```

### Stack Overflow
- Default evaluator stack size: `EVALUATOR_STACK_SIZE = 100000`.
- Check stack depth with `Pike_sp` in GDB.
- Recursive calls without a base case are the usual cause.

### Type Errors
- Use `typeof(x)` at runtime to inspect actual types.
- Use type annotations at compile time for early detection.
- `sprintf("%O", x)` reveals the full structure.

### GC Issues
- `Pike.gc_parameters()` for live statistics.
- Avoid circular references in C code — Pike's GC only tracks Pike-level references.
- `gc()` to force collection and observe behavior.

### Module Not Loading
- Verify `PIKE_MODULE_PATH` includes the module directory.
- Use `master()->show_inherited()` to inspect inheritance chains.
- Check for missing `.so` files or unresolved symbols with `ldd`.
