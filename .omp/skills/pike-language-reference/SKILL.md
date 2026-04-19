---
name: pike-language-reference
description: Pike programming language syntax, semantics, type system, and standard library patterns for correct code generation
globs:
  - "**/*.pike"
  - "**/*.pmod"
  - "**/*.cmod"
metadata:
  version: "1.0.0"
  organization: pike-lang
---

# Pike Language Reference

## Overview

Pike is a dynamic, bytecode-compiled, object-oriented language with C-like syntax. Target version: 8.0.1116. It features strong typing with type inference, first-class functions/closures, and a rich standard library. Pike compiles to bytecode and runs on its own virtual machine.

## Key Concepts

- **Value types** (int, float, string) are copied on assignment.
- **Reference types** (array, mapping, multiset, object, program, function) share underlying data on assignment.
- **Strings are immutable** — all string operations return new strings.
- **Arrays, mappings, multisets, objects, functions, programs, and types use identity comparison** with `==`. Use `equal()` for structural/deep comparison.
- **Integer division rounds toward negative infinity** (floor division), not toward zero.
- **switch cases DO fall through** — use `break` to prevent (like C/Java).
- **foreach iterates by value** with optional index; iterate strings as character codes (int).

## Rules

### Rule: Literal Syntax

Pike uses curly-brace forms for collection literals, not square brackets or curly braces alone.

WRONG:
```pike
array a = [1, 2, 3];
mapping m = {"key": "value"};
multiset s = {"a", "b"};
```

CORRECT:
```pike
array a = ({1, 2, 3});
mapping m = (["key": "value"]);
multiset s = (< "a", "b" >);
```

### Rule: String Immutability

Strings cannot be modified in place. All operations produce new strings. Use `String.Buffer` for efficient concatenation.

WRONG:
```pike
string s = "hello";
s[0] = 'H'; // runtime error: strings are immutable
string result = "";
for (int i = 0; i < 1000; i++)
  result += "x"; // O(n^2): allocates new string each iteration
```

CORRECT:
```pike
String.Buffer buf = String.Buffer();
for (int i = 0; i < 1000; i++)
  buf->add("x");
string result = buf->get();
```

### Rule: Array Identity vs Equality

The `==` operator on arrays checks object identity (same array object), not structural equality. Use `equal()` for deep comparison.

WRONG:
```pike
array a = ({1, 2, 3});
array b = ({1, 2, 3});
if (a == b) // false — different array objects
  write("same\n");
```

CORRECT:
```pike
array a = ({1, 2, 3});
array b = ({1, 2, 3});
if (equal(a, b)) // true — same contents
  write("same\n");
```

### Rule: Reference Semantics for Pointer Types

Arrays, mappings, multisets, objects, and programs are reference types. Assigning to a variable or passing to a function shares the same underlying data. Modifying through one alias affects all others.

WRONG:
```pike
array original = ({1, 2, 3});
array copy = original;
copy[0] = 99;
// original is now ({99, 2, 3}) — NOT ({1, 2, 3})
```

CORRECT:
```pike
array original = ({1, 2, 3});
array copy = copy_value(original); // deep copy
copy[0] = 99;
// original remains ({1, 2, 3})
```

### Rule: Integer Division Rounding

Integer division rounds toward negative infinity (floor division), not toward zero. This differs from C/Java.

WRONG:
```pike
// Assuming C-style truncation toward zero:
int a = -8 / 3;  // expecting -2 (wrong)
```

CORRECT:
```pike
int a = -8 / 3;  // -3 — rounds toward -inf
int b = 8 / 3;   // 2 — same as C for positive numbers
// If truncation toward zero is needed, cast:
int c = (int)((float)-8 / 3); // -2
```

### Rule: Constructor and Destructor Naming

Constructors are named `create()`, not `__init__` or `__construct`. Destructors are `destroy()`.

WRONG:
```pike
class Foo {
  void __init() { }     // not a constructor
  void __construct() { } // not a constructor
}
```

CORRECT:
```pike
class Foo {
  void create(string name) {
    // constructor — called on Foo("bar")
  }
  void destroy() {
    // destructor — called when garbage collected
  }
}
```

### Rule: Inheritance Syntax

Use `inherit` keyword. Call parent methods with `::` prefix. Use `this_program::` for disambiguation.

WRONG:
```pike
class Child extends Parent { }
class Child : Parent { }
void parent_method() { super(); }
```

CORRECT:
```pike
class Child {
  inherit Parent;
  void create() {
    ::create(); // call parent constructor
  }
  void method() {
    ::method(); // call parent method
  }
}
```

### Rule: Switch Fall-Through

Pike switch cases DO fall through (like C/Java). Use `break` to prevent fall-through. Missing `break` is a common bug.

WRONG (missing break — will fall through to next case):
```pike
switch (x) {
  case 1:
    do_one();
  case 2:
    do_two();
  // BUG: falls through into do_range()!
  case 3..5:
    do_range();
}
```

CORRECT:
```pike
switch (x) {
  case 1:
    do_one();
    break;
  case 2:
    do_two();
    break;
  case 3..5:
    do_range();
    break;
  default:
    do_default();
    break;
}
// Use stacked case labels for shared handling:
switch (x) {
  case 1:
  case 2:
  case 3:
    handle_one_two_three();
    break;
}
```

### Rule: Foreach Syntax

`foreach` iterates by value. Use the three-argument form for index access. Strings iterate as character codes (int).

WRONG:
```pike
foreach (arr as val) { }        // PHP syntax
for (val in arr) { }            // Python syntax
foreach (string s in "hello") { // strings don't iterate as strings
```

CORRECT:
```pike
foreach (arr; int idx; mixed val) { }
foreach (mapping; mixed key; mixed val) { }
foreach (multiset; mixed key; int(0..1) present) { }
foreach ("hello"; int idx; int char) {
  // char is the character CODE (int), e.g. 104 for 'h'
  string ch = sprintf("%c", char);
}
```

### Rule: sprintf and sscanf

`sprintf` uses Pike format specifiers. `%O` dumps a readable representation of any value. `sscanf` returns the number of matched fields, not the parsed values (those are output parameters).

WRONG:
```pike
string s = "%d items" % count;              // not Python
array parts = sscanf("42 items", "%d %s");   // wrong: sscanf returns count
write(value);                                 // use write("%O\n", value) to inspect
```

CORRECT:
```pike
string s = sprintf("%d items", count);
int n;
string rest;
int matched = sscanf("42 items", "%d %s", n, rest);
// matched == 2, n == 42, rest == "items"
write("%O\n", some_value); // %O gives readable dump of any type
```

### Rule: zero_type for Missing Keys

`m[key]` returns `0` (zero) for both a missing key and a key mapped to `0`. Use `zero_type()` to distinguish them.

WRONG:
```pike
mapping m = (["a": 0]);
if (!m["a"]) { }          // true — but key EXISTS
if (m["b"] == 0) { }      // true — but key is MISSING
if (m["b"] == UNDEFINED) { } // also true for missing, but zero_type is canonical
```

CORRECT:
```pike
mapping m = (["a": 0]);
if (!zero_type(m["a"])) {
  // key exists (even though value is 0)
}
if (zero_type(m["b"]) == 1) {
  // key is genuinely missing
}
// zero_type returns: 0 (value exists), 1 (UNDEFINED/missing key), 2 (destructed object)
```

### Rule: Operator Overloading via lfuns

Pike operator overloading uses named "lfuns" (low-level functions). These are special method names that Pike calls when operators are used on objects.

WRONG:
```pike
class Vec {
  int x, y;
  Vec operator+(Vec other) { } // C++ syntax
  Vec __add__(Vec other) { }   // Python syntax
}
```

CORRECT:
```pike
class Vec {
  int x, y;
  Vec `+(Vec other) {
    return Vec(this->x + other->x, this->y + other->y);
  }
  string _sprintf(int fmt) {
    return sprintf("(%d, %d)", x, y);
  }
  int _equal(mixed other) {
    return objectp(other) && equal(x, other->x) && equal(y, other->y);
  }
}
// Common lfuns: `+, `-, `*, `/, `%, `&, `|, `^, `<<, `>>, `+=, `[..],
//               `==, `<, `>, `!, `[], `[]=, _sizeof, _values,
//               _indices, _sprintf, _equal, _hash, cast, _random, _search,
//               _types, _serialize, _deserialize, _get_iterator, _m_delete, __hash
```
