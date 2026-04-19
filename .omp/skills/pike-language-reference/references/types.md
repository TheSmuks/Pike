# Pike Type System Reference

## Value Types (Copied on Assignment)

### int
Signed integer. Automatically promoted to `Gmp.mpz` (bignum) when exceeding native range.

```pike
int x = 42;
int hex = 0xFF;
int neg = -1;
// Overflow transparently promotes to bignum:
int big = 1;
for (int i = 0; i < 200; i++) big *= 10; // still works, now Gmp.mpz internally
```

Constants: `Int.NATIVE_MIN`, `Int.NATIVE_MAX` (platform native range), `Int.MAX` (largest representable, may be bignum).

### float
IEEE 754 double-precision floating point.

```pike
float x = 3.14;
// For infinity/NaN, use Math.inf and Math.nan from the Math module

Constants: `Float.MIN`, `Float.MAX`. For infinity/NaN, use `Math.inf` and `Math.nan` from the Math module.

### string
Immutable sequence of bytes/characters. All operations return new strings.

```pike
string s = "hello";
string upper = String.upper_case(s);  // "HELLO" — s unchanged
int len = sizeof(s);                   // 5
```

## Reference Types (Shared on Assignment)

### array
Ordered, dynamically-sized, typed collection. 0-indexed.

```pike
array(int) a = ({1, 2, 3});
a += ({4});           // append — creates new array
a[0] = 99;           // modify in place
sizeof(a);            // length
a[-1];                // last element
a[1..2];              // slice ({2, 3})
sort(a);              // sort in place
reverse(a);           // new reversed array
```

### mapping
Key-value dictionary. Keys can be any type (but typically int or string).

```pike
mapping(string:int) m = (["a": 1, "b": 2]);
m["c"] = 3;           // add/set
m["a"];               // get (returns 0 for missing)
m_delete(m, "a");     // remove, returns removed value
indices(m);           // array of keys
values(m);            // array of values
sizeof(m);            // number of entries
```

### multiset
Unordered collection of unique values. Essentially a mapping(T:int(0..1)).

```pike
multiset(string) s = (< "a", "b" >);
s["a"];               // 1 (present)
s["c"];               // 0 (absent)
s["c"] = 1;           // add
m_delete(s, "a");     // remove
```

### object
Instance of a class. Access members with `->`.

```pike
object o = MyClass(42);
o->method();
o->field;
program p = object_program(o);
object(MyClass) typed = MyClass(42);
```

### program
A class/program value (the blueprint, not an instance).

```pike
program p = MyClass;
object o = p(42);     // instantiate
object_program(o);    // returns MyClass
```

### function
First-class callable. Can reference methods, lambdas, and builtins.

```pike
function(int:int) fn = lambda(int x) { return x * 2; };
fn(21);               // 42
function(:void) cb = my_object->my_method;
```

## Type Annotations

### Union Types
```pike
int|string x;             // either int or string
int|string|float y;       // one of three
array(int|string) z;      // array whose elements are int or string
mapping(string:int|string) m;
```

### Generic Collection Types
```pike
array a;                  // array of mixed
mapping m;                // mapping(mixed:mixed)
multiset ms;              // multiset(mixed)
array(int) ai;            // array of int
mapping(string:int) msi;  // string keys, int values
```

### Function Types
```pike
function(int:string) f;           // one int arg, returns string
function(int, int:int) g;         // two int args, returns int
function(:void) cb;                // no args, no return
function(mixed...:mixed) varfn;    // variadic
```

## Special Types

### mixed
Any value at all.

```pike
mixed x = 42;
x = "hello";
x = ({1, 2, 3});
```

### void
Used only as a return type annotation — function returns nothing.

```pike
void log(string msg) {
  write("%s\n", msg);
}
```

### zero
The type of `0` / `UNDEFINED`. More restrictive than `int` — only the value zero.

```pike
zero x = 0;    // OK
zero y = 1;    // compile error
```

### object(Program)
Constrains the object to implement a specific program.

```pike
object(MyClass) o = MyClass(42);
o->my_class_method(); // type-checked
```

### auto
Type inferred from initializer expression. Available from Pike 8.1 onwards (not available in 8.0).

```pike
auto x = 42;             // int
auto fn = lambda() { };  // function(:void)
```

## zero_type() — Distinguishing Missing Keys and Destructed Objects

`m[key]` returns `0` for both a missing key and a key mapped to value `0`. `zero_type()` distinguishes them and also detects destructed objects.

Returns:
- `0` — value exists (even if the value is `0`)
- `1` — value is `UNDEFINED` (missing key)
- `2` — value is a destructed object

```pike
mapping(string:int) m = (["present": 0]);

m["present"];             // 0 — key exists, value is 0
m["missing"];             // 0 — key does not exist

zero_type(m["present"]);  // 0 — key exists
zero_type(m["missing"]);  // 1 — key is missing

// Correct existence check:
if (!zero_type(m[key])) {
  // key exists in mapping
}
```

## Bignum Auto-Promotion

When an integer operation overflows the native range, Pike transparently promotes to `Gmp.mpz` (arbitrary precision). No code changes needed.

```pike
int big = Int.NATIVE_MAX;
big += 1;  // now a Gmp.mpz, still works with all int operations
big * big;  // still works, no overflow
```

## Singleton Objects: Val.true, Val.false, Val.null

```pike
Val.true     // boolean true singleton
Val.false    // boolean false singleton
Val.null     // null singleton (distinct from integer 0)

if (x == Val.true) { }
if (x == Val.false) { }
// In boolean context, non-zero is truthy, zero/Val.false is falsy
```

## Type Checking

```pike
intp(x)        // is x an int?
floatp(x)      // is x a float?
stringp(x)     // is x a string?
arrayp(x)      // is x an array?
mappingp(x)    // is x a mapping?
multisetp(x)   // is x a multiset?
objectp(x)     // is x an object?
programp(x)    // is x a program?
functionp(x)   // is x a function?
zero_type(x)   // 0 (value exists), 1 (UNDEFINED/missing), 2 (destructed object)
undefinedp(x)  // true if UNDEFINED
```


## Type Coercion (Casting)

```pike
// String <-> Int
(int)"42"        // 42 — decimal only, NO hex/octal
(int)"0xFF"      // 0 — hex NOT parsed by (int) cast
(int)"3.14"      // 3 — truncates, parses integer part only
(int)""           // 0 — empty string casts to 0
(int)"hello"      // 0 — non-numeric string casts to 0
(string)42         // "42"

// Float conversions
(float)"3.14"     // 3.14
(float)"hello"    // 0.0 — non-numeric casts to 0.0
(int)3.14          // 3 — truncates toward zero
(int)3.9           // 3 — truncates, does NOT round
(string)3.14       // "3.14"

// String <-> Array
(array)"hi"       // ({ 104, 105 }) — chars become int codes
(string)({104,105}) // "hi" — int codes become chars

// Mapping <-> Array
(array)(["a":1])  // ({ ({"a", 1}) }) — array of key-value pairs

// sprintf %t gives type name
sprintf("%t", 42)       // "int"
sprintf("%t", 3.14)     // "float"
sprintf("%t", "hi")     // "string"
sprintf("%t", ({1}))     // "array"
sprintf("%t", ([]))      // "mapping"
sprintf("%t", (<1>))     // "multiset"
sprintf("%t", class{}()) // "object"
sprintf("%t", class{})   // "program"
sprintf("%t", lambda(){}) // "function"
sprintf("%t", Val.null)   // "object" — Val.null is a singleton object
```

#### Gotchas
- `(int)"0xFF"` returns 0 — use `sscanf("0xFF", "%x", int v)` for hex
- `(int)` on float TRUNCATES (toward zero), does not round — use `(int)round(f)` if needed
- `(float)"hello"` returns 0.0, not an error
- `sprintf("%t", 0)` returns `"int"`, not `"zero"` — zero is a subtype of int at runtime
- `typeof(42)` returns `int(42..42)` (range type), not just `int`

## typeof() — Compile-Time Type Inspection

```pike
// typeof() returns the TYPE of an expression, not a string
// Returns a type object that describes the static type
typeof(42)     // int(42..42) — literal gets range type
typeof(3.14)   // float
typeof("hi")   // string(104..105) — char range of literal
typeof(({1}))  // array(int(1..1))
typeof(([]))   // mapping(zero:zero)
typeof(write)  // function(string:int) | function(...:int)

// Use with type annotations
int x = 42;
typeof(x)  // int

// string width types
string(8bit) s = "bytes";
typeof(s)  // string(8bit)
```

## Array Arrow Operator and column()

```pike
// Arrow operator on array of mappings — extracts values by key
array(mapping(string:mixed)) data = ({(["x":1]), (["x":2])});
data->x;          // ({ 1, 2 }) — extracts all "x" values
data->y;          // ({ 0, 0 }) — missing keys return 0

// Works on array of objects — calls method
array(objects) objs = ({obj1, obj2});
objs->method();   // array of return values

// column() — extract nth element from array of arrays
array(array(int)) matrix = ({({1,2}), ({3,4})});
column(matrix, 0);  // ({ 1, 3 }) — first column
column(matrix, 1);  // ({ 2, 4 }) — second column
```

#### Gotchas
- `array->key` on mappings returns 0 for missing keys, not an error
- `column()` works on any array of arrays/objects
- Arrow operator on objects calls the method; on mappings, looks up the key