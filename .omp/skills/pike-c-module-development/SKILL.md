---
name: pike-c-module-development
description: Writing native C modules for Pike using .cmod files, svalue stack, and GC-safe patterns
globs:
  - "**/*.cmod"
  - "src/modules/**"
  - "src/post_modules/**"
metadata:
  version: "1.0.0"
  organization: pike-lang
---

# Pike C Module Development

## Overview
Native C modules extend Pike with high-performance primitives. Modules use `.cmod` files
processed by Pike's precompiler, which generates C code with proper boilerplate for
PIKECLASS, PIKEFUN, and reference-counted svalue stack management. Every module must
balance reference counts correctly, respect the svalue stack discipline, and integrate
with Pike's garbage collector.

## Key Concepts
- **.cmod files**: Precompiler input combining C code with Pike class/function declarations
- **Svalue stack**: Arguments arrive at `Pike_sp[-args]` through `Pike_sp[-1]`; results are pushed
- **Reference counting**: All Pike objects (strings, arrays, mappings, objects) are refcounted
- **GC integration**: C modules must cooperate with Pike's mark-and-sweep collector
- **THIS macro**: Accesses per-object C storage via `Pike_fp->current_storage`

## Rules

### 1. Module Structure (PIKECLASS, PIKEFUN, CVAR, INIT, EXIT)

Use the .cmod macros to declare classes, functions, per-object storage, and lifecycle hooks.

**WRONG**: Manually defining Pike classes and vtables in raw C.
```c
// Never do this — raw C without .cmod macros
struct program *my_class_prog;
void my_func(INT32 args) {
    struct my_storage *s = (struct my_storage *)
        get_storage(Pike_fp->current_object, my_class_prog);
    // manual dispatch, no type checking...
}
```

**CORRECT**: Use .cmod macros for structure.
```c
PIKECLASS MyClass
{
  CVAR struct {
    int fd;
    char *buffer;
  } storage;

  PIKEFUN int get_fd()
  {
    RETURN THIS->fd;
  }

  PIKEFUN void set_fd(int f)
  {
    THIS->fd = f;
  }

  INIT
  {
    THIS->fd = -1;
    THIS->buffer = NULL;
  }

  EXIT
  {
    if (THIS->buffer) {
      free(THIS->buffer);
      THIS->buffer = NULL;
    }
    if (THIS->fd != -1) {
      close(THIS->fd);
      THIS->fd = -1;
    }
  }
}
```

### 2. Svalue Stack Discipline (pop args, push results)

PIKEFUN receives arguments on the stack. For void functions, pop args. For valued
functions, the `RETURN` macro handles pop+push. When doing manual stack work, always
pop consumed elements.

**WRONG**: Forgetting to pop arguments or leaving stale stack entries.
```c
PIKEFUN void process(string data)
{
  // Uses data but never pops — stack corrupted
  do_something(data->str);
  // No pop_n_elems(args) or RETURN
}
```

**CORRECT**: Use RETURN (handles pop+push) or explicit stack management.
```c
PIKEFUN void process(string data)
{
  do_something(data->str);
  pop_n_elems(args);
}

PIKEFUN int get_length(string data)
{
  // RETURN handles pop_n_elems(args) + push result
  RETURN data->len;
}
```

### 3. Reference Counting (add_ref/free_svalue balance)

Pike objects are reference-counted. When you store a reference beyond the current
call, you must `add_ref`. When you release it, you must free it. The svalue stack
owns what's on it — don't free stack values you didn't add_ref.

**WRONG**: Storing a Pike object pointer without incrementing its refcount.
```c
CVAR struct pike_string *cached_name;

PIKEFUN void set_name(string name)
{
  // Storing without add_ref — if name is freed by Pike,
  // THIS->cached_name becomes a dangling pointer
  THIS->cached_name = name;
}
```

**CORRECT**: Add reference on store, free on replace and in EXIT.
```c
CVAR struct pike_string *cached_name;

PIKEFUN void set_name(string name)
{
  if (THIS->cached_name)
    free_string(THIS->cached_name);
  copy_shared_string(THIS->cached_name, name);
}

EXIT
{
  if (THIS->cached_name) {
    free_string(THIS->cached_name);
    THIS->cached_name = NULL;
  }
}
```

### 4. String Handling (make_shared_string, free_string)

Pike strings are shared and reference-counted. Always use `make_shared_string` or
`make_shared_binary_string` to create them. Use `free_string` (which decrements
the refcount and frees only when it reaches zero). Use `string_builder` for
construction of dynamic strings.

**WRONG**: Creating strings from raw C without going through Pike's allocator.
```c
PIKEFUN string greet(string name)
{
  char buf[256];
  snprintf(buf, sizeof(buf), "Hello %s", name->str);
  // Leaked — no one owns this, and it bypasses Pike's string table
  struct pike_string *s = (struct pike_string *)malloc(sizeof(struct pike_string));
  RETURN s; // Completely wrong — not a valid Pike string
}
```

**CORRECT**: Use Pike's string APIs.
```c
PIKEFUN string greet(string name)
{
  struct string_builder sb;
  init_string_builder(&sb, 0);
  string_builder_strcat(&sb, "Hello ");
  string_builder_shared_strcat(&sb, name);
  RETURN finish_string_builder(&sb);
}
```

### 5. Type Checking (PIKE_T_INT, PIKE_T_STRING, etc.)

PIKEFUN with declared types handles argument verification automatically. When working
with raw svalues (e.g., in generic code), check TYPEOF() against PIKE_T_* constants.

**WRONG**: Assuming svalue types without checking.
```c
PIKEFUN void handle(mixed val)
{
  // Assumes val is a string without checking
  struct pike_string *s = val->u.string; // crash if val is int
  process_string(s);
}
```

**CORRECT**: Check types before accessing union fields.
```c
PIKEFUN void handle(mixed val)
{
  if (TYPEOF(*val) == PIKE_T_STRING) {
    process_string(val->u.string);
  } else if (TYPEOF(*val) == PIKE_T_INT) {
    process_int(val->u.integer);
  } else {
    Pike_error("Expected string or int, got %d.\n", TYPEOF(*val));
  }
}
```

### 6. Module Lifecycle (PIKE_MODULE_INIT, PIKE_MODULE_EXIT)

Module-level initialization and cleanup go in `PIKE_MODULE_INIT` and
`PIKE_MODULE_EXIT`. Use INIT for per-class setup and PIKE_MODULE_INIT for
module-wide constants and global state.

**WRONG**: Doing module-level setup inside a PIKECLASS INIT block.
```c
PIKECLASS Foo
{
  INIT
  {
    // This runs per-instance, not per-module load
    add_integer_constant("VERSION", 1, 0); // Wrong place
  }
}
```

**CORRECT**: Module constants in PIKE_MODULE_INIT, per-instance in INIT.
```c
PIKECLASS Foo
{
  INIT
  {
    // Per-instance setup
    THIS->initialized = 0;
  }

  EXIT
  {
    // Per-instance cleanup
  }
}

PIKE_MODULE_INIT
{
  // Module-level constants
  add_integer_constant("VERSION", 1, 0);
  // Start any global services
}

PIKE_MODULE_EXIT
{
  // Cleanup global state
}
```

## References
- [Module API](references/module-api.md) — Complete .cmod macro reference
- [Svalue Handling](references/svalue-handling.md) — Stack, types, and refcount operations
