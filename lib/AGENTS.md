# Pike Standard Library (lib/) — AGENTS.md

## Overview
Pure Pike standard library. All modules are written in Pike (.pmod files).
The master object (lib/master.pike.in) handles module resolution and autoloading.

## Module Organization
- `.pmod` files are module files (analogous to Python `.py`)
- `.pmod` directories are module trees
- `module.pmod` inside a directory is the module entry point
- Compat shims in `lib/7.6/` and `lib/7.8/` for older Pike versions

## Key Modules (lib/modules/)

### Core Types & Utilities
|Module|Description|
|---|---|
|Array.pmod|Array operations (reduce, shuffle, sort_array, diff, uniq)|
|Mapping.pmod|Mapping operations (ShadowedMapping)|
|String.pmod|String operations (trim, width, capitalize, fuzzy)|
|Function.pmod|Function utilities|
|Int.pmod|Int constants (NATIVE_MIN, NATIVE_MAX, MAX)|
|Float.pmod|Float constants (MIN, MAX, INFINITY, NAN)|
|Val.pmod|Singleton objects: Val.true, Val.false, Val.null|
|Multiset.pmod|Multiset utilities|

### Data Structures
|Module|Description|
|---|---|
|ADT.pmod|Stack, Queue, List, Priority_queue, History, CritBit, Table|

### I/O & System
|Module|Description|
|---|---|
|Stdio.pmod|File I/O (read_file, write_file, cp, mv, mkdirhier, etc.)|
|Process.pmod|Process management (create_process, popen, system, spawn)|
|Thread.pmod|Threading (Thread, Mutex, Condition, Local, Farm, Queue)|
|Filesystem.pmod|Directory traversal (Traversion)|
|Getopt.pmod|CLI option parsing|
|Arg.pmod|Declarative CLI argument parsing|

### Date & Time
|Module|Description|
|---|---|
|Calendar.pmod|Date/time (ISO default). Objects are immutable. Use .distance() for spans.|

### Crypto & Security
|Module|Description|
|---|---|
|Crypto.pmod|High-level crypto (SHA256, AES, RSA, ECC) wrapping Nettle|
|SSL.pmod|SSL/TLS implementation|

### Networking
|Module|Description|
|---|---|
|Protocols.pmod|HTTP, DNS, SMTP, IMAP, POP3, LDAP, TELNET|
|Sql.pmod|Database abstraction (Sql.Sql with URL-based connection strings)|

### Parsing
|Module|Description|
|---|---|
|Parser.pmod|HTML, XML, C, Pike, CSV parsing|
|MIME.pmod|MIME handling|
|Charset.pmod|Character set conversion|

### Standards
|Module|Description|
|---|---|
|Standards.pmod|URI, UUID, BASE64, PEM, PKCS, X509, JSON|

### Concurrency
|Module|Description|
|---|---|
|Concurrent.pmod|Future/Promise for async operations|

### Other
|Module|Description|
|---|---|
|Colors.pmod|Color parsing (parse_color, rgb_to_hsv)|
|Debug.pmod|Debugging utilities (subject, Profiling)|
|Error.pmod|Error types (Error.Generic and subclasses)|
|Tools.pmod|AutoDoc, Hpf, Install, Legal, sed|
|Web.pmod|Crawler, RSS, Sass|
|Locale.pmod|Language/locale support|
|Local.pmod|Local module override mechanism|

## Code Style
- Tab indentation
- `//!` for autodoc documentation comments
- Types are optional but encouraged
- Use Pike type syntax: `int|string`, `array(string)`, `mapping(string:mixed)`, `function(int:int)`
- Constructor: `void create(mixed ... args)`
- Destructor: `void destroy()`
- Inheritance: `inherit ParentClass;`
- Parent calls: `::create(args);`

## Calendar Gotchas (CRITICAL)
- Calendar objects are IMMUTABLE — all operations return new objects
- Calendar.Day()+n steps by DAYS; Calendar.now()+n does nothing
- a-b does NOT give distance — use .distance()
- ISO weeks start Monday
- set_timezone() is expensive — cache results
- dwim_day/dwim_time throw on failure

## Sql Module Notes
- Always use Sql.Sql() abstraction, not raw drivers
- Connection string: Sql.Sql("mysql://user:pass@host/db")
- Also: sqlite://, pgsql://, odbc://
- query() returns array(mapping(string:mixed))
- Use %d, %s etc. for parameterized queries (auto-escaped)


## Standard Library — Complete Module Index

|Module|Description|
|---|---|
|Array|Array operations (reduce, shuffle, sort_array, diff, uniq)|
|ADT|Stack, Queue, List, Priority_queue, History, CritBit, Table, Heap|
|Calendar|Date/time handling (ISO default). Immutable objects.|
|Charset|Character set conversion|
|Colors|Color parsing (parse_color, rgb_to_hsv)|
|Concurrent|Future/Promise for async operations|
|Crypto|Hashing (SHA256, SHA512), symmetric (AES), asymmetric (RSA, ECC), HMAC, password hashing|
|Debug|Debugging utilities (subject probe, Profiling)|
|Error|Typed error hierarchy — Error.Generic and subclasses|
|Filesystem|Directory traversal (Traversion)|
|Float|Float constants (MIN, MAX, INFINITY, NAN)|
|Function|Function utilities|
|Getopt|CLI argument parsing (find_option, find_all_options)|
|Image|Image decode/load/transform (JPEG, PNG, GIF, TIFF)|
|Int|Int constants (NATIVE_MIN, NATIVE_MAX, MAX)|
|MIME|MIME type handling and parsing|
|Parser|HTML (callback-driven), XML, C, Pike, CSV parsing|
|Process|Process management (run, create_process, spawn, system)|
|Protocols|HTTP (sync/async + server), DNS, SMTP, IMAP, POP3, LDAP|
|Regexp|Regular expression matching|
|Serializer|Flexible serialization with Serializer.Encodeable|
|Sql|Database abstraction — Sql.Sql with URL connection strings|
|SSL|SSL/TLS — SSL.File, SSL.Context, SSL.Port|
|Standards|URI, UUID, BASE64, PEM, PKCS, X509, JSON|
|Stdio|File I/O, Stdio.File, Stdio.read_file/write_file, Stdio.Buffer, Stdio.Port|
|String|String operations (trim, width, capitalize, case_convert)|
|Thread|Threading primitives (Thread, Mutex, Condition, Local, Farm, Queue)|
|Val|Singleton objects: Val.true, Val.false, Val.null|

Detailed patterns: `.agents/skills/pike-language-reference/references/stdlib-patterns.md` (1247 lines).