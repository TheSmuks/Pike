# Pike Syntax Reference

## Identifiers and Keywords

Identifiers: letter or `_` followed by letters, digits, `_`. Case-sensitive.

Reserved keywords: `break`, `case`, `catch`, `class`, `constant`, `continue`, `default`, `do`, `else`, `final`, `for`, `foreach`, `gauge`, `if`, `import`, `inherit`, `inline`, `int`, `lambda`, `local`, `mapping`, `mixed`, `multiset`, `nomask`, `object`, `optional`, `private`, `program`, `protected`, `public`, `return`, `sscanf`, `static`, `string`, `switch`, `typedef`, `typeof`, `variant`, `void`, `while`, `enum`, `global`, `bits`, `deprecated`, `extern`, `auto`.

## Literals

### Integer
```pike
42              // decimal
0x2A            // hexadecimal
0o52            // octal
0b101010        // binary
1_000_000       // underscores for readability
-1              // negative
Int.NATIVE_MAX  // platform native int max
Int.MAX         // maximum representable integer (bignum)
```

### Float
```pike
3.14
1.0e10
1.0e-5
Math.inf          // positive infinity (Math module)
Math.nan          // not-a-number (Math module)
```

### String
```pike
"hello"              // double-quoted
'c'                  // character literal → int (character code)
#"raw string"
``prefix"string"      // prefix quoting
```

Escape sequences: `\a` (bell), `\b` (backspace), `\t` (tab), `\n` (newline), `\v` (vertical tab), `\f` (form feed), `\r` (carriage return), `\e` (escape), `\\` (backslash), `\"` (double quote), `\xXX` (hex byte), `\dXXX` (decimal byte), `\uXXXX` (16-bit unicode), `\UXXXXXXXX` (32-bit unicode).

### Array
```pike
({1, 2, 3})                      // literal
({})                              // empty
(array)scope_var                  // cast
```

### Mapping
```pike
(["key": "value", "k2": 42])     // literal
([])                              // empty
```

### Multiset
```pike
(< "a", "b", "c" >)              // literal
(< >)                             // empty
```

## Operators

### Arithmetic
`+  -  *  /  %  **` (power)

Division on integers rounds toward negative infinity.

### Comparison
`==  !=  <  >  <=  >=`

### Logical
`!  &&  ||`

### Bitwise
`&  |  ^  <<  >>  ~`

### String
`+` (concatenation), `*` (repetition: `"ab" * 3 → "ababab"`), `/` (split by delimiter), `*` (also: `"." * ({"a","b"}) → "a.b"`)

### Array
`+` (concatenation), `-` (difference), `*` (join: `({"a","b"}) * "," → "a,b"`), `&` (intersection), `|` (union)

### Mapping
`+` (union, right-biased), `-` (difference)

### Range
```pike
arr[..3]     // first 4 elements (indices 0..3)
arr[2..]     // from index 2 to end
arr[2..4]    // indices 2, 3, 4
```

### Negative Indices
```pike
arr[-1]      // last element
arr[-2]      // second to last
```

### Assignment
`=  +=  -=  *=  /=  %=  &=  |=  ^=  <<=  >>=  **=`

### Ternary
`cond ? a : b`

### Other
`..` (range), `->` (member access), `sizeof()`, `typeof()`, `_typeof()`

## Control Structures

### if/else
```pike
if (cond) {
  // ...
} else if (other) {
  // ...
} else {
  // ...
}
```

### while
```pike
while (cond) {
  // ...
}
```

### do-while
```pike
do {
  // ...
} while (cond);
```

### for
```pike
for (int i = 0; i < 10; i++) {
  // ...
}
```

### foreach
```pike
foreach (arr; int idx; mixed val) { }
foreach (mapping; mixed key; mixed val) { }
foreach (multiset; mixed key; int(0..1) present) { }
foreach ("string"; int idx; int char_code) { }
```

### switch
```pike
switch (val) {
  case 1:
    handle_one();
    break;
  case 2..5:
    handle_range();
    break;
  case "a":
  case "b":
    handle_ab();
    break;
  default:
    handle_default();
    break;
}
// Cases fall through without break — use break to prevent (same as C/Java)
```

### break and continue
```pike
while (1) {
  if (done) break;       // exit loop
  if (skip) continue;    // skip to next iteration
}
```

## gauge

`gauge` times the execution of an expression or block and returns elapsed seconds as a float. The expression result is discarded — only timing is returned.

```pike
float elapsed = gauge {
  heavy_computation();
};
write("Took %.3f seconds\n", elapsed);
```

Uses `gethrvtime()` (CPU time), not wall-clock time. For both timing and result, use `gethrvtime()` manually:

```pike
int start = gethrvtime();
string result = compute();
float sec = (gethrvtime() - start) / 1e9;
```

## Functions

### Declaration
```pike
return_type function_name(type1 arg1, type2 arg2) {
  return value;
}

// Optional arguments via default values:
void greet(string name, string greeting "Hello") {
  write("%s, %s!\n", greeting, name);
}
```

### Lambda / Closures
```pike
function(int:int) double = lambda(int x) { return x * 2; };
function(int:int) triple = lambda(int x) { return x * 3; };
array(int) result = map(({1, 2, 3}), double);
```

### Variadic Functions

```pike
// Variadic: extra arguments collected into array
void log(string format, mixed ... args) {
  write(format + "\n", @args);  // @ expands array into arguments
}
log("Hello %s, age %d", "Alice", 30);

// Type annotation for variadic:
void process(int first, string ... rest) {
  // rest is array(string)
}
```

### Function Type Annotation
```pike
function(int, int:string) fn;  // takes two ints, returns string
function(:void) cb;             // takes nothing, returns void
function(int:string) | zero fn; // function or zero
```

## Classes and Inheritance

### Class Declaration
```pike
class MyClass {
  int x;
  string name;

  void create(int x, string name) {
    this->x = x;
    this->name = name;
  }

  string _sprintf(int fmt) {
    return sprintf("MyClass(%d, %s)", x, name);
  }
}
```

### Inheritance
```pike
class Child {
  inherit Parent;
  void create(mixed ... args) {
    ::create(@args); // pass all args to parent
  }
}
```

### Access Modifiers
```pike
class Example {
  public int a;        // accessible from anywhere (default)
  protected int b;     // accessible in class and subclasses
  private int c;       // accessible only in this class
  int d;               // public by default
}
```

### Final
```pike
class Base {
  final void dont_override() { }
}
```

### import vs inherit

```pike
// import: adds symbols to namespace (no parent relationship)
import Stdio;
// Now Stdio.File, Stdio.read_file, etc. are directly accessible

// inherit: creates parent-child relationship
// enables :: operator, allocates storage for parent variables
class MyFile {
  inherit Stdio.File;
  void create(string path) {
    ::create(path, "r");  // call parent constructor
  }
}

// Multiple inheritance
class Combo {
  inherit A;
  inherit B;
  // Resolve ambiguity with renamed inherits:
  // inherit A : MyA;
  // inherit B : MyB;
}

// :: operator variants:
// ::method()       — parent's method
// local::method()  — current class only (not inherited)
// this_program::method() — current program including inherited
// Renamed::method() — specific named inherit
```

## Type Annotations

### Basic Types
```pike
int x;
float y;
string s;
```

### Collection Types
```pike
array(int) ai = ({1, 2, 3});
mapping(string:int) msi = (["a": 1]);
multiset(string) ms = (< "x", "y" >);
```

### Function Types
```pike
function(int:string) fn;        // int → string
function(int, int:int) add;     // (int, int) → int
function(:void) cb;              // void → void
function(mixed...:mixed) varfn;  // variadic
```

### Union Types
```pike
int|string x;           // x is either int or string
int|string|float y;     // one of three
array(int|string) z;    // array of int-or-string elements
```

### Special Types
```pike
mixed x;            // any value
void                // no value / no return
zero                // only the value 0 / UNDEFINED
program p;          // a class/program object
object(Program) o;  // object implementing Program
auto x = expr;      // inferred from initializer
```

## Preprocessor Directives

```pike
#define NAME value
#define MACRO(x) (x * 2)
#undef NAME
#if condition
#elif condition
#else
#endif
#ifdef NAME
#ifndef NAME
#include "file.h"
#include <module.h>
#warning "message"
#error "message"
#string "file.txt"     // include file contents as string literal
#pike 8.0              // set Pike version compatibility
// stringize: #param
// token paste: param##suffix
```

## Type Casting

```pike
(int)3.14          // 3
(string)42         // "42"
(float)"3.14"      // 3.14
(array)"abc"       // ({'a', 'b', 'c'})  — array of character codes
(string)({'a','b'})// "ab"
(array)([mapping])
```

## Operator Overloading (lfuns)

Pike allows operator overloading via specially named methods (lfuns):

```pike
class Vec {
  float x, y;
  void create(float x, float y) { this->x = x; this->y = y; }

  // Arithmetic
  Vec `+(Vec other) { return Vec(x + other->x, y + other->y); }
  Vec `-(Vec other) { return Vec(x - other->x, y - other->y); }
  Vec `-() { return Vec(-x, -y); }  // unary minus: no argument

  // Comparison
  int `==(mixed other) {
    return objectp(other) && x == other->x && y == other->y;
  }
  int `<(Vec other) { return x < other->x; }

  // Index
  float `[](int i) { return i == 0 ? x : y; }
  Vec `[]=(int i, float v) { if (i == 0) x = v; else y = v; return this; }

  // Display
  string _sprintf(int fmt) {
    return fmt == 'O' ? sprintf("Vec(%g, %g)", x, y) : sprintf("%c", fmt);
  }

  // Hash (for mappings/multisets)
  int __hash() { return (int)x * 31 + (int)y; }

  // Cast
  mixed cast(string to) {
    if (to == "array") return ({ x, y });
    return 0;
  }
}
```

### Complete lfun reference

| Operator | Lfun | Notes |
|----------|------|-------|
| `+` | `` `+ `` | Variadic: receives all right-side args |
| `+=` | `` `+= `` | In-place add; called if refs==1 |
| `-` (binary) | `` `- `` | Subtraction |
| `-` (unary) | `` `- `` | No argument |
| `*` | `` `* `` | Variadic |
| `/` | `` `/ `` | Variadic |
| `%` | `` `% `` | Modulo |
| `&` | `` `& `` | Variadic (intersection) |
| `\|` | `` `\| `` | Variadic (union) |
| `^` | `` `^ `` | Variadic (symmetric diff) |
| `<<` | `` `<< `` | Left shift |
| `>>` | `` `>> `` | Right shift |
| `==` | `` `== `` | Equality, return 0 or 1 |
| `<` | `` `< `` | Less-than |
| `>` | `` `> `` | Greater-than |
| `!` | `` `! `` | Logical not |
| `~` | `` `~ `` | Bitwise complement |
| `()` | `` `() `` | Function call |
| `[]` | `` `[] `` | Index access |
| `[]=` | `` `[]= `` | Index assignment |
| `[..]` | `` `[..] `` | Range index |
| `->` | `` `-> `` | Member access |
| `->=` | `` `->= `` | Member assignment |

**Utility lfuns:** `_sprintf(int fmt)`, `_destruct(void|int reason)`, `cast(string type)`, `__hash()`, `_random()`, `_search()`, `_serialize()`, `_deserialize()`, `_types()`, `_get_iterator()`, `_m_delete()`

### Automatic Cloning

Calling a program like a function creates a new object:

```pike
// These are equivalent:
Stdio.File f = Stdio.File("file.txt", "r");
// Same as:
Stdio.File f = Stdio.File();
f->create("file.txt", "r");
```

The program is cloned, then `create()` is called with the arguments.

## Constant Expressions

```pike
constant PI = 3.14159;
enum { RED, GREEN, BLUE }; // RED=0, GREEN=1, BLUE=2
enum { A=1, B=2, C=4 };
```

## this, this_program, this_object()

```pike
// this — typed reference to current object instance
class Foo {
  int x = 42;
  Foo get_self() { return this; }  // typed return
}

// this_program — the current program/class
// Useful for cloning self or referencing own type
class Factory {
  Factory clone() { return this_program(); }
}

// this_object() — untyped object reference
// this_object(1) returns parent object (level up)
object me = this_object();
object parent = this_object(1);
```

## compile, compile_string, compile_file

```pike
// compile() — compile Pike source WITHOUT preprocessor
program p = compile("int x = 1; int get() { return x; }");
object o = p();  // clone
o->get();  // 1

// compile_string() — compile WITH preprocessor
// Source must be a COMPLETE program, not a single expression
program p = compile_string("int x = 42;");

// compile_file() — compile a .pike file
program p = compile_file("module.pike");
```

## Modifiers

```pike
// Access modifiers
public int a;        // accessible anywhere (default)
protected int b;     // class and subclasses (was "static" in old Pike)
private int c;       // this class only
local int d;         // this class only, not inherited

// Behavior modifiers
final void dont_override() { }  // cannot be overridden in subclass
inline int fast() { return 1; }  // suggest inline
optional string maybe;           // optional member (may not exist)
variant void foo(int x) { }     // overloaded variant
__deprecated void old_method() { }  // emits deprecation warning

// NOTE: "static" is deprecated — use "protected"
```

## Compile-Time Assertions (_Static_assert)

Pike provides `_Static_assert()` for compile-time constant expression checks.

### Syntax
```pike
_Static_assert(constant_expression, "error message");
static_assert(constant_expression, "error message");  // alias
```

### Rules
- Both arguments must be compile-time constants.
- If argument 1 is zero/false, compilation fails with argument 2 message.
- If argument 1 is non-zero, compiles to nothing (no runtime effect).

### Examples
```pike
_Static_assert(1, "always passes");
static_assert(sizeof("hello") == 5, "string length mismatch");
static_assert(__MAJOR__ >= 8, "Pike 8.0 or later required");
```

#### Gotchas
- Cannot use runtime values — only compile-time constants
- Not a runtime construct — eliminated by compiler

## Foreach Variants

```pike
// Array with index and value
foreach (({10, 20, 30}); int i; int val)
  write("[%d] = %d\n", i, val);  // [0]=10, [1]=20, [2]=30

// Array value only
foreach (({"a", "b"}); string val)
  write("%s\n", val);

// Mapping with key and value
foreach ((["a": 1, "b": 2]); string key; int val)
  write("%s: %d\n", key, val);

// String with index and character code
foreach ("ABC"; int i; int c)
  write("[%d] = %d\n", i, c);  // [0]=65, [1]=66, [2]=67

// String character only
foreach ("hello"; int c)
  write("%c", c);  // hello

// Multiset
foreach ((<"a", "b">); string key)
  write("%s\n", key);
```

#### Gotchas
- String foreach gives int character codes, NOT single-character strings
- Array index is always int, starting from 0
- Foreach does NOT copy the collection — modifications during iteration are visible

## Range Indexing

```pike
// Inclusive ranges
array a = ({0,1,2,3,4,5});
a[2..4];    // ({ 2, 3, 4 }) — inclusive
a[..3];     // ({ 0, 1, 2, 3 }) — from start to 3
a[3..];     // ({ 3, 4, 5 }) — from 3 to end

// Negative indices (from end)
a[<2..];    // ({ 4, 5 }) — last 2 elements
a[..<2];    // ({ 0, 1, 2, 3 }) — all but last 2
a[<3..<1];  // ({ 3, 4 }) — third-from-last to second-from-last

// String ranges
"hello world"[6..];  // "world"
"hello"[1..<1];       // "ell" — all but first and last

// Range returns NEW copy, never a reference
```

#### Gotchas
- Ranges are INCLUSIVE on both ends (a[2..4] includes index 4)
- Negative index `<N` counts from end: `<1` = last element, `<2` = second-to-last
- `<N..` is useful for "last N elements"
- String ranges return strings, not arrays

## Complete lfun Reference

Pike supports 45 operator overloading functions (lfuns). All are defined in `src/program.h`.

### Operator lfuns
| lfun | Called by | Notes |
|------|-----------|-------|
| `+ | a + args | Variadic. Also ``+ for right-side. += for destructive. |
| `- | a - args | Unary: no args = negation. Right-side: ``- |
| `* | a * args | Variadic. Right-side: ``* |
| `/ | a / args | Variadic. Right-side: ``/ |
| `% | a % args | Modulo. Right-side: `% |
| `& | a & args | Bitwise AND/intersection. Right-side: ``& |
| `| | a | args | Bitwise OR/union. Right-side: ``| |
| `^ | a ^ args | XOR/symmetric diff. Right-side: ``^ |
| `<< | a << n | Left shift. Right-side: ``<< |
| `>> | a >> n | Right shift. Right-side: ``>> |
| `~ | ~a | Bitwise complement. No arguments. |
| `! | !a | Logical not. Returns 0 for true. |
| `== | a == b | Equality. Must pair with __hash. |
| `< | a < b | Less-than comparison. |
| `> | a > b | Greater-than comparison. |

### Indexing lfuns
| lfun | Called by |
|------|----------|
| `[] | a[i] or a[i..j] (if `[..] absent) |
| `[]= | a[i] = val |
| `[..] | a[i..j] (range, if implemented) |
| `-> | a->member |
| `->= | a->member = val |

### Object lifecycle lfuns
| lfun | Called by |
|------|----------|
| create | Object construction (new/clone) |
| destroy | Object destruction (reason: EXPLICIT, NO_REFS, GC, CLEANUP) |
| __INIT | Compiler-generated init (cannot override) |

### Utility lfuns
| lfun | Called by |
|------|----------|
| cast | (type)a — type coercion |
| _sizeof | sizeof(a) |
| _indices | indices(a) |
| _values | values(a) |
| `() | a(args) — callable object |
| _sprintf | sprintf("%O", a) |
| _equal | equal(a, b) — deep equality |
| __hash | hash_value(a), mapping keys |
| _is_type | intp(a), stringp(a) etc type checks |
| _m_delete | m_delete(collection, key) |
| _serialize | encode_value(a) |
| _deserialize | decode_value(data) |

#### Gotchas
- `+ is variadic: receives ALL right-side arguments at once
- ``+ (double backtick) is for RIGHT-side: called when object is on right of +
- `+= is destructive version of `+: called when safe (refs==1)
- If you implement `==, you MUST implement __hash too
- destroy receives reason code: Object.DESTRUCT_EXPLICIT(0), .DESTRUCT_NO_REFS(1), .DESTRUCT_GC(2), .DESTRUCT_CLEANUP(3)

## sprintf/sscanf Complete Reference

### sprintf Format Operators

| Specifier | Output |
|-----------|--------|
| %% | Literal % |
| %b | Signed binary integer |
| %d | Signed decimal integer |
| %u | Unsigned decimal integer |
| %o | Signed octal integer |
| %x | Lowercase hex |
| %X | Uppercase hex |
| %c | Character (int → char). Width=n outputs n bytes in network order |
| %f | Float, locale-dependent (3.140000) |
| %g | Heuristic float (shortest representation) |
| %e | Exponential notation (1.234500e+03) |
| %s | String |
| %q | Quoted string (escapes control chars, backslash, quotes) |
| %O | Debug readable representation (%O on any value) |
| %H | Hollerith string: length byte + string data |
| %t | Type name ("int", "string", "array", etc) |
| %F | Binary IEEE float (%4F=single, %8F=double) |
| %{...%} | Iterate format over array elements |

### sprintf Modifiers

| Modifier | Effect |
|----------|--------|
| 0 | Zero-pad numbers |
| - | Left-justify |
| \| | Center within field |
| + | Show + sign for positive numbers |
| 'x | Use 'x' as pad character |
| * | Next arg is field width/precision |
| < | Reuse same argument |
| @[n] | Select argument by index |

### sprintf Examples
```pike
sprintf("%d %f %s %O %x %c", 42, 3.14, "hi", ({1}), 255, 65)
// "42 3.140000 hi ({ /* 1 element */\n  1\n}) ff A"

sprintf("%-10d", 4711)   // "4711      "
sprintf("%|10d", 4711)   // "   4711   "
sprintf("%010d", 4711)   // "0000004711"
sprintf("%.2f", 3.14159) // "3.14"
sprintf("%t", 42)        // "int"
sprintf("%t", "hi")      // "string"
sprintf("%[0]d, %[0]x, %[0]o", 75) // "75, 4b, 113"
sprintf("%{%d %}", ({1,2,3}))      // "1 2 3 "
```

### sscanf Format Directives

| Directive | Reads |
|-----------|-------|
| %d | Decimal integer |
| %o | Octal integer |
| %x | Hexadecimal integer |
| %D | Auto-detect base (0x=hex, 0=octal, else decimal) |
| %b | Binary integer |
| %f | Float |
| %c | Single character as int |
| %s | String (greedy — use carefully with multiple %s) |
| %S | String (non-whitespace, until whitespace or end) |
| %H | Hollerith string (length byte + data) |
| %F | Binary IEEE float |
| %n | Current position (no input consumed) |
| %{ | Start set/mapping parse |
| %3s | Read exactly 3 characters |
| %[a-z] | Character class — read chars matching set |
| %[^a-z] | Complement class — read chars NOT in set |

### sscanf Examples
```pike
int n = sscanf("42 hello 3.14", "%d %s %f", int i, string s, float f);
// n=3, i=42, s="hello", f=3.14

int n = sscanf("2024-04-18", "%d-%d-%d", int y, int m, int d);
// n=3, y=2024, m=4, d=18

int n = sscanf("hello world", "%s %s", string a, string b);
// WARNING: first %s is greedy — a="hello world", b=0, n=1
// Use %S instead:
int n = sscanf("hello world", "%S %S", string a, string b);
// n=2, a="hello", b="world"
```

#### Gotchas
- sscanf returns number of successfully matched items
- %s is GREEDY — will consume all remaining input. Use %S for whitespace-delimited tokens
- sscanf declarations (int y, string s, etc) work as output variables in the call
- %O is for sprintf only, not sscanf
- %[set] and %[^set] read until a non-matching character
- sscanf with %d and %f auto-stops at non-numeric characters

## Lambda and Closures

```pike
// Anonymous function
function(:int) adder = lambda(int a, int b) { return a + b; };
adder(3, 4);  // 7

// Short form
function(:int) doubler = lambda(int x) { return x * 2; };

// Closures capture variables by reference
int counter = 0;
function(:int) incrementor = lambda() { counter++; return counter; };
incrementor();  // 1
incrementor();  // 2
write("%d\n", counter);  // 2 — closure modified it

// Function type test
functionp(adder);      // 1
functionp(42);         // 0

// Function as argument
array(int) result = map(({1, 2, 3}), lambda(int x) { return x * x; });
// ({ 1, 4, 9 })

// Filter
array(int) even = filter(({1, 2, 3, 4}), lambda(int x) { return !(x % 2); });
// ({ 2, 4 })
```

#### Gotchas
- Closures capture by reference — modifying the closure modifies the original variable
- Lambda creates a new function object each time
- functionp() tests if value is callable
- Use map() and filter() for functional-style operations on arrays

## String Escape Sequences

```pike
// Standard escapes
"\n"     // newline
"\t"     // tab
"\r"     // carriage return
"\\"     // backslash
"\""     // double quote
"\0"     // null byte

// Octal: \0 through \377
"\101"   // "A" (65 decimal)

// Hex: \x00 through \xff
"\x41"   // "A"

// Wide character: \uXXXX or \UXXXXXXXX
"\u0041"    // "A" (creates wide string)
"\u00E9"    // "e" with accent (creates 16-bit string)

// Constant string prefix
#string "filename"  // includes file contents as string literal
```

#### Gotchas
- 'c' is an int (character code), NOT a string — use "c" for string
- \u escapes create wide strings — mixing with 8-bit is fine (auto-widens)
- \0 is NUL byte — Pike strings are NUL-safe (unlike C)

## Built-in Type Predicates and Reflection

```pike
// Type predicates — all return 1/0
intp(42)           // 1 — is int?
floatp(3.14)       // 1 — is float?
stringp("hello")   // 1 — is string?
arrayp(({}))        // 1 — is array?
mappingp(([]))      // 1 — is mapping?
multisetp((<>))     // 1 — is multiset?
objectp(obj)        // 1 — is object?
programp(ADT.Stack) // 1 — is program?
callablep(write)    // 1 — is callable?

// typeof — returns the TYPE of a value
typeof(42);        // int(42..42) — range type for literals
typeof("hi");      // string(104..105) — char range
typeof(({}));       // array(zero)
typeof(0);          // zero (not int — 0 is special)

// _typeof — same as typeof
// object_program — returns the program an object was created from
object_program(this);  // /main

// Compile-time introspection
__LINE__     // current line number
__FILE__     // current file path
__VERSION__  // Pike version (e.g., 8.0)
__MAJOR__    // major version (8)
__MINOR__    // minor version (0)
__BUILD__    // build number (1116)
__DATE__     // compile date string
__TIME__     // compile time string

// Compile-time module checking (preprocessor only)
// #if constant(Protocols.HTTP)
// #if defined(DEBUG)
// #if __MAJOR__ > 7
```

#### Gotchas
- `typeof(0)` returns `zero`, not `int` — 0 is special in Pike's type system
- `typeof(42)` returns `int(42..42)` (range type) — use `intp()` for simple checks
- `callablep()` checks if value can be called (function, object with `()`, etc.)
- `programp()` checks if value is a program/class, `objectp()` checks instances
- `__FILE__` returns the .pike file path, not a module name
- \0 through \377 are octal, NOT decimal — \100 is 64, not 100