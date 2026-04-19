# Pike Svalue and Stack Reference

## Svalue Structure

The svalue (short for "stack value") is Pike's universal value container. Every value
on the evaluation stack, in arrays, and in mappings is an svalue.

### struct svalue

Defined in `svalue.h`. Access type/subtype via macros, not direct field access.

```c
struct svalue {
  union {
    struct {
      unsigned short type;    /* PIKE_T_* constant */
      unsigned short subtype; /* e.g. NUMBER_NUMBER, NUMBER_UNDEFINED */
    } t;
    /* internal combined field: type_subtype */
  } tu;
  union anything u;           /* the actual value */
};
```

### Access Macros

| Macro | Purpose |
|-------|---------|
| `TYPEOF(sv)` | Read the type field |
| `SUBTYPEOF(sv)` | Read the subtype field |
| `SET_SVAL(sv, type, subtype, u_field, val)` | Initialize all fields at once |
| `SET_SVAL_TYPE(sv, type)` | Set type only |
| `SET_SVAL_SUBTYPE(sv, subtype)` | Set subtype only |

```c
SET_SVAL(sv, PIKE_T_INT, NUMBER_NUMBER, integer, 42);
/* equivalent to: */
TYPEOF(sv) = PIKE_T_INT;
SUBTYPEOF(sv) = NUMBER_NUMBER;
sv.u.integer = 42;
```

## Type Constants

Defined in `svalue.h`.

| Constant | Value | C Type | `u` field |
|----------|-------|--------|-----------|
| `PIKE_T_INT` | 0 | `INT_TYPE` | `.u.integer` |
| `PIKE_T_FLOAT` | 1 | `FLOAT_TYPE` | `.u.float_number` |
| `PIKE_T_ARRAY` | 8 | `struct array *` | `.u.array` |
| `PIKE_T_MAPPING` | 9 | `struct mapping *` | `.u.mapping` |
| `PIKE_T_MULTISET` | 10 | `struct multiset *` | `.u.multiset` |
| `PIKE_T_OBJECT` | 11 | `struct object *` | `.u.object` |
| `PIKE_T_FUNCTION` | 12 | callable | `.u.efun` / object+id |
| `PIKE_T_PROGRAM` | 13 | `struct program *` | `.u.program` |
| `PIKE_T_STRING` | 14 | `struct pike_string *` | `.u.string` |
| `PIKE_T_TYPE` | 15 | `struct pike_type *` | `.u.type` |
| `PIKE_T_MIXED` | 251 | (any) | (any) |
| `PIKE_T_ZERO` | 6 | (none) | (none) |
| `PIKE_T_FREE` | 19 | (freed marker) | Freed svalue marker — used internally |

## Integer Subtypes

Defined in `svalue.h`. Only meaningful when `TYPEOF(sv) == PIKE_T_INT`.

| Constant | Value | Meaning |
|----------|-------|---------|
| `NUMBER_NUMBER` | 0 | Regular integer |
| `NUMBER_UNDEFINED` | 1 | `UNDEFINED` value |
| `NUMBER_DESTRUCTED` | 2 | Destructed object reference |

```c
/* Check for UNDEFINED vs 0 */
if (TYPEOF(sv) == PIKE_T_INT && SUBTYPEOF(sv) == NUMBER_UNDEFINED) {
  /* This is UNDEFINED, not the integer 0 */
}
```

## Stack Operations

The svalue stack pointer is `Pike_sp`. It always points one past the top element.

### Stack Access

| Expression | Meaning |
|------------|---------|
| `Pike_sp[-1]` | Top of stack |
| `Pike_sp[-2]` | Second from top |
| `Pike_sp[-args]` | First argument to current function |
| `Pike_sp[0]` | Invalid — one past top |

### Push Operations

Each push increments `Pike_sp` and sets the svalue.

| Function | Signature | Effect |
|----------|-----------|--------|
| `push_int(n)` | `INT_TYPE n` | Push integer |
| `push_float(n)` | `FLOAT_TYPE n` | Push float |
| `push_text("str")` | `const char *` | Push narrow C string (copied) |
| `push_string(s)` | `struct pike_string *` | Push shared string (takes ownership) |
| `push_array(a)` | `struct array *` | Push array (takes reference) |
| `push_mapping(m)` | `struct mapping *` | Push mapping (takes reference) |
| `push_object(o)` | `struct object *` | Push object (takes reference) |
| `push_function(obj, fun)` | `struct object *, int` | Push function |
| `push_undefined()` | — | Push UNDEFINED |
| `push_int(0)` | — | Push integer zero |

```c
push_int(42);           /* Pike_sp now has {PIKE_T_INT, NUMBER_NUMBER, 42} */
push_text("hello");     /* shared string pushed */
push_string(make_shared_binary_string(data, len));
```

### Push with reference increment

These increment the refcount before pushing (for values you also hold a reference to):

| Function | Effect |
|----------|--------|
| `ref_push_string(s)` | `add_ref(s); push_string(s)` |
| `ref_push_array(a)` | `add_ref(a); push_array(a)` |
| `ref_push_mapping(m)` | `add_ref(m); push_mapping(m)` |
| `ref_push_object(o)` | `add_ref(o); push_object(o)` |

### Pop Operations

| Function | Effect |
|----------|--------|
| `pop_stack()` | Remove top element (frees its svalue) |
| `pop_n_elems(n)` | Remove `n` elements from top |
| `pop_2_elems()` | Remove top two elements |

```c
pop_n_elems(args);   /* Remove all function arguments */
push_int(0);         /* Push return value */
```

### Stack Manipulation

| Function | Effect |
|----------|--------|
| `stack_swap()` | Swap top two elements |
| `stack_pop_n_elems_keep_top(n)` | Pop `n` elements below top, keeping top |

## Argument Handling

Within `PIKEFUN`, arguments are available as named C variables matching the
declared Pike types. The precompiler generates type-checking code.

### Named argument access (PIKEFUN)

```c
PIKEFUN int add(int a, int b)
{
  /* 'a' and 'b' are INT_TYPE */
  RETURN a + b;
}

PIKEFUN void process(string data)
{
  /* 'data' is struct pike_string * */
  if (data->size_shift)
    Pike_error("Wide strings not supported.\n");
  pop_n_elems(args);
}
```

### Manual argument access (raw C functions)

```c
void f_my_func(INT32 args)
{
  if (args < 2)
    SIMPLE_TOO_FEW_ARGS_ERROR("my_func", 2);

  if (TYPEOF(Pike_sp[-args]) != PIKE_T_STRING)
    SIMPLE_BAD_ARG_ERROR("my_func", 1, "string");

  if (TYPEOF(Pike_sp[-args+1]) != PIKE_T_INT)
    SIMPLE_BAD_ARG_ERROR("my_func", 2, "int");

  struct pike_string *s = Pike_sp[-args].u.string;
  INT_TYPE n = Pike_sp[-args+1].u.integer;

  pop_n_elems(args);
  push_int(result);
}
```

## Reference Counting

All Pike heap objects are reference-counted. Rules:
- The **stack** owns what's on it. Pushing transfers or shares ownership.
- **Storing** a reference in CVAR or a C struct requires `add_ref` or `copy_*`.
- **Releasing** a stored reference uses `free_*` or `free_svalue`.

### add_ref

```c
add_ref(some_object);   /* Increment reference count */
```

Note: `add_ref` returns the new count, but you typically ignore it. It is a macro.

### Free functions

| Function | Target |
|----------|--------|
| `free_svalue(sv)` | Generic — dispatches on type |
| `free_string(s)` | `struct pike_string *` |
| `free_array(a)` | `struct array *` |
| `free_mapping(m)` | `struct mapping *` |
| `free_multiset(l)` | `struct multiset *` |
| `free_object(o)` | `struct object *` |
| `free_program(p)` | `struct program *` |

All decrement the refcount and free only when it reaches zero.

### Copy helpers

| Function | Effect |
|----------|--------|
| `copy_shared_string(dest, src)` | `add_ref(src); dest = src` |
| `copy_svalues_recursively_no_free(dst, src, n, flags)` | Deep copy |

### Pattern: Replace stored reference

```c
/* Release old, store new with refcount */
if (THIS->cached_str)
  free_string(THIS->cached_str);
copy_shared_string(THIS->cached_str, new_str);
```

### Pattern: Store from stack argument

```c
PIKEFUN void set_callback(function cb)
{
  if (THIS->cb.u.efun)
    free_svalue(&THIS->cb);
  assign_svalue(&THIS->cb, cb);
}

EXIT
{
  if (THIS->cb.u.efun)
    free_svalue(&THIS->cb);
}
```

## String Construction

Pike strings are shared, immutable, and reference-counted. Always use Pike's
allocation functions.

### Creating shared strings

| Function | Effect |
|----------|--------|
| `make_shared_string(cstr)` | From NUL-terminated C string |
| `make_shared_binary_string(data, len)` | From buffer with explicit length |
| `make_shared_binary_string0(data, len)` | From 8-bit chars |
| `make_shared_binary_string1(data, len)` | From 16-bit chars |
| `make_shared_binary_string2(data, len)` | From 32-bit chars |

These return a `struct pike_string *` with refcount 1. Ownership transfers to
the caller.

### Manual string building

| Function | Effect |
|----------|--------|
| `begin_shared_string(len)` | Allocate buffer of `len` bytes |
| `end_shared_string(s)` | Finalize and intern the string |
| `low_end_shared_string(s)` | Finalize without canonicalization |

```c
struct pike_string *s = begin_shared_string(10);
memcpy(s->str, "0123456789", 10);
s = end_shared_string(s);
/* s is now a shared Pike string */
```

### String builder (dynamic construction)

| Function | Effect |
|----------|--------|
| `init_string_builder(sb, shift)` | Initialize (shift=0 for 8-bit) |
| `init_string_builder_alloc(sb, len, shift)` | Init with preallocated size |
| `string_builder_strcat(sb, cstr)` | Append C string |
| `string_builder_shared_strcat(sb, ps)` | Append a Pike string |
| `string_builder_binary_strcat0(sb, data, len)` | Append raw bytes |
| `string_builder_putchar(sb, ch)` | Append single character |
| `string_builder_fill(sb, howmany, from, len, rep)` | Fill with pattern |
| `finish_string_builder(sb)` | Returns `struct pike_string *` |
| `free_string_builder(sb)` | Free builder without producing string |

```c
PIKEFUN string build_greeting(string name)
{
  struct string_builder sb;
  init_string_builder(&sb, 0);
  string_builder_strcat(&sb, "Hello, ");
  string_builder_shared_strcat(&sb, name);
  string_builder_strcat(&sb, "!");
  RETURN finish_string_builder(&sb);
}
```

## Memory Allocation

Pike provides wrappers that integrate with its memory tracking.

| Function | Purpose |
|----------|---------|
| `xalloc(size)` | Pike's malloc — fatal error on OOM |
| `ALLOC_STRUCT(type)` | Allocate and cast: `(struct type *)xalloc(sizeof(struct type))` |

```c
struct my_data *d = ALLOC_STRUCT(my_data);
memset(d, 0, sizeof(struct my_data));
```

## Garbage Collector Integration

Pike uses a mark-and-sweep GC. C modules holding references to Pike objects must
participate.

### gc_mark_external

Marks a Pike object as reachable from C code. Called during the GC mark phase.

```c
gc_mark_external(THIS->some_obj, "my_module: some_obj reference");
```

### gc_cycle_check

Called during cycle detection. Pike provides per-type wrappers:

```c
gc_cycle_check_object(obj, 0);
gc_cycle_check_array(arr, 0);
gc_cycle_check_mapping(m, 0);
gc_cycle_check_svalues(svals, count);
```

### PIKEVAR and GC

`PIKEVAR` declarations automatically generate GC visit callbacks. `CVAR` fields
holding Pike references require manual GC integration via the `EXTRA` block:

```c
EXTRA
{
  /* Tell GC about CVAR Pike references via visit/check callbacks */
}

In most cases, prefer `PIKEVAR` for Pike-visible variables (auto GC) and only
use `CVAR` with manual GC for internal caches.

## Aggregate Construction

| Function | Effect |
|----------|--------|
| `f_aggregate(n)` | Pop `n` values, push as array |
| `f_aggregate_mapping(n*2)` | Pop `n*2` values (key-value pairs), push mapping |
| `f_add(n)` | Add top `n` values (string concat, int add, etc.) |

```c
push_int(1);
push_int(2);
push_int(3);
f_aggregate(3);  /* Pike_sp[-1] is now ({1, 2, 3}) */
```

## Common Patterns

### PIKEFUN returning a string built from arguments

```c
PIKEFUN string join(string a, string b)
{
  struct string_builder sb;
  init_string_builder(&sb, MAXIMUM(a->size_shift, b->size_shift));
  string_builder_shared_strcat(&sb, a);
  string_builder_shared_strcat(&sb, b);
  RETURN finish_string_builder(&sb);
}
```

### PIKEFUN with manual stack work

```c
PIKEFUN array(int) range(int from, int to)
{
  int i;
  pop_n_elems(args);
  for (i = from; i <= to; i++)
    push_int(i);
  f_aggregate(to - from + 1);
}
```

### Calling a stored callback

```c
PIKEFUN void invoke(int code, string msg)
{
  if (TYPEOF(THIS->callback) == PIKE_T_FUNCTION) {
    push_int(code);
    push_string(msg);
    safe_apply_svalue(&THIS->callback, 2, 1);
    pop_stack();
  }
}
```

### Error handling

```c
PIKEFUN void write(string data)
{
  ssize_t written = write(THIS->fd, data->str, data->len);
  if (written < 0)
    Pike_error("write failed: %s\n", strerror(errno));
  RETURN written;
}
```
