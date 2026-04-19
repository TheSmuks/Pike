# Pike Standard Library Patterns

Common usage patterns verified against Pike 8.0.1116 source.

## File I/O

### Simple file read/write
```pike
string data = Stdio.read_file("file.txt");
Stdio.write_file("out.txt", data);
Stdio.append_file("log.txt", entry + "\n");
Stdio.cp("src.txt", "dst.txt");
```

### Directory operations
```pike
Stdio.mkdirhier("/a/b/c/");
Stdio.recursive_rm("dir/");
array(string) entries = get_dir("/path");
```

### File handle
```pike
Stdio.File f = Stdio.File("file.txt", "r");
string chunk = f->read(100);  // read 100 bytes
f->write("%d items\n", 42);   // sprintf-style write
f->close();
```

### Stdin/stdout
```pike
Stdio.stdin.read(1024);
Stdio.stdout.write("output\n");
Stdio.stderr.write("error\n");
```

## HTTP

### Simple requests
```pike
Protocols.HTTP.Query q = Protocols.HTTP.get("http://example.com");
string body = q->data();
int status = q->status;
mapping headers = q->headers;

Protocols.HTTP.Query q = Protocols.HTTP.post("http://example.com", "post data");
Protocols.HTTP.Query q = Protocols.HTTP.put(url, data, headers);
```

### URL handling
```pike
Standards.URI url = Standards.URI("http://example.com/path?q=1");
url->host;
url->path;
url->query;
```

## Database (Sql.Sql)

### Query
```pike
object db = Sql.Sql("mysql://user:pass@host/dbname");
array(mapping(string:mixed)) res = db->query("SELECT * FROM users WHERE id=%d", 42);
// res: ({ (["id": 42, "name": "alice"]), ... })
foreach (res; ; mapping row) {
  write("%O\n", row);
}
```

### Write operations
```pike
db->big_query("INSERT INTO users (name) VALUES (%s)", "bob");
db->big_query("UPDATE users SET name=%s WHERE id=%d", "charlie", 1);
db->big_query("DELETE FROM users WHERE id=%d", 99);
```

Parameterized query format specifiers: `%d` (int), `%s` (string), `%f` (float), `%O` (any). Always use parameterized queries — never interpolate values into SQL strings.

## Date and Time (Calendar)

### Current time
```pike
object now = Calendar.ISO.Second();
int unix = now->unix_time();
string http = now->format_http();
string iso = now->format_ymd();  // "2024-01-15"
```

### Day arithmetic
```pike
object today = Calendar.ISO.Day();
object tomorrow = today + 1;        // steps by DAYS
object yesterday = today - 1;
object span = today->distance(other_day); // TimeSpan, NOT day - other_day
int days_between = span->number_of_days();
```

### Parsing
```pike
object d = Calendar.ISO.parse("%Y-%M-%D", "2024-01-15");
```

## Error Handling

### catch
```pike
mixed err = catch {
  risky_operation();
};
if (err) {
  werror("Error: %O\n", err);
  // err is typically an array: ({ "Error message", backtrace... })
}
```

### throw
```pike
throw("Something went wrong");
throw(({ "Error: bad input", backtrace() }));
```

### Error objects
```pike
if (objectp(err) && err->is_generic_error) {
  werror("%s\n", err->message());
}
```

## JSON

```pike
// Encode any Pike value to JSON
string json = Standards.JSON.encode(data);
string pretty = Standards.JSON.encode(data, Standards.JSON.HUMAN_READABLE);

// Decode JSON string to Pike value
mixed data = Standards.JSON.decode(json);
```

## Threads

### Basic threading
```pike
Thread.Thread t = Thread.Thread(lambda() {
  // work in background
});
t->wait(); // join
```

### Mutex
```pike
Thread.Mutex mtx = Thread.Mutex();
Thread.MutexKey key = mtx->lock();
// critical section
key = 0; // unlock (MutexKey released on reassignment or destruction)
```

### Condition variable
```pike
Thread.Condition cond = Thread.Condition();
Thread.Mutex mtx = Thread.Mutex();
Thread.MutexKey key = mtx->lock();
cond->wait(key);   // releases mutex, waits for signal, reacquires
cond->signal();     // wake one waiter
cond->broadcast();  // wake all waiters
```

## Iteration

### Array
```pike
foreach (arr; int idx; mixed val) {
  write("[%d] = %O\n", idx, val);
}
```

### Mapping
```pike
foreach (mapping; mixed key; mixed val) {
  write("%O => %O\n", key, val);
}
```

### Multiset
```pike
foreach (multiset; mixed key; int(0..1) present) {
  write("%O: %d\n", key, present);
}
```

### String (character codes)
```pike
foreach ("hello"; int idx; int char) {
  // char is the character CODE, not a string
  // 'h' → 104, 'e' → 101, etc.
  string ch = sprintf("%c", char);
}
```

### Iterator protocol
```pike
object iter = get_iterator(collection);
while (iter->next()) {
  mixed val = iter->value();
  mixed key = iter->index();
}
```

## String Utilities

```pike
String.Buffer buf = String.Buffer();
buf->add("hello");
buf->add(" ");
buf->add("world");
string result = buf->get();

String.capitalize("hello");       // "Hello"
String.trim_whites("  hello  ");  // "hello"
String.trim_whites(" \t\n");     // ""
String.common_prefix(({"abc", "abd"})); // "ab"

// Note: upper_case and lower_case are GLOBAL functions (not in String module)
upper_case("hello");             // "HELLO" — global predef
lower_case("HELLO");             // "hello" — global predef
```

## Array Utilities

```pike
map(({1, 2, 3}), lambda(int x) { return x * 2; }); // ({2, 4, 6})
filter(({1, 2, 3, 4}), lambda(int x) { return x > 2; }); // ({3, 4})
sort(({3, 1, 2}));     // ({1, 2, 3}) — sorts in place
Array.uniq(({1, 1, 2})); // ({1, 2})
reverse(({1, 2, 3}));  // ({3, 2, 1})
search(({10, 20, 30}), 20); // 1 (index)
has_value(({1, 2, 3}), 2); // 1 (true)
```

## Process Execution

```pike
// Simple command execution
string output = Process.run(({"ls", "-la"}))->stdout;
int exitcode = Process.run(({"make"}))->exitcode;

// Background process
Stdio.File pipe = Stdio.File();
Process.create_process(({"grep", "pattern", "file.txt"}), (["stdout": pipe]));
```

## Serialization / Inspection

```pike
// Inspect any value
string repr = sprintf("%O", value);

// Debug output
write("%O\n", complex_structure);

// Backtrace
array bt = backtrace();
string formatted = describe_backtrace(bt);
```


## Async / Concurrent (Future/Promise)

### Creating a Future
```pike
// From a value
Concurrent.Future f = Concurrent.resolve(42);

// From a computation that may fail
Concurrent.Future f = Concurrent.resolve(42);
f->on_success(lambda(mixed val) { write("Got: %O\n", val); });
f->on_failure(lambda(mixed err) { werror("Failed: %O\n", err); });
```

### Promise (writeable Future)
```pike
Concurrent.Promise p = Concurrent.Promise();
Concurrent.Future f = p->future();  // get read-only view
// ... later ...
p->success(result);  // or p->failure(error)
```

### Chaining
```pike
// map: transform the success value
future->map(lambda(mixed val) { return val * 2; });

// transform: like map but also receives failure reason
future->transform(
  lambda(mixed val) { return ({ val }); },   // success
  lambda(mixed err) { return ([]); }       // failure → recovery
);

// filter: keep only values matching predicate
future->filter(lambda(mixed val) { return val > 0; });

// recover: provide fallback on failure
future->recover(lambda(mixed err) { return default_value; });
```

### Combining futures
```pike
// All: wait for all to succeed
Concurrent.Future all = Concurrent.all(({
  fetch_url("http://a"),
  fetch_url("http://b"),
}));
all->on_success(lambda(array results) { ... });

// Race: first to complete wins
Concurrent.Future first = Concurrent.first_completed(({
  fetch_with_timeout(url1, 5.0),
  fetch_with_timeout(url2, 5.0),
}));

// Traverse: apply async function to each element
Concurrent.Future f = Concurrent.traverse(items, async_fn);
// Fold: reduce with async function
Concurrent.Future f = Concurrent.fold(items, init, async_folder);
```

### Blocking wait
```pike
// Block until resolved
future->wait();
// Block and get value (throws on failure)
mixed value = future->get();
// Non-blocking probe
mixed value = future->try_get();  // UNDEFINED if pending
```

### Gotchas
// - Promise starts in STATE_NO_FUTURE; must call future() before success()
// - Once resolved, calling success()/failure() throws
//   use try_success()/try_failure() for silent no-op on re-resolution
// - Unhandled rejections are logged via master()->handle_error
// - GC'd pending Promises fail with 'Promise broken'

## Cryptography (Crypto)

### Hashing
```pike
// SHA-256
string digest = Crypto.SHA256.hash("data");          // 32-byte binary string
string hex = String.string2hex(digest);               // hex representation

// Using the object API for streaming
Crypto.SHA256 h = Crypto.SHA256();
h->update("chunk1");
h->update("chunk2");
string digest = h->digest();

// Other hashes: Crypto.SHA1, Crypto.SHA384, Crypto.SHA512, Crypto.MD5,
//   Crypto.SHA3_256, Crypto.RIPEMD160
```

### HMAC
```pike
// HMAC-SHA256
string mac = Crypto.SHA256.HMAC("secret_key")("message");
string hex = String.string2hex(mac);

// Verify
string expected = Crypto.SHA256.HMAC("secret_key")(message);
if (mac == expected) { /* verified */ }
```

### Symmetric Encryption (AES)
```pike
// AES-256-CBC
string key = Crypto.SHA256.hash("password");  // 32 bytes
string iv = Crypto.Random.random_string(16);   // 16-byte IV

// Encrypt
string ciphertext = Crypto.AES.CBC.Buffer()->set_encrypt_key(key)
  ->set_iv(iv)->crypt("plaintext")->pad(Crypto.PAD_PKCS7)->drain();

// Decrypt
string plaintext = Crypto.AES.CBC.Buffer()->set_decrypt_key(key)
  ->set_iv(iv)->unpad(ciphertext, Crypto.PAD_PKCS7);

### Password Hashing
```pike
// Hash a password (returns string with algorithm+salt+hash)
string hashed = Crypto.Password.hash("user_password");

// Verify
if (Crypto.Password.verify("input", hashed)) {
  // password matches
}
```

### RSA
```pike
// Generate key pair
Crypto.RSA rsa = Crypto.RSA();
rsa->generate_key(2048);

// Sign and verify
string signature = rsa->pkcs_sign(message, Crypto.SHA256);
int valid = rsa->pkcs_verify(message, Crypto.SHA256, signature);
```

### Gotchas
// - Crypto.*.hash(data) returns raw binary, NOT hex
// - Use String.string2hex() for hex output
// - AES key size must match: 16/24/32 bytes for AES-128/192/256
// - IV must be block_size bytes (16 for AES)
// - Crypto.Password.hash uses Argon2 or bcrypt depending on build

## Serialization (encode_value / decode_value)

### Basic serialization
```pike
// Encode any Pike value to binary string
string encoded = encode_value(("key": 42));

// Decode back
mixed decoded = decode_value(encoded);  // (["key": 42])
```

### What can be encoded
// int, float, string, array, mapping, multiset — always portable
// object — requires Codec (or _serialize/_deserialize lfuns)
// program — bytecode dump (same-version only)
// function — only Pike functions, NOT C builtins
// circular references — handled via back-references

### What CANNOT be encoded
// - Builtin (C) functions — throws error
// - Thread objects, file descriptors
// - Destructed objects — silently becomes UNDEFINED

### Canonical encoding (for comparison)
```pike
// encode_value_canonic produces deterministic ordering
// Suitable for hashing/signatures
string canonical = encode_value_canonic(data);
```

### Custom codec
```pike
// Objects need _serialize/_deserialize for encode_value
class MyClass {
  int x;
  mapping _serialize() { return (["x": x]); }
  void _deserialize(mapping data) { x = data->x; }
}
```

### Gotchas
// - Binary format: magic header \266ke0
// - Programs encoded as bytecode are NOT portable across versions
// - Simple types (int, float, string, array, mapping) ARE portable
// - Use Codec parameter for custom object serialization

## TLS / SSL

### Client-side HTTPS
```pike
// Simple HTTPS request using SSL.File
SSL.Context ctx = SSL.Context();
Stdio.File raw = Stdio.File();
raw->connect("example.com", 443);
SSL.File conn = SSL.File(raw, ctx);
conn->connect("example.com");  // hostname for SNI
conn->write("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n");
string response = conn->read(10000);
conn->close();
```

### Server-side TLS
```pike
// Create context with certificate and key
SSL.Context ctx = SSL.Context();
ctx->add_cert(Stdio.read_file("server.pem"));
// Or load cert and key separately:
// ctx->add_cert(Tools.PEM.pem_msg("cert.pem"),
//   Tools.PEM.pem_msg("key.pem"));

// Accept TLS connections
Stdio.File client = port->accept();
SSL.File tls = SSL.File(client, ctx);
tls->set_blocking();
if (tls->accept()) {
  string data = tls->read(4096);
  tls->write("HTTP/1.1 200 OK\r\n\r\nHello TLS");
  tls->close();
}
```

### Context configuration
```pike
SSL.Context ctx = SSL.Context();
// Require certificate verification
ctx->verify_certificates = 1;
ctx->require_trust = 1;
// Set minimum TLS version
ctx->preferred_protocols = ({ SSL.Constants.PROTOCOL_TLS_1_2 });
// Add CA certificates
ctx->add_trust_file("/etc/ssl/certs/ca-certificates.crt");
```

### Gotchas
// - SSL.File takes over the underlying stream's callbacks
// - Always call connect() (client) or accept() (server) after creating SSL.File
// - Nonblocking mode is default — call set_blocking() for sync code
// - Use get_peer_certificates() to inspect client certs

## HTTP Server (Comprehensive)

### Basic HTTP Server
```pike
void handle_request(Protocols.HTTP.Server.Request req) {
  req->response_and_finish(([
    "error": 200,
    "type": "text/plain",
    "data": "Hello from Pike!"
  ]));
}

Protocols.HTTP.Server.Port server =
  Protocols.HTTP.Server.Port(handle_request, 8080);
return -1; // keep backend alive
```

### HTTPS Server
```pike
// Auto-generates self-signed cert if none provided
Protocols.HTTP.Server.SSLPort server =
  Protocols.HTTP.Server.SSLPort(handle_request, 8443);
return -1;
```

### Static File Server
```pike
Protocols.HTTP.Server.Port server =
  Protocols.HTTP.Server.Port(
    Protocols.HTTP.Server.Filesystem.look_at_files("./public", 0, 0),
    8080);
return -1;
```

### Request Object Key Fields
- `request_type` — "GET", "POST", etc
- `not_query` — URL path
- `query` — query string after ?
- `variables` — parsed GET+POST params (mapping)
- `cookies` — parsed cookies (mapping)
- `body_raw` — raw request body
- `request_headers` — all headers
- `get_ip()` — client IP address

### response_and_finish mapping
```pike
req->response_and_finish(([
  "error": 200,           // HTTP status code
  "type": "text/html",   // Content-Type
  "data": "<h1>Hi</h1>", // body
  "extra_heads": (["X-Custom": "value"]),
  "modified": time(1),   // Last-Modified
]));
```

### JSON API Example
```pike
void api_handler(Protocols.HTTP.Server.Request req) {
  if (req->not_query == "/api/data" && req->request_type == "GET") {
    string json = Standards.JSON.encode((["status": "ok"]));
    req->response_and_finish(([
      "type": "application/json",
      "data": json
    ]));
  } else {
    req->response_and_finish((["error": 404, "data": "Not Found"]));
  }
}
```

#### Gotchas
- Port defaults to 80, SSLPort to 443
- Request handler MUST call response_and_finish() for every request
- variables mapping contains BOTH GET and POST params
- File streaming via `"file"` key in response mapping
- For HTTPS, provide key+certificate to avoid self-signed auto-generation
## HTML Parsing (Parser.HTML)

### Basic callback-driven parsing
```pike
Parser.HTML p = Parser.HTML();
p->add_tag("br", lambda(object t, mapping attr, string content) {
  return "<br />";  // self-close void tags
});
p->add_container("b", lambda(object t, mapping attr, string content) {
  return "<strong>" + content + "</strong>";  // transform tags
});
string result = p->finish("<b>hello</b> <br>")->read();
```

### Attribute handling
```pike
p->add_container("a", lambda(object t, mapping attr, string content) {
  // attr contains all tag attributes: (["href": "url", "class": "link"])
  return sprintf("<a href=\"%s\">%s</a>", attr->href || "#", content);
});
```

### Entity callbacks
```pike
p->add_entity("amp", lambda() { return "&"; });
p->add_entity("lt", lambda() { return "<"; });
// Also: add_entities((["amp": "&", "lt": "<"]))
```

### Streaming/incremental parsing
```pike
Parser.HTML p = Parser.HTML();
p->add_container("p", my_handler);
// Feed chunks without parsing
p->feed(chunk1, 0);
p->feed(chunk2, 0);
// Trigger parse and get result
string result = p->finish()->read();
```

### Mixed mode (structured output)
```pike
Parser.HTML p = Parser.HTML();
p->mixed_mode(1);  // enable mixed mode
p->add_container("item", lambda(object t, mapping a, string c) {
  return ({ "item", c });  // return non-string for mixed output
});
array mixed = p->finish("text <item>data</item> more")->read();
// ({ "text ", ({ "item", "data" }), " more" })
```

### Gotchas
// - add_tag: void/empty tags (br, img, hr, input) — no closing tag expected
// - add_container: paired tags (div, p, b) — receives content between open/close
// - feed(s, 0) queues without parsing; feed(s) or feed(s, 1) parses immediately
// - Callbacks: tag gets (parser, attrs), container gets (parser, attrs, content)
// - Return string to replace content, array for mixed mode, 0 to discard


## ADT — Data Structures

### Stack
```pike
ADT.Stack s = ADT.Stack();
s->push(1);
s->push(2);
int top = s->top();    // 2 (peek without removing)
int val = s->pop();     // 2 (remove and return)
int empty = s->isempty();
```

### Queue
```pike
ADT.Queue q = ADT.Queue();
q->put("first");
q->put("second");
string front = q->peek();  // "first" (peek)
string val = q->get();      // "first" (remove and return)
```

### Table
```pike
ADT.Table.table t = ADT.Table.table(
  ({ ({"Alice", 30}), ({"Bob", 25}), ({"Charlie", 35}) }),
  ({"name", "age"})
);
// All operations return a NEW table (immutable pattern)
t->sort("age");          // sort ascending by age
t->rsort("age");         // sort descending
t->select("name");       // project to one column
t->where("age", lambda(int a) { return a > 28; }); // filter
t->col("name");          // array of column values
t->row(0);               // array: ({"Alice", 30})
sizeof(t);               // number of rows
indices(t);              // column names
```

### Heap (Priority Queue)
```pike
ADT.Heap h = ADT.Heap();
h->push(5); h->push(1); h->push(3);
mixed smallest = h->pop();  // 1 (min-heap)
mixed next = h->peek();     // next without removing
int empty = (sizeof(h) == 0);
```

### CritBit Tree
```pike
ADT.CritBit.Tree tree = ADT.CritBit.Tree();
tree->insert("hello", 1);
tree->insert("world", 2);
int|zero val = tree->get("hello");  // 1
tree->remove("hello");
array keys = tree->keys();
array values = tree->values();
// Prefix search:
ADT.CritBit.Tree sub = tree->get_subtree("he");
```

### Gotchas
// - ADT.Table operations are non-destructive — each returns a new table
// - ADT.Heap is a min-heap by default (smallest value pops first)
// - ADT.CritBit.Tree is a string-keyed trie, not a general-purpose map
// - ADT.Stack.pop() on empty stack throws error

## Math Module

```pike
// Constants
float pi = Math.pi;
float inf = Math.inf;
float nan = Math.nan;

// Math.Angle may not exist in all builds — check:
// if (programp(Math.Angle)) { ... }

## sprintf Format Specifiers

```pike
// Common specifiers:
write("%d", 42);          // "42" — integer
write("%s", "hello");     // "hello" — string
write("%f", 3.14);        // "3.140000" — float
write("%x", 255);         // "ff" — hexadecimal
write("%o", 8);           // "10" — octal
write("%b", 10);          // "1010" — binary
write("%c", 65);          // "A" — character from code
write("%O", value);       // debug representation
write("%q", "str\"ing");  // quoted string: "str\"ing"
write("%%");               // literal %

// Width and precision:
write("%10d", 42);        // "        42" (right-align, width 10)
write("%-10d", 42);       // "42        " (left-align)
write("%05d", 42);        // "00042" (zero-pad)
write("%.2f", 3.14);      // "3.14" (2 decimal places)
write("%10.2f", 3.14);    // "      3.14"

// Container formatting:
write("%{%s %}", ({"a","b","c"}));  // "a b c "
write("%{%d,%}", ({1,2,3}));          // "1,2,3,"
```

### Gotchas
// - %O is for debugging: outputs full Pike representation
// - %s on non-string values calls _sprintf('s') or gives error
// - %% is literal percent — needs no argument
// - %q adds quotes and escapes: useful for generating source code

## sscanf Patterns

```pike
// sscanf returns number of successfully matched specifiers
int n = sscanf("hello 42", "%s %d", string name, int age);
// n == 2, name == "hello", age == 42

// %s is greedy — matches everything up to next literal or end
sscanf("foo/bar/baz", "%s/%s", string a, string b);
// a == "foo", b == "bar/baz"

// %[^x] matches characters NOT in set
sscanf("hello world", "%[^ ] %s", string first, string rest);
// first == "hello", rest == "world"

// %d for integers, %f for floats
sscanf("3.14", "%f", float f);  // f == 3.14

// Literal percent: %%
sscanf("100%", "%d%%", int val);  // val == 100, n == 1

// %H for hex-encoded strings
sscanf("48656c6c6f", "%H", string decoded);  // decoded == "Hello"

// %{format%} — matched repeatedly into array of arrays
sscanf("1 2 3", "%{%d%}", array matches);
// matches == ({ ({1}), ({2}), ({3}) })
```

### Gotchas
// - sscanf returns the number of matched specifiers, NOT the values
// - %s is greedy: in "%s/%s", first %s takes everything before LAST /
// - %[^x] is a character class: %[^abc] matches anything except a,b,c
// - When sscanf fails partially, unmatched variables are unchanged

## Process Module (Extended)

### Process.run (capture output)
```pike
mapping result = Process.run(({"ls", "-la"}));
// result: (["stdout": "...", "stderr": "", "exitcode": 0])

// With working directory:
mapping r = Process.run(({"make"}), (["cwd": "/tmp/build"]));

// With stdin input:
mapping r = Process.run("awk '{print toupper($0)}'",
  (["stdin": "hello\nworld\n"]));
```

### Process.create_process (full control)
```pike
Stdio.File stdout_pipe = Stdio.File();
Process.create_process proc = Process.create_process(
  ({"grep", "pattern"}),
  (["stdout": stdout_pipe, "cwd": "/tmp"])
);
// proc->wait() returns exit code
// Read from stdout_pipe for captured output
```

### Process.spawn (shell command)
```pike
// Spawns via /bin/sh -c
object proc = Process.spawn("ls -la | grep foo");
int exitcode = proc->wait();
```

### Process.system (blocking shell)
```pike
int exitcode = Process.system("make install");
```

### Gotchas
// - Process.run captures ALL output in memory — not for huge output
// - Process.run does NOT let you redirect stdout/stderr yourself
// - Use Process.create_process for streaming I/O
// - Process.spawn uses shell — be careful with user input (injection risk)


## Error Module

### Error.Generic (base class)
```pike
// Creating errors
Error.Generic err = Error.Generic("Something went wrong");
write("%s\n", err->message());   // "Something went wrong"
write("%s\n", err->describe());  // message + formatted backtrace

// Normalizing any value to an Error
Error.Generic e = Error.mkerror("string error");
Error.Generic e = Error.mkerror(({ "array error", backtrace() }));
```

### Error subclasses
```pike
// Each has is_*_error constant for type checking
if (objectp(err) && err->is_index_error) { /* ... */ }

// Available subclasses:
// Error.Index        — is_index_error, fields: index, value
// Error.BadArgument  — is_bad_argument_error, fields: which_argument, expected_type, got_value
// Error.Math         — is_math_error, field: number
// Error.Resource     — is_resource_error, fields: resource_type, howmuch
// Error.Permission   — is_permission_error, field: permission_type
// Error.Decode       — is_decode_error, field: decode_string
// Error.Cpp          — is_cpp_error
// Error.Compilation  — is_compilation_error
// Error.ModuleLoad   — is_module_load_error, fields: path, reason
```

### Gotchas
// - Do NOT check error type with typeof() — use is_*_error flags
// - Error.mkerror() normalizes any value: string, array, object
// - Error subclasses do NOT include Error.Indices, Error.Null, Error.Undefined (those don't exist)

## Val Module (Sentinel Values)

```pike
// Val.true, Val.false, Val.null — singleton objects
// Used where Pike's native 0/1 integers are ambiguous (SQL NULL vs false)

// SQL queries return Val.null for NULL database values
// JSON decode uses Val.true/Val.false for boolean values

mixed val = Val.null;
if (val == Val.null) write("it is null\n");
if (val == Val.true) write("it is true\n");

// _sprintf representation
write("%O\n", Val.true);   // Val.true
write("%O\n", Val.false);  // Val.false
write("%O\n", Val.null);   // Val.null
```

### Gotchas
// - Val.true/Val.false/Val.null are singleton objects, NOT integers
// - Do NOT use Val.TRUE or Val.FALSE (uppercase doesn't exist)
// - Use (val == Val.null) not objectp(val) to check

## Getopt (CLI Argument Parsing)

### find_option
```pike
int main(int argc, array(string) argv) {
  // Single option: short form -v, long --verbose, no argument (flag)
  int verbose = Getopt.find_option(argv, "v", "verbose");

  // Option with argument: -f file, --file=file
  string filename = Getopt.find_option(argv, "f", "file", "MYAPP_FILE", "default.txt");

  // Array of aliases
  string output = Getopt.find_option(argv, ({"o", "O"}), ({"output", "out"}));
}
```

### find_all_options (batch)
```pike
array opts = Getopt.find_all_options(argv, ({
  // ({ name, type, aliases, env_var, default })
  ({"verbose", Getopt.NO_ARG, ({"-v", "--verbose"})}),
  ({"output",  Getopt.HAS_ARG, ({"-o", "--output"})}),
  ({"level",   Getopt.MAY_HAVE_ARG, ({"-l", "--level"}), "LEVEL", "1"}),
}));
// opts: ({ ({"verbose", 1}), ({"output", "file.txt"}) })

array remaining = Getopt.get_args(argv);
```

### Constants
// Getopt.HAS_ARG = 1     — option requires argument
// Getopt.NO_ARG = 2      — flag, no argument
// Getopt.MAY_HAVE_ARG = 3 — argument optional

### Gotchas
// - find_option modifies argv IN-PLACE (replaces parsed options with 0)
// - Only first occurrence parsed per call to find_option
// - Use get_args() after all find_option calls to get remaining args

## Filesystem Module

```pike
// Filesystem.Base — abstract filesystem interface
// Filesystem.System — local filesystem implementation

object fs = Filesystem.System();
write("cwd: %s\n", fs->cwd());
fs->cd("/tmp");
array(string) files = fs->get_dir();
Stat st = fs->stat("file.txt");

// Recursive traversal
void walk_tree(object fs, string path) {
  foreach (fs->get_dir(path);; string name) {
    string full = path + "/" + name;
    Stat st = fs->stat(full);
    if (st && st->isdir) walk_tree(fs, full);
    else write("%s\n", full);
  }
}
```

## MIME Module

```pike
// Parse MIME message
MIME.Message msg = MIME.Message(raw_mime_data);
write("type: %s\n", msg->type());          // "text/plain"
write("params: %O\n", msg->params());      // (["charset": "utf-8"])
write("headers: %O\n", msg->headers());    // all headers
string body = msg->getdata();              // body content

// Multipart messages
array(MIME.Message) parts = msg->body_parts;
```

## Predef Functions

### Collection operations
```pike
// Array construction
array a = aggregate(1, 2, 3);     // same as ({1,2,3})
array z = allocate(10);            // ({0,0,0,0,0,0,0,0,0,0})
array s = allocate(3, "x");        // ({"x","x","x"}) — deep copies init

// Extracting data
array col = column(({({"a",1}),({"b",2})}), 1); // ({1,2})
array r = rows((["x":1,"y":2]), ({"x","z"})); // ({1,UNDEFINED})
array t = transpose(({{1,2}),({3,4})}); // ({({1,3}),({2,4})})

// Search
int idx = search(({10,20,30}), 20);  // 1 (returns -1 if not found)
int found = has_value(({1,2,3}), 2); // 1

// String checks
has_prefix("hello world", "hello");  // 1
has_suffix("hello world", "world");  // 1

// Copy and compare
mapping copy = copy_value(original); // deep copy
equal(({1,2}), ({1,2}));            // 1 (structural equality)
```

### Type checking
```pike
// Each returns 1 (true) or 0 (false)
intp(42);          // 1
floatp(3.14);      // 1
stringp("hi");     // 1
arrayp(({}));      // 1
mappingp(([]));    // 1
multisetp((<>));   // 1
objectp(Stdio.File()); // 1
functionp(write);  // 1
programp(Stdio.File); // 1
undefinedp(UNDEFINED); // 1

// zero_type — distinguishes kinds of zero
// 0 = value exists and is 0
// 1 = UNDEFINED (e.g., missing mapping key)
// 2 = destructed object
mapping m = ([]);
zero_type(m["missing"]);  // 1
```

### Other predef
```pike
// Random
int r = random(100);        // 0..99
float rf = random(1.0);     // 0.0..1.0

// Hash any value
int h = hash_value(data);

// Object introspection
program p = object_program(obj);

// Destruction
destruct(obj);  // force object destruction

// this_object() — reference to current object
// this_program  — current program (class)
```

### Gotchas
// - search() returns -1 for not-found (strings/arrays), UNDEFINED for mappings
// - equal() checks structural equality, == checks identity
// - copy_value() deep-copies — O(n) for nested structures
// - allocate(n, init) deep-copies init for each element
// - random(n) for int returns 0..n-1, random(f) for float returns 0.0..f

## Image Module

```pike
// Decode image from binary data (auto-detects format)
Image.Image img = Image.decode(image_data);

// Load from file
Image.Image img = Image.load("photo.jpg");

// Decode with metadata
mapping info = Image._decode(data);
// info->image (Image.Image), info->alpha, info->format

// Create blank image
Image.Image img = Image.Image(800, 600);  // 800x600 black
img->setcolor(255, 0, 0);  // red
img->fill(0, 0, 800, 600);

// Get dimensions without full decode
array dims = Image.Dims.get_JPEG(Stdio.File("photo.jpg", "r"));
// dims: ({width, height})

// Image operations
img->scale(0.5);     // scale to 50%
img->rotate(90);     // rotate 90 degrees
img->mirrorx();      // horizontal flip
img->grey();         // convert to grayscale
string jpeg = Image.JPEG.encode(img);  // encode to JPEG
string png = Image.PNG.encode(img);    // encode to PNG
```

### Gotchas
// - Image.Image and Image.Layer are C-level types, not in .pmod files
// - Image.decode() returns 0 (not error) for unrecognized data
// - Image.Dims functions take Stdio.File, not string data

## Function Module

```pike
// Function.splice_call — apply array as arguments
mixed result = Function.splice_call(function, args);
// Like function(@args) but handles edge cases

// Function.call_callback — safe callback invocation
Function.call_callback(callback, args...);
// Does nothing if callback is UNDEFINED
```

## Program Module

```pike
// Check inheritance
int inherits = Program.inherits(child_prog, parent_prog);
// Check implementation
int implements = Program.implements(prog, interface_prog);
// Defined functions
array(string) indices = indices(program_obj);
```

## Standards Module

```pike
// URI parsing
Standards.URI url = Standards.URI("https://example.com/path?q=1#frag");
url->scheme;    // "https"
url->host;      // "example.com"
url->path;      // "/path"
url->query;     // "q=1"
url->fragment;  // "frag"

// UUID generation
string uuid = Standards.UUID.make_version1();
string uuid4 = Standards.UUID.make_version4();

// BASE64
string encoded = Standards.BASE64.encode(data);
string decoded = Standards.BASE64.decode(encoded);
```

## Debug Module (Real APIs)

```pike
// Memory introspection
mapping mem = Debug.memory_usage();
write("%s\n", Debug.pp_memory_usage());  // pretty-printed table
mapping counts = Debug.count_objects();  // count by type

// Debug.Subject — lfun call probe (NOT a topic filter)
// Creates an object that logs all lfun invocations to stderr
object probe = Debug.Subject("myprobe");
// Any operation on probe prints to stderr:
// probe + 1;  → stderr: (myprobe) `+(1)
// (string)probe; → stderr: (myprobe) _sprintf(115,...)

// Object inspection
array objects = Debug.map_all_objects(lambda(object o) { return o; });
int size = Debug.size_object(obj);
```

### Gotchas
// - Debug.Subject is a lfun PROBE, not a topic/observer pattern
// - Debug.debug and other RTL-debug features require --with-rtldebug build
// - Debug.memory_usage() returns raw mapping; use pp_memory_usage() for readable output

## Regexp

```pike
// SimpleRegexp (built-in, 8-bit only)
Regexp.SimpleRegexp r = Regexp("h.llo");
r->match("hello");   // 1
r->split("hello");   // array of captured groups
// Note: replace() is not available on SimpleRegexp
// Use glob-style replace via replace() global function instead

// Convenience: Regexp() returns SimpleRegexp
object r = Regexp("^[a-z]+$");
if (r->match(input)) { /* valid lowercase */ }

// PCRE (if available): Regexp.PCRE
// Regexp.match(pattern, data) — shorthand
// Regexp.split(pattern, data) — shorthand
```

### Gotchas
// - SimpleRegexp only handles 8-bit strings (no NUL, no wide chars)
// - Max 39 capturing groups in SimpleRegexp
// - Use Regexp.PCRE for Unicode/complex patterns (if built with PCRE)

## Colors Module

```pike
// Parse named or hex color
array(int) rgb = Colors.parse_color("red");      // ({255, 0, 0})
array(int) rgb2 = Colors.parse_color("#ff0000"); // ({255, 0, 0})

// Convert between RGB and HSV
array(int) hsv = Colors.rgb_to_hsv(255, 128, 0);
array(int) rgb_back = Colors.hsv_to_rgb(@hsv);

// Hex string via sprintf
string hex = sprintf("#%02x%02x%02x", @rgb);
```

## Int and Float Modules

```pike
// Int module
write("NATIVE_MAX: %O\n", Int.NATIVE_MAX);  // 9223372036854775807 (64-bit)
// Int.MAX does not exist — Pike integers auto-promote to bignum

// Float module
write("MAX: %O\n", Float.MAX);    // 1.79769e+308
write("MIN: %O\n", Float.MIN);    // 2.22507e-308
```

## Serializer Module

```pike
// Serializer.Encodeable — mixin for encode_value/decode_value
class MyClass {
  inherit Serializer.Encodeable;
  int x;
  string name;
  void create(int x, string name) {
    this->x = x;
    this->name = name;
  }
}

MyClass obj = MyClass(42, "test");
string encoded = encode_value(obj);
MyClass decoded = decode_value(encoded);
// decoded->x == 42, decoded->name == "test"
```

## Protocols.HTTP Async

```pike
// Async HTTP request
Protocols.HTTP.Query q = Protocols.HTTP.Query();
q->set_callbacks(
  lambda(Protocols.HTTP.Query q, mixed ... extra) {
    write("Got %d bytes\n", sizeof(q->data()));
  },
  lambda(Protocols.HTTP.Query q, mixed ... extra) {
    werror("HTTP request failed\n");
  }
);
q->async_request("example.com", 80, "GET / HTTP/1.0\r\n\r\n");

// Convenience async methods
Protocols.HTTP.do_async_method("GET", url, 0, 0, q);
```

### Gotchas
// - Must call set_callbacks() BEFORE async_request()
// - Default timeout is 120 seconds
// - Use Pike.DefaultBackend for async I/O

## HTTP Async (Protocols.HTTP)

### Low-level: Protocols.HTTP.Query
```pike
Protocols.HTTP.Query q = Protocols.HTTP.Query();

void headers_ok(Protocols.HTTP.Query q) {
  write("Status: %d %s\n", q->status, q->status_desc);
  q->async_fetch(body_done);
}

void body_done(Protocols.HTTP.Query q) {
  write("Body: %d bytes\n", sizeof(q->data()));
}

void request_failed(Protocols.HTTP.Query q) {
  werror("Request failed: errno=%d\n", q->errno);
}

q->set_callbacks(headers_ok, request_failed);
q->async_request("example.com", 80,
  "GET / HTTP/1.0",
  (["host": "example.com"]));
return -1; // keep backend running
```

Key Query async methods:
- `set_callbacks(function ok, function fail, mixed... extra)`
- `async_request(host, port, request_line, headers, data)`
- `async_fetch(callback)` — called from headers callback to fetch body
- `.status`, `.status_desc`, `.headers`, `.data()` — fields after completion

### Mid-level: do_async_method
```pike
Protocols.HTTP.Query q = Protocols.HTTP.Query();
q->set_callbacks(ok_cb, fail_cb);
Protocols.HTTP.do_async_method("GET", "http://example.com",
  (["q": "test"]), (["accept": "text/html"]), q);
return -1;
```

### High-level: Protocols.HTTP.Session (connection pooling)
```pike
Protocols.HTTP.Session s = Protocols.HTTP.Session();
// Connection pooling, keep-alive, cookies, redirect following
// Use do_sync(), do_async(), do_thread() on session requests
```

#### Gotchas
// - Must return -1 from main() to keep backend alive for async callbacks
// - Query.ok=1 means TCP success; a 404 still has ok=1
// - Headers are lowercased by the parser
// - HTTPS requires `#if constant(SSL.File)`
// - do_async_method does NOT create a Query — create and set_callbacks first

## Regexp (Detailed)

### Regexp.SimpleRegexp (always available, C builtin)
```pike
Regexp.SimpleRegexp r = Regexp("^[a-z]+$");
int matched = r->match("hello");        // 1
array(string) hits = r->match(({"hello", "World"}));  // ({ "hello" })
array result = r->split("abc-123");    // captures: ({ "abc", "123" })
string out = r->replace("abc 123 def", "NUM");  // "abc NUM def"
string out2 = r->replace("abc 123", lambda(string m) { return "[" + m + "]"; });
```

Syntax: `. [] () * + | ^ $ \\ \< \>` (word boundaries)
Max 39 capturing groups. `.` matches any char including newline.

### Regexp.PCRE (if compiled with libpcre)
```pike
#if constant(Regexp.PCRE)
Regexp.PCRE.Plain r = Regexp.PCRE.Plain("\\b\\w+@\\w+");
int hit = r->match("test");
array a = r->split("subject");        // subpatterns only
array b = r->split2("subject");       // full match + subpatterns
string s = r->replace("subject", "repl");
string s2 = r->replace1("subject", "repl");  // first only
r->matchall("subject", lambda(array m, array pos) { ... });

// Shorthand functions
Regexp.match("pattern", "data");
Regexp.split("pattern", "data");
Regexp.replace("pattern", "data", "repl");
#endif
```

#### Gotchas
// - SimpleRegexp only handles 8-bit strings — no NUL, no wide chars
// - SimpleRegexp.split() returns 0 on no match, not empty array
// - SimpleRegexp.replace() exists (contrary to earlier docs) — takes string or function callback
// - PCRE shorthands create temporary objects — avoid in hot loops
// - PCRE availability: check with `#if constant(Regexp.PCRE)`

## Yabu — Transaction Database

Yabu is a Pike-native transactional key-value store using encode_value serialization.

```pike
// Open database (mode: r=read, w=write, c=create, t=transactions)
Yabu.DB db = Yabu.DB("/path/to/dbdir", "rwc");

// Get/create table
Yabu._Table users = db->table("users");

// CRUD via indexing syntax
users["alice"] = (["name": "Alice", "age": 30]);
mixed data = users["alice"];
m_delete(users, "alice");

// Direct methods
t->set("key", value);
mixed val = t->get("key");
t->delete("key");
array(string) keys = t->list_keys();
t->sync();          // force flush to disk
t->statistics();    // (["keys": N, "size": N])

// Transactions (requires "t" in mode)
Yabu.DB db = Yabu.DB("path", "rwct");
Yabu.Transaction tx = db->table("acct")->transaction();
tx->set("alice", (tx->get("alice") || 0) - 100);
tx->set("bob", (tx->get("bob") || 0) + 100);
tx->commit();     // throws on conflict
tx->rollback();   // discard
```

#### Gotchas
// - Only one DB per directory (process lock via lock.pid)
// - Values serialized with encode_value — must be encodable types
// - Transactions detect conflicts: commit() throws if key was modified externally
// - Sync is automatic after 128 writes; manual sync() forces immediate

## Colors Module (Detailed)

```pike
// Named color to RGB
array(int) rgb = Colors.parse_color("red");          // ({ 255, 0, 0 })
array(int) rgb = Colors.parse_color("#ff0000");       // ({ 255, 0, 0 })
array(int) rgb = Colors.parse_color("unknown", ({42,42,42})); // fallback

// Color name lookup
string name = Colors.color_name(({255, 0, 0}));     // "red"
string hex = Colors.color_name(({128, 64, 32}));    // "#804020"

// Conversions (accept individual args OR single array arg)
array hsv = Colors.rgb_to_hsv(255, 0, 0);     // ({ 0, 255, 255 })
array rgb = Colors.hsv_to_rgb(({0, 255, 255})); // ({ 255, 0, 0 })
array cmyk = Colors.rgb_to_cmyk(255, 0, 0);    // ({ 0, 100, 100, 0 })
array rgb = Colors.cmyk_to_rgb(({0, 100, 100, 0})); // ({ 255, 0, 0 })

// Hex string via sprintf
string hex = sprintf("#%02x%02x%02x", @rgb);
```

#### Gotchas
// - parse_color returns black ({0,0,0}) for unknown colors unless default provided
// - color_name returns hex #rrggbb if no named color matches
// - All conversions accept both (r,g,b) args and ({r,g,b}) array
// - Requires Image module (delegates to Image.Color internally)

## Annotations

Pike 8.0 does NOT have a built-in annotation system. No `@annotation` syntax, no `Pike.Annotations` module.

Workaround patterns:
```pike
// Convention-based doc comments
//! @param x - The value
//! @returns The result
int process(int x) { return x * 2; }

// Wrapper class with metadata
class Annotated {
  mapping(string:mixed) attributes = ([]);
  function(:mixed) func;
  void create(function(:mixed) f, mapping(string:mixed) attrs) {
    func = f; attributes = attrs;
  }
  mixed `()(mixed ... args) { return func(@args); }
}
```

## Int Module (Detailed)

```pike
// Constants
Int.NATIVE_MIN    // -9223372036854775808 (64-bit)
Int.NATIVE_MAX    // 9223372036854775807
// Int.MAX does NOT exist — Pike auto-promotes to bignum

// Integer infinity
Int.inf + 1;         // Int.inf
Int.inf * -1;        // -Int.inf
(string)Int.inf;     // "inf"
intp(Int.inf);       // 0 (it's an object, not an int)

// Byte swapping
int swapped = Int.swap_word(0x1234);    // 0x3412
int swapped2 = Int.swap_long(0x12345678); // 0x78563412

// Bit reflection
int r = Int.reflect(0b1101, 4);  // 0b1011

// Parity (odd popcount = 1)
Int.parity(7);  // 1 (three set bits = odd)
```

#### Gotchas
// - Int.inf is an object, not an int — intp() returns 0
// - parity() rejects negative values
// - Values outside NATIVE_MIN..NATIVE_MAX use slower bignum

## Float Module (Detailed)

```pike
// Constants
Float.MAX        // 1.79769e+308
Float.MIN        // 2.22507e-308
Float.EPSILON    // 2.22045e-16
Float.DIGITS_10  // 15

// NaN check
Float.isnan(0.0/0.0);   // 1
Float.isnan(Math.inf);  // 0

// Precision constant (one of these exists, value 1)
// Float.DOUBLE_PRECISION (typical 64-bit)
```

#### Gotchas
// - Float.MIN is smallest *normalized* positive float, not denormalized
// - Only one precision constant exists at runtime

## master() and the Master Object

```pike
object m = master();           // Get current master

// Key master methods:
m->is_absolute_path("/foo");   // 1
m->explode_path("/a/b/c");    // ({ "", "a", "b", "c" })
m->dirname("/a/b/c");         // "/a/b"
m->basename("/a/b/c");        // "c"
m->normalize_path("/a/../b"); // "/b"
m->getenv("HOME");            // environment variable

// Master replaces the compilation and module resolution system
// replace_master(new_master) — requires root, dangerous

// Important mappings on master:
// programs: filename → compiled program cache
// objects: program → default object
// fc: filename → compiled object cache
```

#### Gotchas
// - master() may return UNDEFINED before master is loaded
// - replace_master() requires security root
// - Mutating master state affects entire runtime

## Geography Module

Note: Module is `Geography`, not `Geographical`.

```pike
// Position (lat, long, alt)
Geography.Position p = Geography.Position(59.33, 18.07);
string lat = p->latitude();          // "59°19.8'N"
string lon = p->longitude();         // "18°4.2'E"
int zone = p->UTM_zone_number();     // 33
string utm = p->UTM(2);             // "33W 672381.87 6580822.93"

// String parsing
Geography.Position p2 = Geography.Position("59 19 30N", "18 4 12E");

// Distance
float d = p->euclidian_distance(p2); // through-earth distance in meters

// Countries
Geography.Countries.se;                    // Country(Sweden)
Geography.Countries.from_domain("us");      // Country(United States)
mapping continents = Geography.Countries.continents();

// GeoIP
Geography.GeoIP.IPv4 geo = Geography.GeoIP.IPv4("GeoIP.csv");
mixed country = geo->from_ip("8.8.8.8");
```

#### Gotchas
// - euclidian_distance() is straight-line through earth, NOT great-circle
// - set_ellipsoid() does NOT convert existing coordinates
// - approx_height() is very crude (10°×10° grid)

## Protocols.DNS — DNS Client/Server

```pike
// Synchronous DNS client (reads /etc/resolv.conf by default)
Protocols.DNS.client dns = Protocols.DNS.client();

// Resolve hostname to IPs
array info = dns->gethostbyname("example.com");
// ({ "example.com", ({ "93.184.216.34" }), ({ }) })

// Reverse lookup
array rev = dns->gethostbyaddr("93.184.216.34");

// MX records
string primary_mx = dns->get_primary_mx("example.com");
array(string) all_mx = dns->get_mx("example.com");

// SRV records
array srv = dns->getsrvbyname("http", "tcp", "example.com");
// ({ ({ priority, weight, port, target }), ... })

// Global shortcuts (no client needed)
Protocols.DNS.gethostbyname("example.com");
Protocols.DNS.get_primary_mx("example.com");
```

### Async DNS
```pike
Protocols.DNS.async_client dns = Protocols.DNS.async_client();

// Callback: (host, ip, @args)
Protocols.DNS.async_host_to_ip("example.com",
  lambda(string host, string|zero ip) {
    write("%s → %s\n", host, ip || "NOT FOUND");
  });
return -1; // keep backend alive
```

### DNS Server
```pike
class MyDNS {
  inherit Protocols.DNS.server;
  mapping reply_query(mapping query, mapping udp_data, function cb) {
    return ([ "rcode": Protocols.DNS.NOERROR,
              "an": ({ ([ "name": query->qd[0]->name,
                          "type": Protocols.DNS.EntryType.T_A,
                          "cl": Protocols.DNS.ResourceClass.C_IN,
                          "ttl": 300, "a": "127.0.0.1" ]) }) ]);
  }
}
```

#### Gotchas
- gethostbyname returns array, not string — access info[1][0] for first IP
- Async callbacks require running backend (return -1 from main)
- Server reply_query can return mapping synchronously or call cb() for async

## Protocols.SMTP — Email Sending

```pike
// Synchronous SMTP client
Protocols.SMTP.Client client = Protocols.SMTP.Client();
client->send_message("from@example.com",
  ({ "to@example.com" }),
  (["subject": "Test", "content-type": "text/plain",
   "Hello from Pike"));
```

Key APIs:
- `Client->send_message(from, recipients, headers_and_body)` — blocking send
- `AsyncClient` — nonblocking variant
- `ClientHelper->parse_addr(addr)` — RFC2822 address parsing
- `ClientHelper->rfc2822date_time()` — date formatting

## Protocols.XMLRPC — XML-RPC

```pike
// Client
Protocols.XMLRPC.Client client = Protocols.XMLRPC.Client("http://example.com/rpc");
mixed result = client["method_name"](arg1, arg2);

// Server
Protocols.XMLRPC.Server server = Protocols.XMLRPC.Server();
server->add_method("add", lambda(int a, int b) { return a + b; });
```

## Protocols.LDAP — LDAP Client

```pike
// Connect and bind
Protocols.LDAP.client ldap = Protocols.LDAP.client("ldap://server:389");
ldap->bind("cn=admin,dc=example,dc=com", "password");

// Search
object res = ldap->search("(objectClass=person)",
  (["basedn": "dc=example,dc=com",
    "scope": 2]));  // 0=base, 1=one, 2=sub

// Iterate results
int n = res->num_entries();
for (int i = 0; i < n; i++) {
  mapping entry = res->fetch(i);
  write("%O\n", entry);
}
ldap->unbind();
```

## System Module

```pike
// User and home
string user = System.get_user();
string home = System.get_home();

// High-resolution timer
System.Timer t = System.Timer();
// ... do work ...
float elapsed = t->get();  // seconds since last get() or create

// File system monitoring (Linux)
#if constant(System.Inotify)
System.Inotify.Instance instance = System.Inotify.Instance();
int wd = instance->add_watch("/tmp",
  System.Inotify.IN_MODIFY | System.Inotify.IN_CREATE,
  lambda(int mask, int cookie, string path) {
    write("Event %s on %s\n",
      System.Inotify.describe_mask(mask), path);
  });
#endif
```

## Local Module

```pike
// Local module namespace for custom Pike modules
// Place modules in $HOME/pike_modules/ or /usr/local/pike_modules/

// Add custom module search path
Local.add_path("/opt/myapp/pike_modules");
Local.remove_path("/opt/myapp/pike_modules");
```

## Sql Module — Database Access

```pike
// Connect (driver auto-detected from URL)
Sql.Sql db = Sql.Sql("pgsql://user:pass@host/dbname");
Sql.Sql db = Sql.Sql("mysql://user:pass@host/dbname");
Sql.Sql db = Sql.Sql("sqlite:///path/to/db.sqlite3");

// Query
array(mapping(string:mixed)) rows = db->query("SELECT * FROM users WHERE id = %d", 42);
foreach (rows; ; mapping row) {
  write("%O\n", row);
}

// Typed query
array(mapping(string:mixed)) rows = db->typed_query("SELECT * FROM users");

// Insert
db->query("INSERT INTO users (name, age) VALUES (%s, %d)", "Alice", 30);

// Get last insert ID
int id = db->master_sql->insert_id();

// Big query (streaming)
Sql.sql_result result = db->big_query("SELECT * FROM large_table");
while (mapping row = result->fetch_row()) {
  // process row
}

// Transaction
db->query("BEGIN");
db->query("UPDATE accounts SET balance = balance - 100 WHERE id = 1");
db->query("UPDATE accounts SET balance = balance + 100 WHERE id = 2");
db->query("COMMIT");
```

Key APIs:
- `Sql.Sql(url)` — connect, driver auto-detected
- `query(sql, args...)` — parameterized query, returns array of mappings
- `typed_query(sql, args...)` — same but with type coercion
- `big_query(sql, args...)` — streaming result via Sql.sql_result
- `quote(value)` — SQL-escape a value
- `list_tables()` — list database tables

#### Gotchas
- Parameter placeholders use Pike format specifiers: %d, %s, %f, %s — NOT ? or $1
- big_query returns Sql.sql_result with fetch_row() returning mapping
- quote() is NOT equivalent to parameterized queries — prefer query() with % args

## HTTP Convenience Functions

```pike
// GET request
Protocols.HTTP.Query q = Protocols.HTTP.get_url("http://example.com");
string body = q->data();
int status = q->status;

// POST with form data
Protocols.HTTP.Query q = Protocols.HTTP.post_url("http://example.com/api",
  (["key": "value"]));

// PUT with body
Protocols.HTTP.Query q = Protocols.HTTP.put_url("http://example.com/api",
  "body data");

// DELETE
Protocols.HTTP.Query q = Protocols.HTTP.delete_url("http://example.com/api/1");

// Get content-type and data
array(string) nice = Protocols.HTTP.get_url_nice("http://example.com");
// ({ content_type, data })

// Just the body
string data = Protocols.HTTP.get_url_data("http://example.com");

// URL encoding helpers
string encoded = Protocols.HTTP.http_encode_query((["q": "hello world"]));
// "q=hello%20world"
string uri = Protocols.HTTP.uri_encode("hello world"); // "hello%20world"
```

Key functions:
- `get_url(url, query_vars, headers, con)` → Query or 0
- `post_url(url, data, headers, con)` → Query or 0
- `put_url(url, body, query_vars, headers, con)` → Query or 0
- `delete_url(url, query_vars, headers, con)` → Query or 0
- `get_url_nice(url)` → ({content_type, data}) or 0 (follows redirects)
- `get_url_data(url)` → string body or 0
- `do_method(method, url, query_vars, headers, con, data)` → Query or 0

#### Gotchas
- NO function named do_sync_method — use do_method instead
- post_url encodes mapping as application/x-www-form-urlencoded; string body sent verbatim
- get_url_nice follows redirects; post_url_nice does NOT
- Returns 0 on I/O failure, not an error

## Useful Builtin Functions

```pike
// limit(min, x, max) — clamp value
limit(0, 5, 10);   // 5
limit(0, -3, 10);  // 0
limit(0, 15, 10);  // 10
// Note: args are (min, x, max), NOT (x, min, max)

// set_weak_flag — weak references for GC
mapping m = ([ "a": some_object ]);
set_weak_flag(m, PIKE_WEAK_VALUES);  // values can be GC'd
set_weak_flag(m, PIKE_WEAK_INDICES); // keys can be GC'd
set_weak_flag(m, 1);                  // shorthand for PIKE_WEAK_BOTH
// Constants: PIKE_WEAK_INDICES=2, PIKE_WEAK_VALUES=4, PIKE_WEAK_BOTH=6

// object_program — get program from object
class Foo {}
Foo f = Foo();
write("%O\n", object_program(f)); // Foo

// next_object — iterate all live objects (requires root security)
// Not recommended for production code
```

#### Gotchas
- limit() args are (min, x, max) — easy to confuse with clamp(x, min, max)
- set_weak_flag returns the same container reference (modifies in place)
- PIKE_WEAK_BOTH=6, NOT 1 (flag=1 is shorthand for both)

## String.Buffer (Deep)

```pike
String.Buffer b = String.Buffer(1024);  // optional initial size
b->add("hello ");
b->add("world");
b->sprintf("%d items", 42);
string result = b->get();     // returns contents AND clears buffer

// get_copy() returns copy WITHOUT clearing
string peek = b->get_copy();  // slower, preserves buffer

// putchar for single characters (supports wide chars)
b->putchar(65);  // 'A'
b->putchar(256); // wide character

// sizeof returns current length
write("%d bytes\n", sizeof(b));

// Cast to string
string s = (string)b;  // equivalent to get() or get_copy() depending on ref count

// += appends in-place, + returns new buffer
b += "more";      // modifies b
String.Buffer c = b + "extra";  // new buffer, b unchanged

// Fast pattern: cache add method
function add = b->add;
for (int i = 0; i < 10000; i++) add("line\n");
string result = b->get();
```

#### Gotchas
- get() CLEARS the buffer — use get_copy() to preserve
- += modifies in-place; + creates new buffer
- Buffer starts at 256 bytes by default (minimum 512 internally)
- Wide characters (putchar > 255) make entire buffer wide-string

## ADT.Struct — Binary Data Parsing

```pike
// Define a binary structure
class PacketHeader {
  inherit ADT.Struct;
  Item magic    = Byte();      // 1 byte unsigned
  Item version  = Byte();      // 1 byte unsigned
  Item length   = Word();      // 2 bytes big-endian unsigned
  Item flags    = SWord();     // 2 bytes big-endian signed
  Item checksum = Long();      // 4 bytes big-endian unsigned
  // Little-endian: Drow(), Gnol()
  // Aliases: uint8(), uint16(), int16(), uint32(), int32(), int64(), uint64()
}

// Decode from binary string
object hdr = PacketHeader("\xff\x01\x00\x10\xff\xff\x00\x00\x00\x01");
write("magic: %d version: %d length: %d\n", hdr->magic, hdr->version, hdr->length);

// Encode back to binary
string encoded = (string)hdr;  // or hdr->encode()

// Dynamic-length fields (reference another field for size)
class Msg {
  inherit ADT.Struct;
  Item strlen = Byte();
  Item data = Chars(strlen);  // size determined by strlen field
}
object m = Msg("\x05hello");
write("%s\n", m->data);  // "hello"

// Decode from file
Stdio.File f = Stdio.File("data.bin", "r");
object pkt = PacketHeader(f);
```

Item types: Byte(8-bit), Word(16-bit BE), Drow(16-bit LE), Long(32-bit BE), Gnol(32-bit LE), SByte/SWord/SLong (signed), Chars(n) (fixed-length string), Varchars() (null-terminated), int8..uint64 aliases.

#### Gotchas
- Default encoding is big-endian (network order); use Drow/Gnol for little-endian
- Chars() rejects wide strings — 8-bit only
- sizeof(struct) returns total encoded byte count
- Dynamic size via Chars(field_reference) — size determined at decode time

## ADT.CritBit — Trie Data Structures

```pike
// String trie with prefix search
ADT.CritBit.StringTree tree = ADT.CritBit.StringTree();
tree["apple"] = 1;
tree["apricot"] = 2;
tree["banana"] = 3;

// Ordered iteration (always sorted)
foreach (tree; string key; mixed val)
  write("%s: %O\n", key, val);

// Prefix lookup
object sub = tree->get_subtree("ap");
write("%O\n", indices(sub));  // ({ "apple", "apricot" })

// Navigation
tree->first();          // "apple"
tree->last();           // "banana"
tree->next("apple");    // "apricot"
tree->previous("banana"); // "apricot"
tree->nth(1);           // ({ "apricot", 2 })

// Range query
object range = tree["apple".."apricot"];

// Set operations
object other = ADT.CritBit.StringTree();
other["apricot"] = 10;
object union_tree = tree + other;  // union
object diff_tree = tree - other;   // difference

// Factory function for different key types
object t1 = ADT.CritBit.Tree();            // StringTree
object t2 = ADT.CritBit.Tree("int");       // IntTree
object t3 = ADT.CritBit.Tree("float");     // FloatTree
object t4 = ADT.CritBit.Tree("ipv4");      // IPv4Tree

// IPv4 routing table
ADT.CritBit.IPv4Tree routes = ADT.CritBit.IPv4Tree();
routes["10.0.0.0/8"] = "private";
routes["192.168.0.0/16"] = "LAN";
routes["0.0.0.0/0"] = "default";
object match = routes->get_subtree("192.168.1.5");
```

#### Gotchas
- Tree types: StringTree, IntTree, FloatTree, IPv4Tree, DateTree
- IPv4Tree keys are dotted-decimal with optional CIDR prefix
- + operator for union: right tree wins on key collision
- Range uses `[..]` operator: tree["a".."z"]
- Pre-populate from mapping: ADT.CritBit.Tree((["a":1, "b":2]))

## Protocols.WebSocket

```pike
// Client
Protocols.WebSocket.Connection ws = Protocols.WebSocket.Connection();
ws->onopen = lambda(mixed id) { ws->send_text("hello"); };
ws->onmessage = lambda(Protocols.WebSocket.Frame f, mixed id) {
  write("Got: %s\n", f->text);
};
ws->onclose = lambda(int status, mixed id) { write("Closed: %d\n", status); };
ws->connect(Standards.URI("ws://example.com/ws"));
return -1; // keep backend alive

// Server
Protocols.WebSocket.Port(
  lambda(object req) { return 0; },  // HTTP fallback
  lambda(object req) {
    // WebSocket upgrade; return Connection
    Protocols.WebSocket.Connection conn = req->websocket_accept("chat");
    conn->onmessage = lambda(Protocols.WebSocket.Frame f, mixed id) {
      conn->send_text("echo: " + f->text);
    };
    return conn;
  },
  8080);
return -1;
```

Frame types: FRAME_TEXT, FRAME_BINARY, FRAME_CLOSE, FRAME_PING, FRAME_PONG
Close status: CLOSE_NORMAL(1000), CLOSE_GONE_AWAY(1001), CLOSE_ERROR(1002)

## Protocols.IRC

```pike
Protocols.IRC.Client client = Protocols.IRC.Client("irc.example.com",
  ([ "nick": "PikeBot",
     "port": 6667 ]));

// Send message
client->send_message("#channel", "Hello from Pike!");

// Command proxy
client->cmd->join("#channel");
client->cmd->privmsg("#channel", "Hello");
client->cmd->async_privmsg("#channel", "Async msg");
```

## Calendar Module

```pike
// Current date/time
Calendar.Day today = Calendar.Day();          // today
Calendar.Second now = Calendar.Second();       // current second
Calendar.Day christmas = Calendar.Day(2024, 12, 25);

// Formatting
string iso = Calendar.Day()->format_ymd();   // "2026-04-19"
string time = now->format_tod();              // "14:30:05"

// Arithmetic
Calendar.Day tomorrow = today + 1;             // add day
Calendar.Day next_week = today + 7;  // add 7 days
int days_until = christmas - today;             // TimeRange subtraction

// Parsing
Calendar.Day d = Calendar.parse("%Y-%M-%D", "2024-04-18");

// Time ranges
Calendar.TimeRange range = Calendar.Day(2024,1,1) .. Calendar.Day(2024,12,31);

// Timezone
object now_utc = Calendar.now()->set_timezone("UTC");  // returns TimeofDay, not Day
```

#### Gotchas
- Calendar.Day is a TimeRange (full day from 00:00 to 24:00)
- Subtraction of TimeRanges returns number of calendar units (not seconds)
- Calendar.parse format is Pike-specific, NOT strftime

## Threading (Deep)

```pike
// Create and join thread
object t = thread_create(lambda() { return 42; });
int result = t->wait();  // blocks, returns exit status

// Mutex
Thread.Mutex mx = Thread.Mutex();
object key = mx->lock();   // blocks until acquired
// ... critical section ...
key = 0;  // release

object key2 = mx->trylock();  // non-blocking: returns 0 if locked

// Condition variable
Thread.Condition cond = Thread.Condition();
// Waiter:
object key = mx->lock();
while (!ready) cond->wait(key);  // atomically releases mutex
// Signaler:
cond->signal();   // wake one waiter
cond->broadcast(); // wake all

// Thread-local storage
Thread.Local mydata = Thread.Local();
mydata->set(42);
write("%O\n", mydata->get());  // 42

// Bounded FIFO queue
Thread.Fifo fifo = Thread.Fifo(128);  // capacity 128
fifo->write(data);   // blocks when full
mixed val = fifo->read();  // blocks when empty
mixed val2 = fifo->try_read();  // UNDEFINED if empty

// Unbounded queue
Thread.Queue queue = Thread.Queue();
queue->write(data);  // never blocks

// Thread pool (Farm)
Thread.Farm farm = Thread.Farm();  // default max 20 workers
farm->set_max_num_threads(4);  // reduce to 4
Thread.Farm.Result r1 = farm->run(lambda() { return 42; });
Thread.Farm.Result r2 = farm->run(lambda() { return 43; });
mixed val1 = r1->get();  // blocks until done, returns 42
mixed val2 = r2->get();  // blocks until done, returns 43
```

Thread states: THREAD_NOT_STARTED(-1), THREAD_RUNNING(0), THREAD_EXITED(1), THREAD_ABORTED(2)

#### Gotchas
- MutexKey released on destruction/scope exit, NOT by method call
- thread_create is a global function (not Thread.thread_create)
- Condition.wait() atomically releases mutex and blocks
- Thread.Fifo default capacity is 128
- Thread.Farm workers are reused across tasks

## Stdio.Port — TCP Server

```pike
// Simple TCP server
Stdio.Port port = Stdio.Port(8080, accept_callback);

void accept_callback(Stdio.Port port) {
  Stdio.File conn = port->accept();
  conn->set_nonblocking(read_cb, write_cb, close_cb);
}

void read_cb(mixed id, string data) {
  write("Received: %O\n", data);
}

void write_cb(mixed id) { }

void close_cb(mixed id) { write("Connection closed\n"); }

return -1; // keep backend alive
```

## Stdio.UDP — UDP Networking

```pike
// UDP receiver
Stdio.UDP udp = Stdio.UDP();
udp->bind(9090);
udp->set_nonblocking(lambda(string data, string from_ip, int from_port) {
  write("From %s:%d: %O\n", from_ip, from_port, data);
  udp->send(from_ip, from_port, "ACK");
});
return -1;

// UDP sender
Stdio.UDP udp = Stdio.UDP();
udp->send("127.0.0.1", 9090, "Hello");

// Blocking read
mapping response = udp->read();  // (["data": ..., "ip": ..., "port": ...])
```

## Stdio.Buffer — Buffered I/O

```pike
Stdio.Buffer buf = Stdio.Buffer("initial data");

// Add data
buf->add("more data");

// Read functions
string line = buf->match("%[^\n]");  // sscanf-style format, read until newline
int(0..255) byte = buf->read_int8();
int word = buf->read_int16();   // big-endian
int dword = buf->read_int32();
string chunk = buf->read(10);  // read exactly 10 bytes
string rest = buf->read();      // read all remaining

// Write functions
buf->add_int8(42);
buf->add_int16(1000);
buf->add_int32(100000);
buf->sprintf("%s %d", "hello", 42);

// Locking (sub-buffers)
Stdio.Buffer sub = buf->lock(); // prevents external reads
// sub released automatically on scope exit / destruction
```

#### Gotchas
- Stdio.Buffer is both readable and writable (unlike String.Buffer which is write-only)
- read_int* functions consume bytes from the buffer
- match(fmt) uses sscanf-style format strings, NOT literal delimiters. Use "%[^\n]" to read until newline.
- Lock prevents other code from consuming buffer contents

## Search Functions

```pike
// search(haystack, needle, start)
int pos = search("hello world", "world");    // 6
int pos2 = search(({1,2,3,4,5}), 3);        // 2
int pos3 = search("hello", "xyz");           // -1

// has_index
int has = has_index((["a":1, "b":2]), "a");   // 1
int has2 = has_index(({10,20,30}), 1);         // 1 (index 1 exists)

// has_value
int has3 = has_value(({10,20,30}), 20);       // 1
int has4 = has_value((["a":1]), 1);            // 1 (searches values)
```

#### Gotchas
- search returns -1 if not found (NOT 0 or UNDEFINED)
- has_index on arrays checks if index is in range

## Protocols.Bittorrent

```pike
// Bencode encode/decode
import Protocols.Bittorrent.Bencoding;

string encoded = encode((["key": "value", "num": 42]));
// d3:key5:value3:numi42ee

mapping m = decode(encoded);
// ([ "key":"value", "num":42 ])

// Bitfield utilities
array(int(0..1)) bits = string2bits(bitfield_string);
array(int) set_indices = string2arr(bitfield_string);
string bf = bits2string(({1,0,1,0}));
```

Key classes: Torrent (full client), Generator (.torrent creator), Tracker (HTTP/UDP tracker server), PeerID (identify client from peer ID).

## Protocols.NNTP

```pike
// NNTP client
Protocols.NNTP.client nntp = Protocols.NNTP.client("news.example.com");
array(string) groups = nntp->list_groups();
nntp->go_to_group("comp.lang.pike");
mapping header = nntp->head(1);   // header of article 1
string body = nntp->body(1);       // body of article 1
string full = nntp->article(1);    // full article
```

## Protocols.TELNET

```pike
// Telnet protocol handler
Protocols.TELNET.LineMode telnet = Protocols.TELNET.LineMode(
  Stdio.File fd, read_cb, write_cb, close_cb, callbacks, id);
// Override read_callback for line input
// Note: Class is LineMode, not LineBased
```

## Audio Module

```pike
// MP3 parsing (requires Audio module)
#if constant(Audio.Format.MP3)
Audio.Format.MP3 mp3 = Audio.Format.MP3();
mp3->read_file("song.mp3");
mapping frame = mp3->get_frame();
// (["bitrate": 128000, "channels": "stereo", "sampling": 44100, ...])
#endif
```

Note: Audio.Codec requires _Ffmpeg — may not be available in all builds.

## Pike Module — Runtime Introspection

```pike
// Runtime information
mapping info = Pike.get_runtime_info();
// ([ "pike_version": ..., "bytecode_method": ..., etc ])

// GC tuning
Pike.gc_parameters();  // returns current GC settings
Pike.gc_parameters((["enabled": 1]));  // modify (note: mapping literal)

// Memory analysis
int bytes = Pike.count_memory(0, object_or_array);

// Weak reference flags
int flag = Pike.WEAK_INDICES;  // 2
int flag2 = Pike.WEAK_VALUES;  // 4
int flag3 = Pike.WEAK_BOTH;   // 6
```

## predef Functions (Comprehensive)

Key predef functions beyond what's already documented:

```pike
// Type checking
intp(x), floatp(x), stringp(x), arrayp(x), mappingp(x),
multisetp(x), objectp(x), programp(x), functionp(x)

// Type conversion
(int)x, (float)x, (string)x, (array)x, (mapping)x
zero_type(x)  // 0=value exists, 1=UNDEFINED, 2=destructed

// Collections
aggregate(val, ...)    // create array: ({val, ...})
allocate(n, val|void)  // create array of n elements
indices(x)             // get keys/indices
values(x)              // get values
sizeof(x)              // size of collection
copy_value(x)          // deep copy

// Searching
search(haystack, needle, start)
has_index(collection, index)
has_value(collection, value)

// Sorting
sort(array)            // in-place sort (returns sorted array)
reverse(array_or_string)

// Math
abs(x), sqrt(x), pow(x, y), exp(x), log(x)
sin(x), cos(x), tan(x), asin(x), acos(x), atan(x), atan2(y, x)
floor(x), ceil(x), round(x)
limit(min, x, max)     // clamp
random(n)              // random int [0, n)
Crypto.Random.random_string(n)  // cryptographically secure random

// String
strlen(s)             // same as sizeof(s)
stringp(s)
string_to_utf8(s), utf8_to_string(s)
upper_case(s), lower_case(s), capitalize(s)

// I/O
write(fmt, args...), werror(fmt, args...)

// Control flow
call_out(func, delay, args...)
remove_call_out(id_or_func)

// Object/Program
object_program(o)     // get program from object
destruct(o)            // destroy object
clone(program, args)  // create object from program

// Compilation
compile(source), compile_string(source), compile_file(path)

// Error handling
catch { ... }, throw(value)
master()->describe_error(err)
master()->describe_backtrace(bt)

// Misc
gauge { expr }         // CPU time as float seconds
gethrtime()            // nanoseconds since epoch
gethrvtime()           // CPU time in nanoseconds
time()                 // unix timestamp
ctime(timestamp)       // formatted time string
```

#### Gotchas
- sort() modifies the array in-place AND returns it
- random(n) returns int in [0, n), NOT [0, n]
- limit() args are (min, x, max), NOT (x, min, max)
- zero_type returns 1 for UNDEFINED (missing mapping key), 0 for actual 0
- search returns -1 on not found

## Filesystem.Traversion — Directory Traversal

```pike
// Recursive directory traversal (iterator)
foreach (Filesystem.Traversion("."); string dir; string file) {
  string path = combine_path(dir, file);
  Stdio.Stat st = Filesystem.Traversion(".")->stat();
  // dir = current directory path, file = current filename
}

// Options: skip symlinks, ignore errors, sort entries
foreach (Filesystem.Traversion("/path", 1, 1, Array.sort);
         string dir; string file) {
  write("%s/%s\n", dir, file);
}
```

## HTTP Cookies (via Session)

```pike
// Cookies are managed by Protocols.HTTP.Session
Protocols.HTTP.Session s = Protocols.HTTP.Session();

// Set from Set-Cookie header
s->set_http_cookie("session=abc123; path=/", Standards.URI("http://example.com"));

// Get cookies for a URL
array(string) cookies = s->get_cookies(Standards.URI("http://example.com/page"));
// ({ "session=abc123" })

// Persist/restore cookies
string saved = s->encode_cookies();
s->decode_cookies(saved);
```

Note: No standalone Protocols.HTTP.Cookie module exists. Cookie handling is internal to Session.

## Standards.UUID

```pike
// Generate UUIDs
string uuid_v1 = Standards.UUID.make_version1();   // time-based
string uuid_v3 = Standards.UUID.make_version3("namespace", "name");  // MD5
string uuid_v4 = Standards.UUID.make_version4();   // random
string uuid_v5 = Standards.UUID.make_version5("namespace", "name");  // SHA1

// Compare
int cmp = Standards.UUID.compare(uuid1, uuid2);

// DNS namespace constant
string ns = Standards.UUID.NAMESPACE_DNS;
```

## Standards.PKCS

Available submodules (if compiled with Nettle):
- `Standards.PKCS.Signature` — PKCS#1 signature operations
- `Standards.PKCS.Certificate` — X.509 certificate handling
- `Standards.PKCS.CSR` — Certificate Signing Requests
- `Standards.PKCS.RSA` — RSA PKCS#1 operations
- `Standards.PKCS.DSA` — DSA operations
- `Standards.PKCS.MessageDigest` — message digest algorithms

```pike
// Certificate handling
string pem = Stdio.read_file("cert.pem");
object cert = Standards.PKCS.Certificate.get_certificate(pem);
```

## Calendar Module (Comprehensive)

```pike
// Current time
Calendar.Day today = Calendar.Day();
Calendar.Second now = Calendar.Second();

// Creation
Calendar.Day christmas = Calendar.Day(2024, 12, 25);
Calendar.Month jan = Calendar.Month(2024, 1);
Calendar.Week wk1 = Calendar.Week(2024, 1);
Calendar.Year yr = Calendar.Year(2024);

// Formatting
string ymd = today->format_ymd();         // "2026-04-19"
string iso = today->format_iso_ymd();     // "2026-04-19"
string tod = now->format_tod();           // "14:30:05"
string http_date = now->format_http();    // "Sat, 19 Apr 2026 14:30:05 GMT"
string smtp_date = now->format_smtp();    // RFC2822 format

// Arithmetic
Calendar.Day tomorrow = today + 1;
Calendar.Day next_week = today + 7;
Calendar.Month next_month = Calendar.Month() + 1;
int days = christmas - today;  // number of days between

// Sub-units
array(Calendar.Hour) hrs = today->hours();
Calendar.Hour h3 = today->hour(3);  // 03:00
int n_hours = today->number_of_hours();

// Timezone
object utc = now->set_timezone("UTC");
object local = now->set_timezone("Europe/Stockholm");
int offset = utc->utc_offset();
string tz = utc->tzname();  // "UTC"

// Comparison
(today < christmas);  // 1
(today == Calendar.Day());  // 1

// Date components
int y = today->year_no();
int m = today->month_no();
int d = today->month_day();
int wd = today->week_day();  // 1=Monday, 7=Sunday
string name = today->month_name();  // "April"
string wname = today->week_day_name();  // "Saturday"
```

#### Gotchas
- Calendar.Day is a TimeRange (full 24-hour period), not a point in time
- Subtraction returns number of calendar units, not necessarily seconds
- Calendar.Second for precise timestamps, Calendar.Day for dates
- set_timezone() returns NEW object — original is not modified
- week_day(): 1=Monday, 7=Sunday (ISO standard)
- format_http() uses GMT, not local timezone

## Standards.URI — URI Parsing and Manipulation

```pike
// Parse a URI
object u = Standards.URI("https://user:pass@example.com:8443/path?q=1#frag");
u->scheme;     // "https"
u->host;       // "example.com"
u->port;       // 8443
u->path;       // "/path"
u->query;      // "q=1"
u->fragment;   // "frag"
u->user;       // "user"
u->password;   // "pass"

// Relative URI resolution
object base = Standards.URI("http://example.com/dir/page.html");
object rel = Standards.URI("../other.html", base);
// (string)rel == "http://example.com/other.html"

// Query variable manipulation
object u = Standards.URI("http://example.com/path?a=1&b=2");
u->get_query_variables();     // ([ "a":"1", "b":"2" ])
u->add_query_variable("c", "3");
u->set_query_variables((["x":"10"]));

// Modify components (auto-updates related fields)
u->host = "other.com";  // authority auto-updates
(string)u;              // reassembles canonical form

// Cast to string returns canonical URI
(string)Standards.URI("HTTP://EXAMPLE.COM/");  // "http://example.com/"

// Static helper
Standards.URI.combine_uri_path("/dir/file", "../other");  // "/other"
```

#### Key Methods
- `get_query_variables()` — parse query string into mapping
- `set_query_variables(mapping)` — set query from mapping
- `add_query_variable(string, string)` — append query param
- `get_http_path_query()` — RFC 1738 encoded path+query
- `http_encode(string)` — percent-encode unsafe chars
- `reparse_uri(base)` — re-resolve against new base

#### Gotchas
- Port auto-defaults from scheme (443 for https, 80 for http)
- Setting host/port auto-updates authority; setting authority auto-parses host/port
- Case-insensitive equality: Standards.URI normalizes scheme/host
- base_uri is second arg to create() for relative resolution

## Web Module — JWT, OAuth, API Clients

### JWT/JWS/JWK
```pike
// Create a signing key
Crypto.DSA.State key = Crypto.DSA(2048);
// or: Crypto.RSA.State key = Crypto.RSA(2048);
// or HMAC: Crypto.MAC.HMAC(Crypto.SHA256)("secret")

// Encode JWT
string jwt = Web.encode_jwt(key, ([ "alg":"HS256", "typ":"JWT" ]));
// JWT header auto-derives alg from key type

// Decode JWT (provides same key for verification)
mapping claims = Web.decode_jwt(key, jwt);

// JWS (JSON Web Signature — lower level)
string jws = Web.encode_jws(key, payload_string, "optional-cty");
array decoded = Web.decode_jws(key, jws);

// JWK (JSON Web Key)
string jwk_json = Web.encode_jwk(key);
// Returns JSON with kty, crv, x, y, etc

object decoded_key = Web.decode_jwk(jwk_json);
// Returns a Sign State object

// JWK Set
string jwk_set = Web.encode_jwk_set(({ key1, key2 }));
array keys = Web.decode_jwk_set(jwk_set_json);
```

### OAuth2 (Web.Auth.OAuth2)
```pike
// OAuth2 authorization flow
Web.Auth.OAuth2.Client client = Web.Auth.OAuth2.Client("client_id", "client_secret");
client->set_auth_uri("https://accounts.google.com/o/oauth2/auth");
client->set_token_uri("https://accounts.google.com/o/oauth2/token");
string auth_url = client->get_auth_uri((["scope":"email"]));
// Redirect user to auth_url, get code back
client->request_access_token(code);
```

### Web.Auth Providers
Built-in auth clients for: Facebook, Github, Google, Instagram, Linkedin, Twitter

### Web.Api Providers
Built-in API clients for: Facebook, Github, Google, Instagram, Linkedin, Twitter

### Web.RSS
RSS feed parsing and generation

### Web.Crawler
Web crawling/spidering framework

### Web.OWL / Web.RDF / Web.RDFS
Semantic web (OWL ontologies, RDF triples, RDFS)

## _Roxen — Low-Level HTTP Utilities

```pike
// HTML entity encoding
_Roxen.html_encode_string("<b>hi & \"bye\"</b>");
// "&lt;b&gt;hi &amp; &quot;bye&quot;&lt;/b&gt;"

// URL/HTTP percent-decoding
_Roxen.http_decode_string("hello%20world");  // "hello world"
_Roxen.http_decode_string("a%2Bb");            // "a+b"

// HTTP header parsing
_Roxen.HeaderParser hp = _Roxen.HeaderParser();
mapping headers = hp->feed("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n");
// Returns mapping with request line + headers

// Construct HTTP response headers
string hdr = _Roxen.make_http_headers((["Content-Type":"text/html"]));
```

#### Gotchas
- _Roxen is a C builtin module (not in lib/modules)
- HeaderParser is used internally by Protocols.HTTP
- html_encode_string converts <, >, &, ", ' to HTML entities
- http_decode_string decodes %XX sequences

## Protocols.SNMP — SNMP Protocol

```pike
// SNMP protocol codec (ASN.1 based)
// Constants for request types
Protocols.SNMP.REQUEST_GET;        // 0
Protocols.SNMP.REQUEST_GETNEXT;    // 1
Protocols.SNMP.REQUEST_SET;        // 3
Protocols.SNMP.REQUEST_TRAP;       // 7

// Error codes
Protocols.SNMP.ERROR_NOERROR;     // 0
Protocols.SNMP.ERROR_TOOBIG;      // 1
Protocols.SNMP.ERROR_NOSUCHNAME;  // 2
Protocols.SNMP.ERROR_BADVALUE;    // 3
Protocols.SNMP.ERROR_READONLY;    // 4
Protocols.SNMP.ERROR_GENERROR;    // 5

// SNMP agent (server)
Protocols.SNMP.agent snmp_agent = Protocols.SNMP.agent();
// Agent provides SNMP get/set handling

// SNMP protocol (low-level)
Protocols.SNMP.protocol pdu = Protocols.SNMP.protocol();
// PDU encoding/decoding via ASN.1
```

#### Gotchas
- SNMP v1/v2c only (no v3 support)
- Uses Protocols.HTTP.Promise-style async patterns
- protocol.pike handles ASN.1 BER encoding/decoding
- agent.pike provides a simple SNMP agent framework

## Tools.Monger — Pike Package Manager

```pike
// MongerUser — install/search packages
Tools.Monger.MongerUser monger = Tools.Monger.MongerUser();

// Search for a module
monger->search("mysql");

// Install latest version
monger->install("Mysql");

// Package constants
Tools.Monger.MongerUser.SOURCE_PACKAGE;      // "source package"
Tools.Monger.MongerUser.SOURCE_CONTROL;      // "source control"
Tools.Monger.MongerUser.PURE_PIKE_PMAR;      // "pure pike"
Tools.Monger.MongerUser.PLATFORM_SPECIFIC_PMAR;  // "platform specific"

// Also available: Tools.Monger.MongerDeveloper
// For module authors: register, upload, manage releases
```

#### Gotchas
- MongerUser is for consumers; MongerDeveloper for module authors
- Uses XML-RPC to communicate with module repository
- Install location depends on PIKE_MODULE_PATH
- CLI tool: `pike -x monger install ModuleName`

## SQLite — Embedded Database

```pike
// Open in-memory database
object db = SQLite.SQLite("file::memory:?cache=shared");

// Open file database
object db = SQLite.SQLite("/path/to/db.sqlite");

// Execute DDL
 db->query("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)");

// Insert
db->query("INSERT INTO users VALUES (1, 'Alice', 'alice@example.com')");

// Query — returns array of mappings
array(mapping(string:string)) rows = db->query("SELECT * FROM users");
// ({ (["id":"1", "name":"Alice", "email":"alice@example.com"]) })

// Note: ALL values are returned as strings, including integers
// Cast as needed: (int)row["id"]

// Parameterized queries (if supported by API version)
// Use sprintf with caution — prefer literal values
```

#### Gotchas
- All column values returned as STRING, even integers — cast with `(int)` as needed
- SQLite.SQLite is a C module (requires libsqlite3)
- In-memory databases use `file::memory:` URI
- No built-in parameterized query API — sanitize inputs manually

## Image Module — Image Processing

```pike
// Create blank image
object img = Image.Image(200, 100);  // width x height
img->setcolor(255, 128, 0);         // fill with orange

// Dimensions
img->xsize();  // 200
img->ysize();  // 100

// Get/set pixels
img->setpixel(10, 20, 255, 0, 0);  // set red pixel
array(int) rgb = img->getpixel(10, 20);  // ({255, 0, 0})

// Drawing primitives
img->line(0, 0, 199, 99, 255, 255, 255);  // white diagonal line
img->box(10, 10, 50, 50, 0, 0, 255);      // blue rectangle
img->circle(100, 50, 30, 255, 0, 0);     // red circle

// Transformations
object resized = img->scale(100, 50);    // resize
object rotated = img->rotate(45);        // rotate 45 degrees
object grey = img->grey();               // convert to grayscale
object mirrored = img->mirrorx();        // horizontal flip
object inverted = img->invert();          // invert colors
object cropped = img->autocrop();         // remove borders
object blurred = img->blur(3);            // gaussian blur

// Compositing
object bg = Image.Image(200, 100)->setcolor(0, 0, 0);
bg->paste(img);                           // paste at 0,0
bg->paste_alpha(img, 128, 10, 10);       // paste with 50% alpha at 10,10

// Color space conversion
array(float) hsv = img->rgb_to_hsv();    // not on Image object

// Encoding/decoding
string pnm = Image.PNM.encode(img);     // PNM format
object decoded = Image.PNM.decode(pnm); // decode

// Auto-detect format
object any = Image.ANY.decode(image_data);  // auto-detect
mapping header = Image.ANY.decode_header(data);  // just header

// Named colors
object red = Image.Color("red");
object blue = Image.Color("#0000FF");

// Operators: +, -, *, /, |, & work on images (per-pixel)
object combined = img1 + img2;   // additive blend
object mask = img * 0.5;         // darken by 50%
```

#### Supported Formats
PNM, BMP, X, TGA, PCX, ILBM, RAS, TIM, AVS, HRZ, DSI, NEO, PVR, WBF, WBMP, ANY (auto-detect)

#### Key Image.Image Methods (101 total)
create, clone, copy, setcolor, getpixel, setpixel, line, box, circle,
polyfill, polygone, scale, rotate, rotate_ccw, rotate_cw, rotate_expand,
mirrorx, mirrory, grey, invert, blur, autocrop, find_autocrop,
paste, paste_alpha, paste_alpha_color, paste_mask,
threshold, gamma, noise, turbulence, gradients, random, randomgrey,
apply_curve, apply_matrix, apply_max, match, match_norm,
sum, sumf, min, max, average, orient, outline, outline_mask,
phaseh, phasev, phasehv, phasevh, turbx_expand, skewx, skewy,
rgb_to_hsv, hsv_to_rgb, rgb_to_yuv, yuv_to_rgb,
find_min, find_max, distancesq, select_from,
change_color, modify_by_intensity, make_ascii,
read_lsb_rgb, write_lsb_rgb, read_lsb_grey, write_lsb_grey,
test, tobitmap, xsize, ysize

#### Gotchas
- `Image.Image(w, h)` creates blank black image — use `setcolor()` to fill
- `grey()` not `gray()` (British spelling)
- All pixel values are 0-255 integers
- Image operators (+, -, *, etc) work per-pixel between images
- `scale()` takes (width, height) or a single factor
- `rotate()` can lose edges — use `rotate_expand()` to expand canvas

## SSL Module — TLS/SSL (Deep)

```pike
// Create SSL context
SSL.Context ctx = SSL.Context();

// Load certificate and key
string cert = Stdio.read_file("server.crt");
string key = Stdio.read_file("server.key");
ctx->add_cert(Standards.PKCS.RSA.parse_private_key(key), ({ cert }));

// Or generate self-signed
Crypto.RSA.State rsa = Crypto.RSA(2048);
string self_signed = Standards.X509.make_selfsigned_certificate(rsa, 365*24*3600);
ctx->add_cert(rsa, ({ self_signed }));

// Configure TLS version
ctx->min_version = SSL.Constants.PROTOCOL_TLS_1_2;
ctx->max_version = SSL.Constants.PROTOCOL_TLS_1_3;

// Filter weak cipher suites
ctx->filter_weak_suites();

// Session cache settings
ctx->use_cache = 1;
ctx->max_sessions = 1000;
ctx->session_lifetime = 3600;

// Certificate verification (client side)
ctx->verify_certificates = 1;
ctx->require_trust = 1;
ctx->set_trusted_issuers(({ /* CA certs */ }));

// ALPN (Application-Layer Protocol Negotiation)
ctx->advertised_protocols = ({"h2", "http/1.1"});

// Get configured suites
array suites = ctx->get_suites();
```

### SSL.File — TLS Socket
```pike
// Server: wrap accepted connection
Stdio.File raw_conn = port->accept();
SSL.File ssl_conn = SSL.File(raw_conn, ctx);

// Client: connect
SSL.File ssl_conn = SSL.File();
ssl_conn->connect("example.com", 443, ctx);
ssl_conn->write("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n");
string response = ssl_conn->read();
```

#### SSL.Context Key Methods
- `add_cert(State key, array(string) certs, array(string)|void extra)` — add certificate
- `filter_weak_suites()` — remove known-weak cipher suites
- `configure_suite_b(int(128|192) bits)` — Suite-B compliance
- `get_suites()` — available cipher suites
- `get_certificates()` — loaded certificates
- `find_cert_domain(string)` — find cert matching domain
- `verify_certificates` — enable client cert verification
- `advertised_protocols` — ALPN protocols
- `min_version`/`max_version` — TLS version bounds

#### Gotchas
- `add_cert()` takes (key, certs_array) — certs must be in an array
- SSL.File wraps Stdio.File — call `connect()` for client, or pass accepted fd for server
- `filter_weak_suites()` should always be called for production
- Certificate verification requires trusted issuers to be set

## Protocols.Ident — RFC 1413 Ident Protocol

```pike
// Synchronous ident lookup
array(string) result = Protocols.Ident.lookup(socket_fd);
// Returns: ({"USERID", "UNIX", "username"}) on success
// Returns: ({"ERROR", "NO-USER"}) on failure

// Async ident lookup
Protocols.Ident.AsyncLookup(socket_fd, lambda(array(string) result, mixed ... extra) {
  write("Ident result: %O\n", result);
}, "optional", "args");
// 60 second timeout
```

#### Gotchas
- Requires a connected socket (extracts ports from query_address)
- Most modern systems don't run identd — expect errors in production
- Response is array of colon-split fields

## Protocols.IPv6 — Address Utilities

```pike
// Parse any IPv6 format
array(int) addr = Protocols.IPv6.parse_addr("::1");
// ({0,0,0,0,0,0,0,1})

Protocols.IPv6.parse_addr("2001:db8::1");
// ({0x2001, 0x0db8, 0, 0, 0, 0, 0, 1})

// Format to shortest canonical form
Protocols.IPv6.format_addr_short(({0,0,0,0,0,0,0,1}));
// "::1"

// Normalize
Protocols.IPv6.normalize_addr_short("2001:0db8:0000::0001");
// "2001:db8::1"

Protocols.IPv6.normalize_addr_basic("::1");
// "0:0:0:0:0:0:0:1"

// Returns 0 for invalid addresses
Protocols.IPv6.parse_addr("invalid");  // 0
```

#### Key Functions
- `parse_addr(string)` → 8-element array of 16-bit ints, or 0
- `format_addr_short(array)` → shortest canonical form
- `normalize_addr_basic(string)` → full x:x:x:x:x:x:x:x form
- `normalize_addr_short(string)` → canonical shortest form

## Protocols.LMTP — Local Mail Transfer Protocol

```pike
// LMTP server (extends Protocols.SMTP)
Protocols.LMTP.Server server = Protocols.LMTP.Server(
  ({"example.com"}),        // domains
  24,                       // port (LMTP default: 24)
  "127.0.0.1",              // bind address
  lambda(string sender) { return 250; },      // cb_mailfrom
  lambda(string rcpt) { return 250; },        // cb_rcptto
  lambda(MIME.Message msg, string sender, string rcpt, string|void raw) {
    return 250;  // or ({code, "error string"})
  }
);
```

#### Gotchas
- LMTP calls cb_data ONCE PER RECIPIENT (not once per message like SMTP)
- Extends Protocols.SMTP — most SMTP features apply
- Default port is 24 (not 25 like SMTP)
- Used for local mail delivery (Postfix → LMTP → Dovecot pattern)

## Parser Module (Deep)

### Parser.C / Parser.Pike — Source Code Tokenization
```pike
// Both share the same API. Takes array of strings (lines).
array(Parser.C.Token) tokens = Parser.C.tokenize(({"int main() { return 0; }"}));
// ({ Parser.C.Token("int main() { return 0; }",0,1) }})

// Split into individual tokens
array(string) parts = Parser.C.split("int main() { return 0; }");
// ({"int", " ", "main", "(", ")", " ", "{", " ", "return", " ", "0", ";", " ", "}", "\n"})

// Same for Pike code
array(string) pike_tokens = Parser.Pike.split("int x = 42;");
// ({"int", " ", "x", " ", "=", " ", "42", ";", "\n"})

// Group tokens into logical units (bracket matching)
array grouped = Parser.C.group(Parser.C.split("int x = foo(1,2);"));

// Strip whitespace
array stripped = Parser.C.hide_whitespaces(Parser.C.split("int x = 1;"));

// Reconstitute with line numbers
string source = Parser.C.reconstitute_with_line_numbers(tokens);
```

#### Shared API (Parser.C and Parser.Pike)
- `tokenize(array(string) lines)` → array of Token objects
- `split(string code)` → array of token strings
- `group(array tokens)` → grouped/nested token array
- `hide_whitespaces(array)` → removes whitespace tokens
- `strip_line_statements(array)` → removes #line directives
- `low_split(string)` → low-level tokenizer
- `reconstitute_with_line_numbers(tokens)` → source with line info
- `simple_reconstitute(tokens)` → source without line info
- `Token` class: has text, line, offset

### Parser.XML — XML Parsing
```pike
// Simple — event-based parser
object parser = Parser.XML.Simple();
parser->_tag_callback = lambda(string tag, mapping attrs, string content) {
  write("Tag: %O attrs: %O\n", tag, attrs);
};
parser->feed("<root><item key=\"val\">text</item></root>");

// DOM — tree-based
Parser.XML.Tree.SimpleNode tree = Parser.XML.Tree.parse_input("<root><a>1</a></root>");
// Navigate tree
array children = tree->get_children();
string tag = tree->get_tag_name();

// Tree types: SimpleNode, SimpleRootNode, SimpleElementNode, SimpleTextNode
// Also: NSTree for namespace-aware, Validating for DTD validation
```

#### Parser.XML Submodules
- `Simple` — event-based (SAX-style)
- `Tree` — DOM tree parsing
- `NSTree` — namespace-aware tree
- `DOM` — alternative DOM
- `SloppyDOM` — lenient parsing of broken XML
- `Validating` — DTD validation

### Parser.LR — LR Parser Generator
```pike
// Generate an LR parser from grammar specification
object parser = Parser.LR.Parser();
parser->set_grammar_from_string("
  %token A B C
  %start expr
  expr: expr '+' term | term ;
  term: term '*' factor | factor ;
  factor: A | B | C ;
");
mixed result = parser->parse(({"A", "+", "B", "*", "C"}));
```

#### Gotchas
- `Parser.C.tokenize` takes array of strings (lines), NOT a single string
- `Parser.C.split` takes a single string
- Token objects have `.text`, `.line` properties
- Parser.HTML is documented separately

## MIME Module (Deep)

```pike
// Parse MIME message from raw string
MIME.Message msg = MIME.Message(
  "Content-Type: text/plain\r\n"
  "Subject: Hello\r\n"
  "\r\n"
  "This is the body");
msg->headers["subject"];       // "Hello"
msg->headers["content-type"]; // "text/plain"
msg->getdata();                // "This is the body"

// Parse headers
array(mixed) parsed = MIME.parse_headers(
  "Content-Type: text/plain\r\n"
  "Subject: Test\r\n"
  "\r\n"
  "Body text");
// Returns: ({ ( ["content-type":"text/plain", "subject":"Test"]), "Body text" })

// Base64
MIME.encode_base64("Hello, World!");  // "SGVsbG8sIFdvcmxkIQ=="
MIME.decode_base64("SGVsbG8=");        // "Hello"

// Base64url (URL-safe variant)
MIME.encode_base64url(data);
MIME.decode_base64url(data);

// Quoted-printable
MIME.encode_qp("Hello = World\r\n");
MIME.decode_qp(encoded);

// UUencode
MIME.encode_uue("filename", data);
MIME.decode_uue(encoded);

// Extension to MIME type mapping
MIME.ext_to_media_type("html");  // "text/html"
MIME.ext_to_media_type("png");   // "image/png"
MIME.ext_to_media_type("txt");   // "text/plain"

// MIME word encoding (RFC 2047)
MIME.encode_word("subject with special chars", "utf-8");
MIME.decode_word("=?utf-8?Q?subject?=");

// Generate boundary string
string boundary = MIME.generate_boundary();

// Tokenize MIME header value
array tokens = MIME.tokenize("text/plain; charset=utf-8");
```

#### MIME.Message Key Methods
- `getdata()` — get body text
- `getdata_parts()` — get body parts (for multipart)
- `headers` — mapping of header name → value
- Cast to string encodes the full message

#### MIME Functions
encode/decode_base64, encode/decode_base64url, encode/decode_qp,
encode/decode_uue, encode/decode_word, encode_words_text,
parse_headers, ext_to_media_type, generate_boundary,
tokenize, tokenize_labled, quote, guess_subtype,
reconstruct_partial

#### Gotchas
- `parse_headers` returns ARRAY of (mapping, body_string), NOT a mapping directly
- `MIME.Message` constructor takes raw string or (StringRange, headers_mapping)
- Headers are lowercase in the mapping ("Content-Type" → "content-type")
- `ext_to_media_type` takes extension WITHOUT the dot

## Protocols.HTTP Async and Session

### Async HTTP with Callbacks
```pike
// Create query with callbacks
Protocols.HTTP.Query q = Protocols.HTTP.Query();
q->set_callbacks(
  lambda(mixed ok, Protocols.HTTP.Query q) {
    // Success callback
    write("Status: %d\n", q->status);
    write("Headers: %O\n", q->headers);
    write("Data: %s\n", q->data());
  },
  lambda(mixed fail, Protocols.HTTP.Query q) {
    // Failure callback
    write("Request failed\n");
  }
);

// Initiate async request
q->async_request("GET", Standards.URI("http://example.com/"));
return -1; // keep backend alive for callbacks
```

### Protocols.HTTP.Query (54 methods)
Key fields and methods:
- `status` — HTTP status code (200, 404, etc)
- `status_desc` — status description
- `headers` — response headers mapping
- `data()` — response body
- `ok` — did request succeed?
- `errno` — system errno on failure
- `set_callbacks(success, failure)` — set async callbacks
- `async_request(method, url, headers, data)` — start async request
- `sync_request(host, port, method, path, headers, data)` — sync request
- `thread_request(...)` — threaded request
- `close()` — close connection
- `timeout` — request timeout
- `data_timeout` — data phase timeout
- `follow_redirects` — follow HTTP redirects
- `ssl_session` — SSL session object for HTTPS

### HTTP Session — Connection Reuse and Cookies
```pike
// Session manages connection pooling, cookies, redirects
Protocols.HTTP.Session session = Protocols.HTTP.Session();

// Synchronous requests (reuses connections)
Protocols.HTTP.Query result = session->get_url("http://example.com/api/data");

// Get just the data string
string body = session->get_url_data("http://example.com/api/data");

// Get as (headers, data) pair
array nice = session->get_url_nice("http://example.com/api/data");

// POST with data
session->post_url("http://example.com/api/submit", "key=value");
string response = session->post_url_data("http://api/endpoint", "payload");

// Async variants
session->async_get_url("http://example.com/", success_cb, fail_cb);
session->async_post_url("http://api/", "data", success_cb, fail_cb);

// Cookie management
session->set_http_cookie("session=abc123", Standards.URI("http://example.com/"));
array cookies = session->get_cookies(Standards.URI("http://example.com/"));
session->all_cookies;  // all stored cookies

// Configuration
session->follow_redirects = 5;  // max redirects
session->maximum_connections_per_server = 4;
session->maximum_total_connections = 20;
session->time_to_keep_unused_connections = 30;
session->default_headers = (["User-Agent": "MyApp/1.0"]);
```

#### Session Methods (41 total)
get_url, get_url_data, get_url_nice, post_url, post_url_data, post_url_nice,
put_url, delete_url, do_method_url, async_get_url, async_post_url,
async_put_url, async_delete_url, async_do_method_url,
set_cookie, set_http_cookie, get_cookies, cookie_lookup,
all_cookies, encode_cookies, decode_cookies, give_me_connection,
return_connection, URL, SessionURL, SessionQuery, Request, Cookie

#### Gotchas
- Async methods require `-1` return from main to keep backend alive
- Session cookies are per-session, not persistent across instances
- `get_url_nice` returns `(headers_mapping, body_string)` pair
- `data()` on Query returns the complete body string
- Query `status` is 0 until request completes

## Stdio.File — Comprehensive Reference (90 methods)

### File Opening
```pike
// Open file for reading
Stdio.File f = Stdio.File("data.txt", "r");

// Open for writing (create/truncate)
Stdio.File f = Stdio.File("output.txt", "wct");

// Open for append
Stdio.File f = Stdio.File("log.txt", "wa");

// Open with specific mode flags
// "r" = read, "w" = write, "a" = append,
// "c" = create, "t" = truncate, "x" = exclusive create

// Read entire file
string data = Stdio.read_file("data.txt");

// Write entire file
Stdio.write_file("output.txt", "content");

// Append to file
Stdio.append_file("log.txt", "new line\n");

// Check if file exists
int exists = Stdio.is_file("data.txt");
int is_dir = Stdio.is_dir("/tmp");
int is_link = Stdio.is_symlink("link");
```

### Network I/O
```pike
// TCP client
Stdio.File conn = Stdio.File();
conn->connect("example.com", 80);
conn->write("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n");
string response = conn->read(4096, 1);  // read up to 4096 bytes
conn->close();

// Async connect
Stdio.File conn = Stdio.File();
conn->async_connect("example.com", 80, lambda(mixed ok) {
  if (ok) conn->write("GET /\r\n");
});

// Unix domain socket
Stdio.File conn = Stdio.File();
conn->connect_unix("/tmp/my.sock");

// Open socket (for servers)
Stdio.File sock = Stdio.File();
sock->open_socket(8080, "0.0.0.0");

// Get peer/local addresses
conn->query_address();  // remote address
conn->query_address(1); // local address
```

### Non-Blocking I/O and Callbacks
```pike
Stdio.File f = Stdio.File();
f->open_socket();

// Set callbacks
f->set_nonblocking(
  lambda(mixed id) { /* read callback */ },
  lambda(mixed id) { /* write callback */ },
  lambda(mixed id) { /* close callback */ }
);

// Or use set_read_callback/set_write_callback individually
f->set_read_callback(lambda(mixed id) {
  string data = f->read(1024, 1);
  if (!data || sizeof(data) == 0) f->close();
});

// Blocking mode
f->set_blocking();

// Set backend (for multiple backends)
f->set_backend(Pike.DefaultBackend);
```

### File Operations
```pike
// Read/Write
string data = f->read(1024, 1);  // read up to 1024 bytes, not all
string all = f->read();           // read to EOF
int written = f->write("data");   // write string

// Seek/Tell
f->seek(0);                // seek to start
f->seek(100, SEEK_SET);    // seek absolute
f->seek(0, SEEK_END);      // seek to end
int pos = f->tell();        // current position
f->truncate(100);           // truncate to 100 bytes

// Stat
Stdio.Stat st = f->stat();
// st->size, st->isreg, st->isdir, st->mtime, st->mode

// Sync
f->sync();  // fsync

// Iterator
String.Iterator iter = f->line_iterator();
foreach (iter; int lineno; string line) {
  // Process line by line
}
```

### Extended Attributes
```pike
f->setxattr("user.key", "value", Stdio.XATTR_CREATE);
string val = f->getxattr("user.key");
array attrs = f->listxattr();
f->removexattr("user.key");
```

### Terminal Control (TTY)
```pike
f->tcgetattr();               // get terminal attributes
f->tcsetattr(attr);           // set terminal attributes
f->tcdrain();                  // wait for output
f->tcflush(Stdio.TCIFLUSH);   // flush input
f->tcsendbreak(0);            // send break
f->tcsetsize(rows, cols);     // set terminal size
```

### Pipe and FD Operations
```pike
// Create pipe
array(Stdio.File) pipe = Stdio.File.pipe();
// pipe[0] = read end, pipe[1] = write end

// Duplicate FD
Stdio.File dup = f->dup();
f->dup2(other_file);  // redirect f's FD to other_file

// Send/receive file descriptors over Unix socket
f->send_fd(some_file);
Stdio.File received = f->receive_fd();

// Take/release raw FD
int fd = f->release_fd();  // take FD without closing
f->take_fd(fd);            // adopt an FD

// Set close-on-exec
f->set_close_on_exec(1);

// Lock
f->lock(Stdio.LOCK_EX);    // exclusive lock
f->trylock(Stdio.LOCK_SH); // shared lock (non-blocking)
```

### Stdio.FILE — Line-Buffered I/O
```pike
// Stdio.FILE adds line-buffered methods on top of Stdio.File
Stdio.FILE f = Stdio.FILE("data.txt", "r");
string line = f->gets();      // read one line (without \n)
int char = f->getchar();      // read one character

// Or use line_iterator (also on Stdio.File)
String.Iterator iter = Stdio.File("data.txt")->line_iterator();
```

#### Key Stdio.File Methods (90 total)
open, close, read, write, seek, tell, truncate, stat, sync,
connect, async_connect, connect_unix, open_socket,
query_address, query_fd, is_open, is_file,
set_nonblocking, set_blocking, set_callbacks,
set_read_callback, set_write_callback, set_close_callback,
set_buffer, set_buffer_mode, set_backend,
set_close_on_exec, set_id, set_keepalive,
line_iterator, pipe, dup, dup2, send_fd,
lock, trylock, peek,
getxattr, setxattr, listxattr, removexattr,
tcgetattr, tcsetattr, tcdrain, tcflush, tcsendbreak, tcsetsize,
read_oob, write_oob, proxy, grantpt, openpt, openat, unlinkat,
assign, take_fd, release_fd, errno, mode

#### Gotchas
- `read(n, 1)` reads UP TO n bytes (non-blocking friendly); `read(n)` waits for exactly n
- `read()` with no args reads ALL data to EOF
- `connect()` returns 0 on failure (check with `is_open()` or `errno()`)
- `write()` returns bytes written, may be less than input — loop for full write
- Non-blocking mode requires backend to be running (`return -1` from main)
- `Stdio.FILE` inherits `Stdio.File` and adds `gets()` and `getchar()`
- `line_iterator()` returns `String.Iterator` — use with foreach
- Use `Stdio.read_file()` / `Stdio.write_file()` for simple file I/O (don't create File objects)

## ADT Module (Deep) — All 20 Data Structures

### ADT.Stack — LIFO Stack
```pike
ADT.Stack s = ADT.Stack();
s->push(1); s->push(2); s->push(3);
s->pop();    // 3
s->top();    // 2 (peek)
sizeof(s);   // 2
s->reset();  // clear all
s->pop();    // 2
s->pop();    // 1
s->pop();    // UNDEFINED (empty — no error)
```

### ADT.Heap — Priority Queue
```pike
ADT.Heap h = ADT.Heap();
h->push(5); h->push(1); h->push(3);
h->pop();   // 1 (smallest first)
h->top();   // 3 (peek at next)
sizeof(h);  // 2
// Elements are compared with `< operator
// Custom: push objects with `<(lfun) for custom ordering
```

### ADT.Set — Unique Collection
```pike
ADT.Set set = ADT.Set(({1, 2, 3}));
set->contains(2);  // 1 (present)
set->add(4);       // add element
set->remove(2);    // remove element
sizeof(set);       // 3
set->add(3);       // no-op, already present
// Set uses `== for comparison
```

### ADT.Queue — FIFO Queue
```pike
ADT.Queue q = ADT.Queue();
q->put("a"); q->put("b"); q->put("c");
q->get();   // "a"
q->peek();  // "b" (peek without removing)
sizeof(q);  // 2
q->flush(); // empty the queue, returns all items
```

### ADT.List — Doubly-Linked List
```pike
ADT.List l = ADT.List();  // deprecated warning in 8.0
l->append("a"); l->append("b"); l->append("c");
l->head();    // "a"
l->tail();    // "c"
sizeof(l);    // 3
l->pop();     // remove from front
l->insert("x"); // insert at front
```

### ADT.History — Fixed-Size Ring Buffer
```pike
ADT.History hist = ADT.History(5);  // max 5 entries
hist->push("e1"); hist->push("e2"); hist->push("e3");
hist->latest();  // "e3"
hist->dump();    // ({"e1", "e2", "e3"})
hist->get(0);    // "e1" (by index, 0=oldest)
sizeof(hist);    // 3
// Auto-discards oldest when exceeding max
```

### ADT.Interval — Numeric Interval
```pike
ADT.Interval iv = ADT.Interval(1.0, 5.0);
iv->contains(3.0);     // 1
iv->intersects(ADT.Interval(4.0, 8.0)); // 1
iv->`&(ADT.Interval(3.0, 7.0));  // intersection: (3.0, 5.0)
iv->`|(ADT.Interval(4.0, 8.0));  // union: (1.0, 8.0)
```

### ADT.BitBuffer — Bit-Level I/O
```pike
ADT.BitBuffer bb = ADT.BitBuffer();
bb->add(42, 8);   // add value 42 as 8 bits
bb->read(4);      // read 4 bits → 2 (0010)
bb->drain();      // get remaining bits
bb->feed("\xFF"); // feed raw bytes
```

### ADT.CircularList — Fixed-Size Circular Buffer
```pike
ADT.CircularList cl = ADT.CircularList(3);  // max 3 elements
cl->push("x"); cl->push("y"); cl->push("z");
(array)cl;  // ({"x", "y", "z"})
cl->push("w");  // overwrites oldest
(array)cl;  // ({"w", "y", "z"})
```

### ADT.Table — Tabular Data
```pike
// Takes (column_names, rows, column_types)
object t = ADT.Table.table(
  ({"Name", "Age"}),
  ({ ({"Alice", "30"}), ({"Bob", "25"}) })
);
// Cast to string for ASCII table display
// Query: t->select("Name"), t->where(lambda(row) { ... })
// Sort: t->sort("Age")
// Sum: t->sum("Age")
```

### Other ADT Structures
- **ADT.Struct** — binary data parsing (documented separately)
- **ADT.CritBit** — crit-bit trie (documented separately)
- **ADT.Trie** — string trie
- **ADT.Relation.Binary** — binary relation (set of pairs)
- **ADT.Priority_queue** — alias for ADT.Heap
- **ADT.Sequence** — array-like sequence with insert/delete

#### ADT Summary Table
| Structure | Key Operations | Use Case |
|-----------|---------------|----------|
| Stack | push/pop/top | LIFO, undo |
| Queue | put/get/peek | FIFO, task queue |
| Heap | push/pop/top | Priority queue, min-first |
| Set | add/remove/contains | Unique collection |
| List | append/pop/head/tail | Linked list |
| History | push/latest/dump | Ring buffer |
| Interval | contains/intersects | Range math |
| BitBuffer | add/read/feed | Bit-level I/O |
| CircularList | push/(cast) | Fixed-size buffer |
| Table | select/where/sort | Tabular data |

## Concurrent Module — Futures and Promises

```pike
// Create a Promise (write end)
Concurrent.Promise p = Concurrent.Promise();

// Get the Future (read end)
Concurrent.Future f = p->future();

// Register callbacks
f->on_success(lambda(mixed value) {
  write("Got: %O\n", value);
});
f->on_failure(lambda(mixed error) {
  write("Error: %O\n", error);
});

// Resolve the promise
p->success(42);  // triggers on_success

// Or reject
// p->failure("something went wrong");

// Already-resolved/rejected futures
Concurrent.Future done = Concurrent.resolve("immediate");
Concurrent.Future fail = Concurrent.reject("error");

// Combinators
Concurrent.all(({ future1, future2, future3 }));
// Resolves when ALL futures resolve → array of values

Concurrent.race(({ future1, future2 }));
// Resolves when FIRST future resolves

Concurrent.first_completed(({ future1, future2 }));
// Same as race

// Map/transform
Concurrent.Future mapped = f->map(lambda(mixed v) { return v * 2; });

// FlatMap chain
Concurrent.Future chained = f->flatmap(lambda(mixed v) {
  return Concurrent.resolve(v + 1);
});

// Fold over multiple futures
Concurrent.fold(futures, 0, lambda(mixed acc, mixed val) { return acc + val; });

// Traverse — apply function to array, return Future of array
Concurrent.traverse(({1, 2, 3}), lambda(int x) {
  return Concurrent.resolve(x * 2);
});

// Backend control
Concurrent.use_backend(Pike.DefaultBackend);
```

#### Key Functions
- `Concurrent.resolve(value)` — already-succeeded Future
- `Concurrent.reject(error)` — already-failed Future
- `Concurrent.all(array(Future))` — all must succeed
- `Concurrent.race(array(Future))` — first wins
- `Concurrent.first_completed(array(Future))` — same as race
- `Concurrent.fold(futures, init, fun)` — accumulate results
- `Concurrent.traverse(array, fun)` — map to Future array

#### Future Methods
- `on_success(cb)`, `on_failure(cb)` — register callbacks
- `map(fun)` — transform result
- `flatmap(fun)` — chain to next Future
- `future()` on Promise — get associated Future

#### Gotchas
- Callbacks run on the Pike backend thread — return -1 from main to keep it alive
- `Concurrent.resolve()` creates already-resolved Future — callback fires immediately on backend tick
- `Concurrent.all()` fails fast — first rejection rejects the combined Future
- `flatmap` callback must return a Future, not a raw value

## System.Inotify — Linux File Monitoring

```pike
// Create inotify instance
System.Inotify.Instance inotify = System.Inotify.Instance();

// Set event callback
inotify->set_event_callback(lambda(System.Inotify.Event ev) {
  write("Event: mask=%O name=%O\n", ev->mask, ev->name);
});

// Add watches
int wd = inotify->add_watch("/tmp",
  System.Inotify.IN_CREATE |
  System.Inotify.IN_DELETE |
  System.Inotify.IN_MODIFY |
  System.Inotify.IN_CLOSE_WRITE);

// Remove watch
inotify->rm_watch(wd);

// Get underlying fd (for select/poll)
int fd = inotify->query_fd();

// Set nonblocking
inotify->set_nonblocking();
```

#### Inotify Constants
| Constant | Value | Meaning |
|----------|-------|---------|
| IN_ACCESS | 1 | File accessed |
| IN_MODIFY | 2 | File modified |
| IN_ATTRIB | 4 | Metadata changed |
| IN_CLOSE_WRITE | 8 | Writable file closed |
| IN_CLOSE_NOWRITE | 16 | Unwritable file closed |
| IN_CLOSE | 24 | Any close |
| IN_OPEN | 32 | File opened |
| IN_MOVED_FROM | 64 | File moved from |
| IN_MOVED_TO | 128 | File moved to |
| IN_MOVE | 192 | Any move |
| IN_CREATE | 256 | File created |
| IN_DELETE | 512 | File deleted |
| IN_DELETE_SELF | 1024 | Watched item deleted |
| IN_ISDIR | 1073741824 | Event is directory |
| IN_ALL_EVENTS | 4095 | All events |

#### Instance Methods (11)
add_watch(path, mask), rm_watch(wd), set_event_callback(cb),
get_event_callback(), set_blocking(), set_nonblocking(),
set_backend(), poll(), query_fd(), create()

#### Gotchas
- Linux-specific (uses inotify syscall)
- Event callback receives Inotify.Event objects with `mask` and `name` fields
- query_fd() returns raw fd — integrate with your event loop
- IN_ISDIR is a flag OR'd into mask, not a separate watch type

## Array Module — 42 Functions

```pike
// Functional operations
Array.map(({1, 2, 3}), lambda(int x) { return x * 2; });
// ({ 2, 4, 6 })

Array.filter(({1, 2, 3, 4, 5}), lambda(int x) { return x > 3; });
// ({ 4, 5 })

Array.reduce(`+, ({1, 2, 3}));  // 6
Array.rreduce(`+, ({1, 2, 3})); // 6 (right-to-left)

Array.all(({1,2,3}), lambda(int x) { return x > 0; });  // 1
Array.any(({1,2,3}), lambda(int x) { return x > 2; });   // 1

// Searching
Array.search_array(({"apple", "banana", "cherry"}), lambda(string s) { return s[0] == 'b'; });
// 1 (index)

// Sorting
Array.sort(({3, 1, 2}));        // ({1, 2, 3}) — returns new sorted array
Array.sort_array(({3, 1, 2}), `>); // ({3, 2, 1}) — custom comparator
Array.dwim_sort_func("b", "a"); // "Do What I Mean" sort for mixed types

// Set operations
Array.uniq(({1, 2, 2, 3, 3})); // ({1, 2, 3}) — unique elements
Array.uniq2(({1, 2, 2, 3}));     // ({1, 2, 3}) — remove consecutive duplicates
Array.common_prefix(({1,2,3}), ({1,2,4})); // ({1, 2})

// Diff
Array.diff(({1,2,3,4}), ({1,3,5}));
// Returns diff structure
Array.diff3(a, b, c); // three-way diff
Array.greedy_diff(a, b); // faster, less optimal diff

// Combinatorics
Array.permute(({1,2,3})); // all permutations
Array.combinations(({1,2,3}), 2); // all 2-element combinations

// Partition
[array neg, array pos] = Array.partition(({ -1, 2, -3, 4 }), lambda(int x) { return x > 0; });
// pos = ({2, 4}), neg = ({-1, -3})

// Transformations
Array.flatten(({{1,2}, {3}, {4,5}})); // ({1, 2, 3, 4, 5})
Array.transpose(({{1,2}, {3,4}}));      // ({ {1,3}, {2,4} })
Array.shuffle(({1,2,3,4,5}));          // random order
Array.everynth(({1,2,3,4,5,6}), 2);   // ({2, 4, 6})
Array.splice(({1,2}), ({"a","b"}));    // ({1, "a", 2, "b"})

// Stack-like operations (on arrays)
Array.push(ref arr, val); // push to end, returns new size
Array.pop(ref arr);       // pop from end
Array.shift(ref arr);     // remove first
Array.unshift(ref arr, val); // prepend

// Utility
Array.count(({1,2,1,3,1})); // ([1:3, 2:1, 3:1])
Array.sum(({1,2,3}));       // 6
Array.sum_arrays(`+, ({1,2}), ({3,4})); // ({4, 6})
Array.enumerate(5);         // ({0, 1, 2, 3, 4})
Array.longest_ordered_sequence(({1,3,2,4,3,5})); // LIS
Array.arrayify(mixed val);  // val if array, else ({val})
Array.columns(data, col_indices); // extract columns

// String split
Array.levenshtein_distance; // (in String, not Array)
```

#### Array Functions Reference (42)
all, any, arrayify, columns, combinations, common_prefix, count,
diff, diff3, diff_compare_table, diff_dyn_longest_sequence,
diff_longest_sequence, dwim_sort_func, enumerate, everynth,
filter, flatten, greedy_diff, interleave_array,
longest_ordered_sequence, lyskom_sort_func, map, oid_sort_func,
partition, permute, pop, push, reduce, rreduce, search_array,
shift, shuffle, sort, sort_array, splice, sum, sum_arrays,
transpose, uniq, uniq2, unshift

#### Gotchas
- `Array.sort()` returns a NEW sorted array (doesn't modify in place)
- `predef::sort()` sorts IN PLACE (modifies the argument)
- `Array.push`/`pop`/`shift`/`unshift` take the array BY REFERENCE
- `Array.permute` generates ALL permutations — expensive for large arrays
- `Array.enumerate` is also available as predef `enumerate()`

## String Module — 34 Functions

```pike
// Case operations
String.capitalize("hello world");  // "Hello world"
String.sillycaps("hello world");    // "hElLo WoRlD"

// Trim whitespace
String.trim_whites("  hello  ");      // "hello"
String.trim_all_whites(" \n hello \t "); // "hello"

// Hex conversion
String.string2hex("Hello");    // "48656C6C6F"
String.hex2string("48656C6C6F"); // "Hello"

// Count occurrences
String.count("hello world", "l");  // 3

// Common prefix/suffix
String.common_prefix(({"foobar", "foobaz", "fooqux"})); // "foo"

// Fuzzy matching
String.levenshtein_distance("kitten", "sitting"); // 3
String.fuzzymatch("hello", "helo");  // similarity score
String.soundex("Robert");  // "R163"

// Roman numerals
String.int2roman(42);  // "XLII"
String.int2char(65);   // "A" (named character)
String.int2hex(255);   // "ff"
String.int2size(1024); // "1.0 kB"

// Expand tabs
String.expand_tabs("hello\tworld"); // tabs → spaces

// Normalize spaces
String.normalize_space("  hello   world  "); // "hello world"

// String width (character width)
String.width("hello");   // 8 (narrow)
String.width("h\u00E9llo"); // 16 (wide)

// HTML utilities
String.HTML.encode("<b>&</b>"); // HTML entity encode

// Range — substring by character indices
String.range("hello", 1, 3); // "el"

// Secure string comparison (timing-safe)
String.secure("secret", "secret"); // 1, constant-time

// Filter non-unicode
String.filter_non_unicode("\xFFhello"); // "hello"

// Implode nicely
String.implode_nicely(({"apple", "banana", "cherry"}));
// "apple, banana and cherry"

// String.Buffer (documented separately)
// String.Replace, String.SingleReplace — compiled replacements
// String.Bootstring — Punycode (RFC 3492)
// String.Elite — "elite speak" text transform
// String.Iterator, String.SplitIterator — iteration
```

#### String Functions Reference (34)
Bootstring, Buffer, Elite, HTML, Iterator, SplitIterator,
Replace, SingleReplace,
capitalize, common_prefix, count, expand_tabs, filter_non_unicode,
fuzzymatch, hex2string, implode_nicely, int2char, int2hex,
int2roman, int2size, levenshtein_distance, normalize_space,
range, secure, sillycaps, soundex, status, string2hex,
trim_all_whites, trim_whites, width

#### Gotchas
- `String.upper_case(s)` and `String.lower_case(s)` are predef functions, NOT String module
- `String.Buffer` is for efficient concatenation (separate section)
- `String.secure()` does timing-safe comparison — use for password/token checks
- `String.width()` returns 8, 16, or 32 indicating character width
- `String.soundex()` implements English Soundex algorithm
- `String.Bootstring` implements Punycode for internationalized domain names
- `String.levenshtein_distance()` returns edit distance (int)

## Protocols.WebSocket (Deep)

```pike
// WebSocket server
void ws_handler(Protocols.WebSocket.Request req) {
  Protocols.WebSocket.Connection ws = Protocols.WebSocket.Connection();
  ws->onopen = lambda() {
    ws->send_text("Welcome!");
  };
  ws->onmessage = lambda(Protocols.WebSocket.Frame frame) {
    ws->send_text("Echo: " + frame->data);
  };
  ws->onclose = lambda(int status, string reason) {
    write("Closed: %d %s\n", status, reason);
  };
  ws->accept(req);
}

// WebSocket server port
Protocols.WebSocket.Port port = Protocols.WebSocket.Port(ws_handler, 8080);

// HTTPS WebSocket server
Protocols.WebSocket.SSLPort ssl_port = Protocols.WebSocket.SSLPort(ws_handler, 8443, ctx);

// Sending frames
ws->send_text("hello");                    // text frame
ws->send_binary("\x00\x01\x02");          // binary frame
ws->send(Protocols.WebSocket.Frame(Protocols.WebSocket.FRAME_TEXT, "data"));
ws->ping("keepalive");                      // ping frame

// Closing
ws->close(Protocols.WebSocket.CLOSE_NORMAL, "bye");
```

#### Connection States
STATE.CONNECTING, STATE.OPEN, STATE.CLOSING, STATE.CLOSED

#### Frame Types
FRAME_TEXT=1, FRAME_BINARY=2, FRAME_CLOSE=8, FRAME_PING=9, FRAME_PONG=10
FRAME_CONTINUATION=0

#### Close Codes
CLOSE_NONE=0, CLOSE_NORMAL=1000, CLOSE_GONE_AWAY=1001,
CLOSE_ERROR=1002, CLOSE_BAD_TYPE=1003, CLOSE_STATUS=1005,
CLOSE_EXTENSION=1010, CLOSE_POLICY=1008, CLOSE_BAD_DATA=1007,
CLOSE_UNEXPECTED=1011

#### Key Classes
- **Port** — WebSocket server (plain HTTP)
- **SSLPort** — WebSocket server (HTTPS/WSS)
- **Request** — incoming handshake request
- **Connection** — WebSocket connection with callbacks
- **Frame** — individual WebSocket frame
- **Parser** — frame parser

#### Gotchas
- `ws->accept(req)` must be called to complete the handshake
- onmessage receives Frame objects — use `frame->data` for payload
- Close code 1005 means "no status received" (sent by peer, never send yourself)
- websocket_version is 13 (RFC 6455)

## Regexp Module

```pike
// Create regexp
Regexp re = Regexp("h.llo");

// Match — returns 1/0
re->match("hello");   // 1
re->match("world");   // 0

// Split — returns array of matched groups
re->split("say hello world"); // ({0}) — match position

// Replace
re->replace("hello there", "hi"); // "hi there"
re->replace("hello hello", lambda(array m, mixed ... extra) { return "HEY"; });

// Method list: match, split, replace, _decode, _encode
```

#### Gotchas
- `Regexp` is a CLASS (not a module with sub-namespaces)
- No PCRE in this build — basic POSIX regex only
- `split` returns match info, not split parts
- `replace` can take a callback for dynamic replacement
- Create once, reuse — compilation is expensive

## Tools.Standalone — CLI Utilities

Pike ships with built-in CLI tools accessible via `pike -x <tool>`:

| Tool | Description |
|------|-------------|
| hilfe | Interactive Pike REPL |
| features | List compiled features/modules |
| benchmark | Run Pike benchmarks |
| monger | Package manager (install/search) |
| dump | Dump compiled bytecode |
| pv | Pike version info |
| httpserver | Simple HTTP file server |
| cgrep | Context-aware grep |
| precompile | Precompile Pike files |
| extract_autodoc | Extract autodoc comments |
| autodoc_to_html | Convert autodoc to HTML |
| check_http | HTTP server checker |
| cflags | Get Pike C compilation flags |

```pike
// Run tools from command line:
// pike -x hilfe          — interactive REPL
// pike -x features       — list available modules
// pike -x monger search mysql
// pike -x monger install Mysql
// pike -x httpserver 8080
```

## Graphics.Graph — Chart Generation

```pike
// Bar chart
object img = Graphics.Graph.bars(([
  "data": ({ ({"Jan", 10}), ({"Feb", 20}), ({"Mar", 15}) }),
  "xsize": 400,
  "ysize": 300,
]));
Stdio.write_file("chart.png", Image.PNM.encode(img));

// Line graph
object line = Graphics.Graph.line(([...]));

// Pie chart
object pie = Graphics.Graph.pie(([...]));

// Graph types: bars, line, pie, polyline, sumbars
// Functions: create_graph, create_bars, create_pie, graph, norm
```

#### Gotchas
- Returns Image.Image objects — use format encoders to save
- Configuration via mapping with "data", "xsize", "ysize" keys
- Data format: array of ({label, value}) pairs

## Crypto Module (62 items) — Hashes, Ciphers, Signatures

```pike
// === ONE-SHOT HASH ===
string sha256 = String.string2hex(Crypto.SHA256.hash("hello"));
// "2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824"

string md5 = String.string2hex(Crypto.MD5.hash("hello"));
string sha1 = String.string2hex(Crypto.SHA1.hash("hello"));
string sha512 = String.string2hex(Crypto.SHA512.hash("hello"));
string ripemd = String.string2hex(Crypto.RIPEMD160.hash("hello"));
string sha3 = String.string2hex(Crypto.SHA3_256.hash("hello"));

// === STREAMING HASH ===
object h = Crypto.SHA256();
h->update("hel");
h->update("lo");
string digest = h->digest();  // same as one-shot
// Methods: init(), update(data), digest(), name(), block_size(), digest_size()

// === HMAC ===
string hmac = Crypto.SHA256.HMAC("secret")("message");
string hex = String.string2hex(hmac);
// HMAC is called as: Crypto.HASH.HMAC(key)(data)

// === RANDOM ===
string bytes = Crypto.Random.random_string(16);  // 16 random bytes
int|mpz rand_int = Crypto.Random.random(100);     // random int in [0, 99]

// === RSA ===
Crypto.RSA rsa = Crypto.RSA();
rsa->generate_key(2048);  // generate 2048-bit key pair
write("key size: %d\n", rsa->key_size());  // 2048

// Sign/verify — pass hash PROGRAM as second arg
string sig = rsa->pkcs_sign("message", Crypto.SHA256);
int ok = rsa->pkcs_verify("message", Crypto.SHA256, sig);  // 1

// Convenience methods (deprecated but functional)
string sig2 = rsa->sha_sign("message");   // SHA1 sign
int ok2 = rsa->sha_verify("message", sig2);  // 1

// Encrypt/decrypt (raw RSA, limited to key_size/8 - padding bytes)
string enc = rsa->encrypt("short");
string dec = rsa->decrypt(enc);  // "short"

// Export keys
string pub_key = rsa->get_public_key()->pkcs_public_key();
// JOSE/JWK support: rsa->jwk(), rsa->jose_sign()

// === AES-CBC ===
string aes_key = Crypto.Random.random_string(32);  // 256-bit
string iv = Crypto.Random.random_string(16);

object enc = Crypto.AES.CBC();
enc->set_encrypt_key(aes_key);
enc->set_iv(iv);
string ct = enc->crypt("hello world!!!!!");  // must be block-aligned (16 bytes)

object dec = Crypto.AES.CBC();
dec->set_decrypt_key(aes_key);
dec->set_iv(iv);
string pt = dec->crypt(ct);  // "hello world!!!!!"

// Other modes: Crypto.AES.CTR(), Crypto.AES.GCM() (if available)
// Other ciphers: Blowfish, DES, DES3, IDEA, CAST, Camellia, Twofish, Serpent

// === PASSWORD HASHING ===
string hash = Crypto.make_crypt_md5("password");  // $1$salt$hash
int ok = Crypto.verify_crypt_md5("password", hash);  // 1

// === UTILITY ===
string rot13 = Crypto.rot13("Hello World");  // "Uryyb Jbeyq"
```

#### Available Hash Algorithms
MD2, MD4, MD5, RIPEMD160, SHA1, SHA224, SHA256, SHA384, SHA512,
SHA3_224, SHA3_256, SHA3_384, SHA3_512

#### Available Ciphers
AES, Blowfish, Camellia, CAST, ChaCha20, DES, DES3, IDEA,
SALSA20, SALSA20R12, Serpent, Twofish, Arcfour (RC4)

#### Cipher Modes
CBC, CTR, GCM, AEAD

#### Key Exchange / Signatures
RSA (45 methods), DSA, DH, ECC

#### Crypto.Random Methods
random(n), random_string(len)

#### Gotchas
- `Crypto.SHA256.hash(data)` — one-shot; `Crypto.SHA256()` — streaming object
- HMAC is called as `Crypto.HASH.HMAC(key)(data)`, not `.HMAC(key, data)`
- RSA `pkcs_sign(data, HashProgram)` takes the PROGRAM, not an instance: `Crypto.SHA256` not `Crypto.SHA256()`
- AES CBC input must be block-aligned (16 bytes). Use `Crypto.AES.CBC().block_size()` to check
- `Crypto.Random.random(n)` returns `Gmp.mpz`, not plain int
- `sha_sign`/`md5_sign` are deprecated — use `pkcs_sign` instead
- No `Crypto.SHA256.hash(...)->bytes()` — hash returns raw string, use `String.string2hex()` for display

## Calendar Module (61 items) — Date/Time

```pike
// Current date
object today = Calendar.Day();
today->format_ymd();       // "2026-04-19"
today->format_ymd_short(); // "20260419"
today->year_no();   // 2026
today->month_no();  // 4
today->month_day(); // 19
today->year_day();  // 109 (day of year, 1-based)
today->week_day();  // 7 (1=Mon..7=Sun)
today->month_name(); // "April"
today->week_name();  // "w16"
today->julian_day(); // 2461150
today->unix_time();  // Unix timestamp (seconds)

// Specific date
object d = Calendar.Day(2024, 6, 15);
d->format_ymd(); // "2024-06-15"

// Current time (Fraction = sub-second precision)
object now = Calendar.now();
now->hour_no();   // 16
now->minute_no(); // 9
now->second_no(); // 48
// now is Calendar.Fraction (has sub-second precision)

// Date arithmetic
object tomorrow = today + 1;
object next_week = today + 7;
object last_week = today - 7;

// Duration (Day - Day)
mixed diff = tomorrow - today;
// diff is a TimeRange representing 1 day

// Parse dates
object parsed = Calendar.parse("%Y-%M-%D", "2024-01-15");
object guessed = Calendar.dwim_day("15 Jan 2024");
object guessed_t = Calendar.dwim_time("2024-01-15 10:30:00");
// dwim = "Do What I Mean" — flexible natural language parsing

// Timezones
object utc = now->set_timezone("UTC");
Calendar.set_timezone("Europe/Stockholm");  // global default

// Components
Calendar.Week();   // current week
Calendar.Month();   // current month
Calendar.Year();    // current year
Calendar.Hour();    // current hour
Calendar.Minute();  // current minute
Calendar.Second();  // current second
Calendar.Fraction(); // with sub-second precision

// Calendar systems
Calendar.Gregorian.Day();  // Gregorian (default)
Calendar.ISO.Day();         // ISO calendar
Calendar.Julian.Day();      // Julian calendar
Calendar.Discordian.Day();  // Discordian (PP 36 of Dsc 3192)
Calendar.Islamic.Day();     // Islamic/Hijri
Calendar.Coptic.Day();      // Coptic
Calendar.Bahai.Day();       // Bahá'í
Calendar.Stardate.now();    // Stardate (TNG)
```

#### Calendar Hierarchy
```
TimeRange (abstract)
  YMD
    Year → Month → Week → Day
  TimeofDay
    Hour → Minute → Second → Fraction
  SuperTimeRange (union of ranges)
```

#### Key Functions
- `Calendar.now()` — current time as Fraction
- `Calendar.Day()`, `Calendar.Day(y,m,d)` — day precision
- `Calendar.parse(format, string)` — parse with format string
- `Calendar.dwim_day(string)` — flexible date parsing
- `Calendar.dwim_time(string)` — flexible datetime parsing
- `Calendar.format_iso(t)`, `Calendar.format_iso_short(t)` — ISO format

#### Gotchas
- Calendar objects do NOT cast to string — use `format_ymd()` or `%O` in sprintf
- `Day - Day` returns a TimeRange, not an int. Use `(tomorrow - today)->how_many(Calendar.Day)` for count
- `Calendar.now()` returns Fraction (sub-second). Cast to Second: `Calendar.now()->second()`
- `week_day()`: 1=Monday, 7=Sunday (ISO convention, NOT 0=Sunday)
- `Calendar.format_day_iso()` takes a timestamp (int), not a Calendar object

## Process Module (22 items) — Spawning and Execution

```pike
// Run command, capture everything
mapping result = Process.run(({"echo", "hello"}));
// (["exitcode": 0, "stdout": "hello\n", "stderr": ""])

// Capture stdout as string
string out = Process.popen("echo hello");  // "hello\n"

// Run command, return exit code
int ec = Process.system("ls -la /tmp");  // 0 on success

// Spawn — full control over stdin/stdout/stderr
object sp = Process.Spawn("grep", ({"grep", "pattern"}));
sp->stdin->write("search this line\n");
sp->stdin->close();
string result = sp->stdout->read();
int exitcode = sp->wait();
// Spawn methods: create, fd, kill, pid, stderr, stdin, stdout, wait

// Spawn a Pike script
object pike_proc = Process.spawn_pike(({"-e", "write(42)"}));

// Shell quoting
string safe = Process.sh_quote("it's a test");  // "it\\'s\\ a\\ test"

// Split quoted string
array parts = Process.split_quoted_string("foo 'bar baz' qux");
// ({"foo", "bar baz", "qux"})

// Find binary in PATH
string path = Process.search_path("ls");  // "/usr/bin/ls"
string full = Process.locate_binary(({"/bin", "/usr/bin"}), "ls");

// Fork
object child = Process.fork();
// Returns 0 in child, Process object in parent

// Daemonize
Process.daemon(0, 0);  // (nochdir, noclose)

// exec — replace current process (never returns)
// Process.exece("/bin/ls", ({"ls", "-la"}), ("PATH": "/usr/bin"));
```

#### Process Functions Reference
create_process, daemon, exec, exece, fork, get_forkd_default,
locate_binary, path_separator, popen, run, search_path,
set_forkd_default, sh_quote, spawn, spawn_pike, split_quoted_string, system

#### Classes
Process (base), Spawn (8 methods), ForkdDecoder, ForkdEncoder, TraceProcess

#### Gotchas
- `Process.run()` takes array of strings, NOT a shell command string
- `Process.popen()` takes a shell command string (goes through /bin/sh)
- `Process.system()` takes a shell command string — stdout goes to terminal, not captured
- `Process.Spawn` stdin/stdout/stderr are Stdio.File objects
- `Process.fork()` returns 0 in child, Process object in parent
- `Process.spawn_pike(args, env, includes)` — spawn a Pike process

## Math Module (21 items) — Constants, Matrix, Angles

```pike
// Constants
Math.pi;   // 3.14159265358979
Math.e;    // 2.71828182845905
Math.inf;  // inf
Math.nan;  // nan

// Logarithms
Math.log2(8.0);    // 3.0
Math.log10(100.0); // 2.0
Math.logn(27.0, 3.0); // base-3 log of 27 (= 0.333...)
// NOTE: logn(x, base) = log(x) / log(base), so logn(27,3) ≈ 0.333
// For log₃(27) = 3, use: 1.0/Math.logn(27.0, 3.0) or log(27.0)/log(3.0)

// Combinatorics
Math.choose(10, 3);  // 120 (C(10,3))

// Factorization
Math.factor(12);  // ({3, 2, 2}) — prime factors

// Angle conversion constants
Math.deg_turn;  // 360 (degrees per turn)
Math.rad_turn;  // 6.28318... (radians per turn)
Math.gon_turn;  // 400 (gradians per turn)
Math.str_turn;  // 6300 (seconds of arc per turn)

// Angle conversion function
// Math.convert_angle(value, from_unit, to_unit)
// Unit strings: "deg", "rad", "gon", "str"

// Matrix types
object m = Math.Matrix(({({1.0, 2.0}), ({3.0, 4.0})}));
m->norm();       // vector norm
m->norm2();      // squared norm
m->transpose();  // transpose
m->t();          // alias for transpose
// Operators: `+, `-, `* (matrix multiply)
// Vector ops: dot_product, cross, convolve, vect
// Dimensions: xsize(), ysize()

// Matrix variants:
// Math.SMatrix — short int matrix
// Math.IMatrix — int matrix
// Math.FMatrix — float matrix
// Math.LMatrix — long matrix
```

#### Gotchas
- `Math.logn(x, base)` computes `log(x)/log(base)` — for log₃(27), use `log(27.0)/log(3.0)` not `Math.logn`
- `Math.factor()` returns array of prime factors: `Math.factor(12)` = `({3, 2, 2})`
- `Math.deg_turn`, `Math.rad_turn` etc. are FLOAT constants (360.0, 2π), not functions
- Matrix `*` operator is matrix multiplication, not element-wise
- `Math.nan` is the object `Math.nan`, not a float literal — `0.0/0.0` throws error

## Debug Module (22 items) — Introspection

```pike
// Memory usage — mapping of allocation categories
mapping mu = Debug.memory_usage();
// (["array": 12345, "string": 67890, ...])

// GC status
mapping gs = Debug.gc_status();
// (["num_objects": 545, "alloc_threshold": 3833, "gc_time": 10386, ...])

// Count objects by type
mapping co = Debug.count_objects();
// (["Stdio.FILE": 3, "master()->dirnode": 8, ...])

// Size of an object in bytes
int bytes = Debug.size_object(some_object);  // e.g., 360

// Reference count
int refs = Debug.refs(some_object);  // e.g., 2

// Find all instances of a class
array clones = Debug.find_all_clones(Stdio.File);
// Returns array of all live Stdio.File objects

// Iterate over all objects
int total = 0;
Debug.map_all_objects(lambda(object o, mixed type) {
  total++;
  return 0;  // continue
});

// Next/prev object in memory
object next = Debug.next(some_object);
object next_obj = Debug.next_object();  // iterate all

// Verify internal consistency
Debug.verify_internals();  // throws on corruption

// Pretty-print memory usage
string report = Debug.pp_memory_usage();

// Per-object usage
string obj_report = Debug.pp_object_usage();

// Tracer — function call tracing
object tracer = Debug.Tracer(0);  // 0 = trace off, 1 = on

// Inspect — interactive inspector
object inspect = Debug.Inspect(lambda() { /* idle callback */ });
```

#### Debug Functions Reference
count_objects, describe_encoded_value, describe_program,
find_all_clones, gc_status, globals, map_all_objects,
memory_usage, next, next_object, pp_memory_usage, pp_object_usage,
prev, refs, size_object, verify_internals, werror

#### Classes
Inspect, Subject, Tracer, Wrapper

#### Gotchas
- `Debug.size_object()` requires an OBJECT, not a value — strings/ints will error
- `Debug.refs()` returns reference count (int)
- `Debug.count_objects()` returns counts keyed by string names, not program objects
- `Debug.find_all_clones(program)` finds ALL live instances — can be expensive
- `Debug.map_all_objects` callback: return 0 to continue, non-zero to stop
- `Debug.Tracer(1)` turns on call tracing; `Debug.Tracer(0)` turns it off

## Yabu Module — Embedded Key-Value Database

```pike
// Open database ("wc" = write/create, "r" = read-only)
object db = Yabu.DB("/path/to/db", "wc");

// Get/create table
object t = db["mytable"];

// Store and retrieve
t["key1"] = "value1";
t["key2"] = ({1, 2, 3});
t["key3"] = (["a": 1]);
string val = t["key1"];  // "value1"
array keys = indices(t); // ({"key1", "key2", "key3"})

// Transaction
object tx = db->transaction();
tx["mytable"]["key"] = "new_value";
tx->commit();  // or tx->abort()

// Lookup table (for indexed lookups)
object ldb = Yabu.LookupDB("/path/to/db", "wc", (["index": "field"]));

// Close — destruct the DB object
destruct(db);
```

#### Gotchas
- `Yabu.DB(path, mode)` — mode is string: "wc" (write/create) or "r" (read-only)
- Values are serialized with encode_value/decode_value — any Pike type works
- Tables are accessed via `db["tablename"]` indexing
- Call `destruct(db)` to flush and close

## Locale Module (16 items) — Internationalization

```pike
// Charset encoding/decoding
object enc = Locale.Charset.encoder("utf-8");
string encoded = enc->feed("hello")->drain();

object dec = Locale.Charset.decoder("utf-8");
string decoded = dec->feed("hello")->drain();

// Normalize charset name
string norm = Locale.Charset.normalize("UTF-8"); // "utf8"

// Gettext (POSIX i18n)
Locale.Gettext.setlocale(Locale.Gettext.LC_ALL, "en_US.UTF-8");
Locale.Gettext.bindtextdomain("myapp", "/usr/share/locale");
Locale.Gettext.textdomain("myapp");
string msg = Locale.Gettext.gettext("Hello");  // translated string
string msg2 = Locale.Gettext.dgettext("myapp", "Hello");  // specific domain

// Locale categories
Locale.Gettext.LC_ALL, LC_COLLATE, LC_CTYPE, LC_MESSAGES,
LC_MONETARY, LC_NUMERIC, LC_TIME

// Locale listing
array langs = Locale.list_languages("eng");

// Translation helper
Locale.register_project("myapp", "/path/to/translations");
Locale.translate("myapp", "id", "Default text");
```

#### Charset Submodule (15 items)
Encoder, Decoder, Tables, DecodeError, EncodeError,
decoder, encoder, decoder_from_mib, encoder_from_mib,
normalize, set_decoder, set_encoder

#### Encoder/Decoder API
Both have: `charset()`, `clear()`, `drain()`, `feed(data)`, `set_replacement_callback(cb)`

#### Gotchas
- Charset encoder/decoder are streaming: call `feed()` then `drain()`
- `Locale.Charset.normalize()` converts charset names to canonical form
- Gettext follows standard POSIX API: bindtextdomain → textdomain → gettext
- Locale.Charset is a MODULE, not a class — use `encoder()`/`decoder()` factory functions

## Thread Module (19 items) — Concurrency

```pike
// Create thread
object t = Thread.thread_create(write, "Hello from thread\n");
t->wait();  // block until thread exits

// Current thread info
Thread.this_thread();    // current Thread object
Thread.all_threads();    // array of all Thread objects

// Mutex (mutual exclusion)
Thread.Mutex mux = Thread.Mutex();
Thread.MutexKey key = mux->lock();    // blocks until available
destruct(key);                         // unlock
Thread.MutexKey try = mux->trylock();  // non-blocking, returns 0 if locked

// Condition variable
Thread.Condition cond = Thread.Condition();
cond->wait(mux);     // wait (releases mutex, re-acquires on wake)
cond->signal();      // wake one waiter
cond->broadcast();   // wake all waiters

// Thread-local storage
Thread.Local tls = Thread.Local();
tls->set(42);
mixed val = tls->get();  // 42 — per-thread value

// Thread-safe queue (FIFO, unlimited)
Thread.Queue q = Thread.Queue();
q->write("item1");  // enqueue
q->write("item2");
mixed item = q->read();  // dequeue (blocks if empty)
int n = sizeof(q);       // current size

// Thread-safe FIFO (size-limited)
Thread.Fifo fifo = Thread.Fifo(10);  // max 10 items
fifo->write("item");  // blocks if full
mixed item = fifo->read();  // blocks if empty

// Resource counting
Thread.ResourceCount rc = Thread.ResourceCount();
Thread.ResourceCountKey rck = rc->acquire();
write("usage: %d\n", rc->usage());  // 1
destruct(rck);  // release

// Thread pool (Farm)
Thread.Farm farm = Thread.Farm();
// Submit work to thread pool

// Thread states
Thread.THREAD_NOT_STARTED  // -1
Thread.THREAD_RUNNING      // 0
Thread.THREAD_EXITED       // 1
Thread.THREAD_ABORTED      // 2
```

#### Thread Classes
- **Thread.Thread** — thread object with wait(), status()
- **Thread.Mutex** — mutual exclusion lock
- **Thread.MutexKey** — lock handle (destruct to unlock)
- **Thread.Condition** — condition variable
- **Thread.Local** — thread-local storage
- **Thread.Queue** — unlimited thread-safe FIFO
- **Thread.Fifo** — size-limited thread-safe FIFO
- **Thread.ResourceCount** — counting semaphore
- **Thread.Farm** — thread pool
- **Thread._Disabled** — disable thread switching

#### Gotchas
- `destruct(key)` releases a Mutex lock — there is no `unlock()` method
- `Thread.Queue.read()` blocks when empty; `Thread.Fifo.write()` blocks when full
- `Thread.Local` values are per-thread — each thread sees its own value
- `Thread.Farm()` creates a thread pool — used for parallel work distribution
- `Thread.thread_create(func, args...)` is the preferred way to create threads
- `Thread.all_threads()` may include the current thread

## ADT.CritBit — Crit-Bit Tries (11 Tree Types)

```pike
// String tree — prefix-based lookup
ADT.CritBit.Tree tree = ADT.CritBit.Tree();
tree["hello"] = 1;
tree["help"] = 2;
tree["world"] = 3;
tree["hello"];  // 1 (exact match)
indices(tree);   // ({"hello", "help", "world"}) — sorted

// Integer tree
ADT.CritBit.IntTree itree = ADT.CritBit.IntTree();
itree[42] = "answer";
itree[100] = "hundred";

// Float tree
ADT.CritBit.FloatTree ftree = ADT.CritBit.FloatTree();
ftree[3.14] = "pi";

// IPv4 tree — stores IPv4 addresses/networks as keys
ADT.CritBit.IPv4Tree ipt = ADT.CritBit.IPv4Tree();
ipt["192.168.1.0/24"] = "lan";  // CIDR notation as key
ipt["10.0.0.1"] = "host";       // exact address as key
// Note: lookup is exact key match, NOT CIDR subnet matching

// BigNum tree (Gmp.mpz keys)
ADT.CritBit.BigNumTree bntree = ADT.CritBit.BigNumTree();

// Date tree (Calendar keys)
ADT.CritBit.DateTree dtree = ADT.CritBit.DateTree();

// Range set — set of ranges
ADT.CritBit.RangeSet rs = ADT.CritBit.RangeSet(ADT.CritBit.IntTree);

// Multi-range set
ADT.CritBit.MultiRangeSet mrs;

// Reverse tree
ADT.CritBit.Reverse rev = ADT.CritBit.Reverse();

// Multi tree (multiple values per key)
ADT.CritBit.MultiTree mt = ADT.CritBit.MultiTree();
```

#### CritBit Tree Types (11)
Tree, IntTree, FloatTree, IPv4Tree, BigNumTree, DateTree,
RangeSet, MultiRangeSet, Reverse, MultiTree, BigNumTree

#### Gotchas
- CritBit trees are sorted by key — iteration is always in sorted order
- IPv4Tree uses CIDR notation strings as keys but does NOT do subnet matching on lookup
- RangeSet takes a tree type as argument: `ADT.CritBit.RangeSet(ADT.CritBit.IntTree)`
- More memory-efficient than mapping for large datasets with common prefixes
- All trees support standard indexing: `tree[key] = val`, `tree[key]`, `indices(tree)`

## Protocols.XMLRPC — XML-RPC Client/Server

```pike
// Encode a method call
string call = Protocols.XMLRPC.encode_call("add", ({1, 2}));
// <?xml version="1.0"?>\n<methodCall>\n<methodName>add</methodName>\n...

// Decode a call
object decoded = Protocols.XMLRPC.decode_call(xml_string);
// Protocols.XMLRPC.Call("add", 2 params)

// Encode response — MUST be array of values
string response = Protocols.XMLRPC.encode_response(({42}));

// Decode response — returns array of values
array vals = Protocols.XMLRPC.decode_response(response);
// ({42})

// Fault response
string fault = Protocols.XMLRPC.encode_response_fault(404, "Not found");

// Client — create with URL
object client = Protocols.XMLRPC.Client("http://example.com/rpc");
// or: Protocols.XMLRPC.Client(Standards.URI("http://example.com/rpc"));

// AsyncClient for non-blocking calls
object async = Protocols.XMLRPC.AsyncClient("http://example.com/rpc");

// magic_false — special value for boolean false in XML-RPC
Protocols.XMLRPC.magic_false;  // special false value
```

#### Key Functions
encode_call(method, params), decode_call(xml),
encode_response(values_array), decode_response(xml),
encode_response_fault(code, string)

#### Classes
Client, AsyncClient, Call, Fault

#### Gotchas
- `encode_response` takes an ARRAY of values, not a single value: `({42})` not `42`
- `decode_response` returns array: `({42})` not `42`
- XML-RPC booleans: use `magic_false` for false, `1` for true
- Call object has `method_name` and `params` fields

## Protocols.IRC — IRC Client

```pike
// Create IRC client
Protocols.IRC.Client client = Protocols.IRC.Client(
  "irc://irc.example.net:6667",
  ([ "nick": "PikeBot",
     "user": "pike",
     "real": "Pike IRC Bot" ])
);

// Or with URI object
Protocols.IRC.Client client = Protocols.IRC.Client(
  Standards.URI("irc://irc.example.net:6667/#channel"),
  ([ "nick": "PikeBot" ])
);

// Request types (16)
// join, part, privmsg, notice, nick, mode, kick,
// names, who, pass, user, ping, pong, join2

// Classes
// Client — main IRC connection
// Channel — channel state
// Person — user representation
// Error — error type
// Raw — raw IRC message parsing
```

#### Gotchas
- Client constructor takes (string_or_uri, options_mapping)
- Options include: nick, user, real, channel
- Requests are in Protocols.IRC.Requests submodule (16 types)

## ADT.Struct — Binary Data Parsing

```pike
// Define a binary structure
ADT.Struct.Header = ADT.Struct.Item();
ADT.Struct.Item = class {
  ADT.Struct.uint8 type;
  ADT.Struct.uint16 length;
  ADT.Struct.uint32 value;
  ADT.Struct.Chars(16) name;
};

// Item types for structure definitions:
// uint8, int8    — 1-byte integer (unsigned/signed)
// uint16, int16  — 2-byte integer
// uint32, int32  — 4-byte integer
// uint64, int64  — 8-byte integer
// Chars(n)       — fixed n-byte string
// Varchars(n)    — variable-length string
// Byte, SByte    — 1-byte (unsigned/signed)
// Word, SWord    — 2-byte
// Long, SLong    — 4-byte
// Drow           — 2-byte (reverse byte order)
// Gnol           — 4-byte (reverse byte order)
```

#### ADT.Struct Item Types (21)
Byte, SByte, Word, SWord, Long, SLong,
uint8, int8, uint16, int16, uint32, int32, uint64, int64,
Chars, Varchars, Drow, Gnol, Item, is_item, is_struct

#### Gotchas
- Item names are case-sensitive: `uint8` not `Uint8`
- `Drow` and `Gnol` are "Word"/"Long" with reversed byte order
- `Chars(n)` for fixed-size, `Varchars(n)` for variable-length strings

## Sql.Sql — Database Access (34 methods)

```pike
// Connect to database
object db = Sql.Sql("sqlite:///path/to/db.sqlite");
// URI format: driver://user:pass@host:port/database
// Supported: sqlite, postgres, pgsql, pgsqls, tds, rsql

// Query (returns array of mappings)
array(mapping(string:mixed)) rows = db->query("SELECT * FROM users WHERE id = %d", 42);
foreach(rows;; mapping row) {
  write("name: %O\n", row->name);
}

// Typed query (returns with proper types instead of all strings)
array(mapping) rows = db->typed_query("SELECT * FROM users");

// Big query (returns result object for streaming)
object result = db->big_query("SELECT * FROM large_table");
while (mapping row = result->fetch_row()) {
  // process row
}

// Streaming query (non-blocking)
object stream = db->streaming_query("SELECT * FROM large_table");

// Quote values for safe SQL
string safe = db->quote("O'Brien");  // "O\'Brien"

// Compile query (prepare statement)
object stmt = db->compile_query("SELECT * FROM users WHERE id = %d");

// Schema inspection
db->list_tables();    // array of table names
db->list_fields("users");  // field info
db->list_dbs();       // list databases

// Date encoding/decoding
db->encode_date(Calendar.Day());  // "2026-04-19"
db->encode_time(Calendar.now());
db->encode_datetime(Calendar.now());
db->decode_date("2026-04-19");  // Calendar object

// Connection info
db->server_info();  // server version string
db->host_info();    // connection info
db->is_open();      // connection status

// Admin
db->create_db("newdb");
db->drop_db("olddb");
db->select_db("database");  // switch database
db->ping();  // keep-alive

// Error handling
db->error();    // last error string
db->sqlstate(); // SQLSTATE code

// Charset
db->set_charset("utf-8");
db->get_charset();
```

#### Sql.Sql Methods (34)
big_query, big_typed_query, case_convert, compile_query, create,
create_db, decode_date, decode_datetime, decode_time, drop_db,
encode_date, encode_datetime, encode_time, error, get_charset,
handle_extraargs, host_info, is_open, list_dbs, list_fields,
list_tables, master_sql, ping, query, quote, reload, select_db,
server_info, set_charset, shutdown, sqlstate, streaming_query,
streaming_typed_query, typed_query

#### Connection URI Formats
- `sqlite:///path/to/db` — SQLite embedded
- `postgres://host/db` — PostgreSQL
- `pgsql://host/db` — PostgreSQL (alternate)
- `pgsqls://host/db` — PostgreSQL with SSL
- `tds://host/db` — TDS (MS SQL / Sybase)
- `rsql://host/db` — Remote Sql

#### Null values
- `Sql.NULL` or `Sql.null` — represents SQL NULL
- Use in queries: `db->query("INSERT INTO t VALUES (%s)", Sql.NULL)`

#### Gotchas
- `query()` returns ALL rows as array of mappings — use `big_query()` for large results
- All values from `query()` are strings; use `typed_query()` for proper types
- `quote()` escapes for SQL — always use it or printf-style `%s`/`%d` placeholders
- `big_query()` returns result object — call `fetch_row()` to iterate
- SQLite returns all values as strings even with typed_query
- Connection string format is URI — must include driver prefix

## Geography Module — Positions and Countries

```pike
// Geographic position
Geography.Position pos = Geography.Position(59.33, 18.07);  // Stockholm
pos->lat();          // 59.33 (latitude)
pos->long();         // 18.07 (longitude)
pos->latitude();     // alias
pos->longitude();    // alias
pos->alt();          // altitude
pos->prettyprint();  // "59°19.800'N, 18°4.200'E"

// Distance between positions
pos->euclidian_distance(other_pos);

// Coordinate systems
pos->UTM();                // UTM coordinates
pos->UTM_zone_number();    // UTM zone
pos->GEOREF();             // GEOREF string
pos->ECEF();               // Earth-Centered Earth-Fixed

// Countries
Geography.Countries.countries;      // list of all countries
Geography.Countries.from_name("Sweden");  // country by name
Geography.Countries.from_domain("se");    // country by TLD
Geography.Countries.continents;     // list of continents

// GeoIP (if database available)
// Geography.GeoIP — IP to country lookup

// PositionRT38 — Swedish RT38 coordinate system
Geography.PositionRT38 pos_rt38 = Geography.PositionRT38();
```

#### Geography.Position Methods (27)
ECEF, GEOREF, RT38, UTM, UTM_offset, UTM_zone_designator,
UTM_zone_number, alt, approx_height, eccentricity_squared,
ellipsoids, equatorial_radius, euclidian_distance, flattening,
height_values, lat, latitude, long, longitude, polar_radius,
prettyprint, set_ellipsoid, set_from_RT38, set_from_UTM, standard_grid

#### Gotchas
- Position constructor: `Position(lat, long)` or `Position(lat, long, alt)`
- Also accepts string: `Position("59°19'48\"N 18°4'12\"E")`
- `euclidian_distance()` is approximate (good for short distances)
- UTM, GEOREF, ECEF are different coordinate representation formats

## Gmp Module — Arbitrary Precision Arithmetic

```pike
// Gmp.mpz — arbitrary precision integer
Gmp.mpz big = Gmp.mpz("123456789012345678901234567890");
big * 2;            // multiplication
big + big;          // addition
big / 3;            // division
big % 100;          // modulo
big->digits();      // decimal string
big->size();        // number of bits

// Gmp.mpf — arbitrary precision float
Gmp.mpf f = Gmp.mpf("3.141592653589793238462643383279");

// Gmp.mpq — rational number
Gmp.mpq r = Gmp.mpq("22/7");

// Factorial
Gmp.fac(100);  // 100!

// bignum — Pike auto-promotes to bignum for large ints
int huge = 2 ** 200;  // auto-bignum
```

#### Gotchas
- Pike auto-promotes large integers to Gmp.mpz — no explicit Gmp needed for big ints
- `Gmp.mpz(string)` creates from decimal string
- `Gmp.fac(n)` computes factorial
- `Gmp.mpz` supports all arithmetic operators: `+, -, *, /, %, **, ^, |, &`

## Protocols.DNS — DNS Resolution (105 items)

```pike
// Module-level convenience functions
array dns = Protocols.DNS.gethostbyname("localhost");
// ({"localhost", ({"127.0.0.1", "::1"}), ({"localhost", "localhost")})
// [0] = canonical name, [1] = IPs, [2] = aliases

array rev = Protocols.DNS.gethostbyaddr("127.0.0.1");
// Same format as gethostbyname

string mx = Protocols.DNS.get_primary_mx("example.com");
// Returns primary MX hostname

// Client for more control
Protocols.DNS.client dns_client = Protocols.DNS.client();
dns_client->gethostbyname("example.com");
dns_client->gethostbyaddr("1.2.3.4");
dns_client->get_mx("example.com");  // array of MX hostnames
dns_client->getsrvbyname("http", "tcp", "example.com");
// Returns SRV records

// Async client
Protocols.DNS.async_client async = Protocols.DNS.async_client();
async->gethostbyname("example.com", lambda(mapping r) {
  write("resolved: %O\n", r);
});

// DNS constants — query types
Protocols.DNS.T_A      // 1 — IPv4 address
Protocols.DNS.T_AAAA   // 28 — IPv6 address
Protocols.DNS.T_CNAME  // 5 — canonical name
Protocols.DNS.T_MX     // 15 — mail exchange
Protocols.DNS.T_NS     // 2 — name server
Protocols.DNS.T_PTR    // 12 — pointer
Protocols.DNS.T_SOA    // 6 — start of authority
Protocols.DNS.T_SRV    // 33 — service
Protocols.DNS.T_TXT    // 16 — text
Protocols.DNS.T_ANY    // 255 — any

// Response codes
Protocols.DNS.NOERROR    // 0
Protocols.DNS.NXDOMAIN   // 3 — domain doesn't exist
Protocols.DNS.SERVFAIL   // 2
Protocols.DNS.REFUSED    // 5
```

#### DNS Client Types
client, async_client, tcp_client, dual_client,
global_client, global_async_client

#### DNS Server Types
server, tcp_server, dual_server, server_base

#### Gotchas
- `gethostbyname` returns `({canonical, ({ips}), ({aliases})})` — NOT a mapping
- `getsrvbyname` takes 3 args: `(service, protocol, domain)` e.g. `("http", "tcp", "example.com")`
- `get_mx` returns array of hostname strings
- Async client takes callback as second argument — requires running backend
- DNS constants like T_A are integers, not strings

```

## Protocols.SMTP — Email Sending

```pike
// Sync client
Protocols.SMTP.Client client = Protocols.SMTP.Client("smtp.example.com", 587);
client->send_message("from@example.com", ({"to@example.com"}),
  "Subject: Hello\r\n\r\nBody text\r\n");

// Verify sender/recipient
client->verify("user@example.com");

// Async client
Protocols.SMTP.AsyncClient async = Protocols.SMTP.AsyncClient("smtp.example.com");

// SMTP server
Protocols.SMTP.Server server = Protocols.SMTP.Server(
  ({"example.com"}),   // local domains
  25,                  // port
  0,                   // bind address
  receive_cb,          // message callback
  sender_cb,           // sender verification
  recipient_cb         // recipient verification
);

// Configuration
Protocols.SMTP.Configuration config = Protocols.SMTP.Configuration(
  ({"trusted-host.com"}),  // trusted hosts
  my_sender_cb,
  my_recipient_cb,
  my_data_cb
);

// Reply codes
Protocols.SMTP.replycodes;
// 211=system status, 220=ready, 250=OK, 354=start input,
// 421=not available, 450=mailbox busy, 500=syntax error, ...
```

#### Classes
Client, AsyncClient, AsyncProtocol, Protocol, Server, Connection,
ClientHelper, Configuration

#### Gotchas
- Client constructor: `(hostname, port)` — port defaults to 25
- `send_message(from, recipients_array, raw_message)` — raw_message includes headers
- Server takes callbacks for sender verification, recipient validation, and message handling
- No built-in TLS — use STARTTLS manually or connect via SSL port

```

## Protocols.HTTP — HTTP Client (79 items)

```pike
// === SIMPLE GET/POST ===
object q = Protocols.HTTP.get_url("http://example.com/api");
write("status: %d\n", q->status);         // 200
write("headers: %O\n", q->headers);       // mapping
write("data: %O\n", q->data());           // response body string

// With custom headers
object q2 = Protocols.HTTP.get_url("http://example.com",
  (["X-Api-Key": "secret"]));

// POST
object q3 = Protocols.HTTP.post_url("http://example.com/api",
  (["field1": "value1", "field2": "value2"]));

// POST with JSON body
object q4 = Protocols.HTTP.post_url("http://example.com/api",
  Standards.JSON.encode((["key": "value"])),
  (["Content-Type": "application/json"]));

// PUT
object q5 = Protocols.HTTP.put_url("http://example.com/api", "body data");

// DELETE
object q6 = Protocols.HTTP.delete_url("http://example.com/api");

// get_url_data — returns body string directly
string body = Protocols.HTTP.get_url_data("http://example.com");

// get_url_nice — returns ({content_type, data})
array nice = Protocols.HTTP.get_url_nice("http://example.com/image.png");

// === URL ENCODING ===
string enc = Protocols.HTTP.http_encode_query((["key": "val ue"]));
// "key=val%20ue"
string enc_str = Protocols.HTTP.http_encode_string("hello world");
string enc_cookie = Protocols.HTTP.http_encode_cookie("val=ue");

// URI encoding
Protocols.HTTP.uri_encode("hello world");   // "hello%20world"
Protocols.HTTP.uri_decode("hello%20world"); // "hello world"
Protocols.HTTP.percent_encode("hello world");
Protocols.HTTP.percent_decode("hello%20world");

// === QUERY OBJECT (full control) ===
Protocols.HTTP.Query q = Protocols.HTTP.Query();
// Sync request
q->sync_request("example.com", 80, "GET / HTTP/1.0\r\n\r\n");
// Async request
q->set_callbacks(lambda(object q) { /* done */ },  // success
                   lambda(object q) { /* fail */ });  // failure
q->async_request("GET", Standards.URI("http://example.com"));

// Thread-based request
q->thread_request("GET", Standards.URI("http://example.com"));

// Query properties after response:
q->status;        // int (200, 404, etc.)
q->status_desc;   // string ("OK", "Not Found")
q->headers;       // mapping(string:string)
q->data();        // full response body
q->total_bytes;   // content-length
q->ok;            // 1 if successful

// === HTTP STATUS CONSTANTS ===
Protocols.HTTP.HTTP_OK              // 200
Protocols.HTTP.HTTP_CREATED        // 201
Protocols.HTTP.HTTP_MOVED_PERM     // 301
Protocols.HTTP.HTTP_FOUND          // 302
Protocols.HTTP.HTTP_NOT_MODIFIED   // 304
Protocols.HTTP.HTTP_BAD            // 400
Protocols.HTTP.HTTP_UNAUTH         // 401
Protocols.HTTP.HTTP_FORBIDDEN      // 403
Protocols.HTTP.HTTP_NOT_FOUND      // 404
Protocols.HTTP.HTTP_BAD_GW         // 502
Protocols.HTTP.HTTP_INTERNAL_ERR   // 500
Protocols.HTTP.HTTP_NOT_IMPL       // 501
Protocols.HTTP.HTTP_UNAVAIL        // 503

// === RESPONSE CODES MAPPING ===
Protocols.HTTP.response_codes;  // mapping(int:string) of all codes

// === ENTITY ENCODING ===
Protocols.HTTP.unentity("&amp; &lt;");  // "& <"
```

#### HTTP Convenience Functions
get_url, get_url_data, get_url_nice, post_url, post_url_data,
post_url_nice, put_url, delete_url

#### URL Encoding Functions
http_encode_query, http_encode_string, http_encode_cookie,
uri_encode, uri_decode, percent_encode, percent_decode,
iri_encode, quoted_string_encode, quoted_string_decode

#### HTTP Methods (low-level)
do_method, do_async_method, do_proxied_method, do_async_proxied_method

#### Gotchas
- `get_url()` returns Query object — access body via `q->data()`
- `get_url_data()` returns body string directly — simpler but less control
- `post_url()` with mapping sends form-encoded; with string sends raw body
- Query has 54 methods including async, TLS, and streaming support
- `http_encode_query` takes mapping and returns query string
- HTTP status constants are integers: `HTTP_OK == 200`

```

## Error Module — Structured Error Handling

```pike
// Error types (12)
Error.Generic      // generic error
Error.BadArgument  // wrong argument type/value
Error.Index        // index out of range
Error.Math         // math error (division by zero, etc.)
Error.Permission   // permission denied
Error.Resource     // resource exhaustion
Error.Decode       // decode/parse failure
Error.Compilation  // compilation error
Error.Cpp          // preprocessor error
Error.MasterLoad   // master loading error
Error.ModuleLoad   // module loading error

// Create structured error
object err = Error.Generic("Something went wrong");
err->message();   // "Something went wrong"
err->backtrace(); // array of backtrace frames
describe_error(err);  // formatted error string

// Throw structured error
throw(err);

// catch/describe_backtrace pattern
mixed error = catch {
  // code that might fail
};
if (error) {
  write("Error: %s\n", describe_error(error));
  write("Backtrace:\n%s\n", describe_backtrace(error[1]));
}

// mkerror — create appropriate error type
object e2 = Error.mkerror(({ "message", backtrace() }));
```

#### Error Handling Patterns
```pike
// Pattern 1: catch with throw array
mixed err = catch {
  throw(({ "error message", backtrace() }));
};
if (err) {
  write("%s\n", err[0]);           // message
  write("%s\n", describe_backtrace(err[1]));  // trace
}

// Pattern 2: structured Error objects
mixed err = catch {
  throw(Error.Generic("bad value"));
};
if (err) {
  write("%s\n", describe_error(err));
}

// Pattern 3: Error.Generic with catch
if (mixed err = catch {
  some_function();
}) {
  werror("Failed: %s\n", describe_error(err));
}
```

#### Gotchas
- `catch` is a keyword/expression, not a function — `catch { ... }` syntax
- `throw` can take any value — array, string, or Error object
- `describe_error()` works with both Error objects and throw arrays
- `describe_backtrace()` takes the backtrace array (usually `err[1]`)
- `Error.Generic("msg")` creates a structured error with automatic backtrace
- `Error.mkerror()` creates the appropriate Error subclass

```

## Val Module — Null and Boolean Values

```pike
// Val.null — represents NULL/missing value
Val.null;          // singleton null object
Val.null == Val.null;  // 1 (identity comparison)
Val.null == 0;         // 0 — NOT equal to zero!
Val.null == UNDEFINED; // 0 — NOT equal to UNDEFINED!

// Val.true / Val.false — explicit boolean objects
Val.true;          // singleton true
Val.false;         // singleton false
Val.true == 1;     // 0 — NOT equal to int 1!
Val.false == 0;    // 0 — NOT equal to int 0!

// Type checks
typeof(Val.null);  // object(is _static_modules.Builtin()->Null)
sprintf("%t", Val.null);  // "object" — NOT "null" or "zero"
sprintf("%t", Val.true);  // "object"

// Val subclasses
Val.Null;    // the Null class/program
Val.Boolean; // the Boolean class/program
Val.True;    // the True class/program
Val.False;   // the False class/program

// Use cases:
// - Val.null for JSON null
// - Val.true/Val.false for JSON true/false
// - Distinguish from Pike's 0/1 integers
```

#### Gotchas
- **Val.true != 1 and Val.false != 0** — they are objects, not integers
- **Val.null != 0** — it is an object, not zero
- `sprintf("%t", Val.null)` returns `"object"` — use `objectp(Val.null)` to check
- Val values are singletons — `Val.true` is always the same object
- Use `Val.null` for JSON null, not Pike's `UNDEFINED` or `0`
- Val.Boolean values cannot be used in arithmetic: `Val.true + 1` will error

## Standards.JSON — JSON Encode/Decode

```pike
// Encode
string json = Standards.JSON.encode(("key": "value", "num": 42));
// {"key":"value","num":42}

// Pretty-print
string pretty = Standards.JSON.encode(data, Standards.JSON.HUMAN_READABLE);
// Multi-line with indentation

// ASCII only (escape non-ASCII)
string ascii = Standards.JSON.encode(data, Standards.JSON.ASCII_ONLY);

// Decode
mixed data = Standards.JSON.decode("{\"key\":\"value\"}");
// (["key":"value"])

mixed arr = Standards.JSON.decode("[1, 2, 3]");
// ({1, 2, 3})

// Validate (returns -1 for valid, 0 for invalid)
int ok = Standards.JSON.validate("[1,2,3]");  // -1
int bad = Standards.JSON.validate("not json");  // 0

// UTF-8 variants
mixed data = Standards.JSON.decode_utf8(utf8_string);
int ok = Standards.JSON.validate_utf8(utf8_string);

// Escape string for JSON
string safe = Standards.JSON.escape_string("hello\"world\n");
// "hello\\\"world\\n"

// JSON null/true/false (maps to Val.null/Val.true/Val.false)
Standards.JSON.null;   // Val.null
Standards.JSON.true;   // Val.true
Standards.JSON.false;  // Val.false

// Encoding with null/boolean
string j = Standards.JSON.encode(([
  "null_val": Standards.JSON.null,
  "bool": Standards.JSON.true
]));

// Error handling
mixed err = catch { Standards.JSON.decode("{bad}"); };
if (err) {
  // Standards.JSON.DecodeError thrown
}

// Validator — custom JSON schema validation
object validator = Standards.JSON.Validator(schema_mapping);
```

#### Encoding Flags
- `Standards.JSON.HUMAN_READABLE` — pretty-print with newlines/indentation
- `Standards.JSON.ASCII_ONLY` — escape all non-ASCII characters
- `Standards.JSON.PIKE_CANONICAL` — Pike canonical encoding

#### Gotchas
- `Standards.JSON.null` is `Val.null` (object, not 0)
- `Standards.JSON.true` is `Val.true` — it is NOT equal to int 1
- `validate()` returns -1 for valid, 0 for invalid (not boolean!)
- `decode()` throws `Standards.JSON.DecodeError` on invalid JSON — use catch
- Pike mappings become JSON objects, Pike arrays become JSON arrays
- JSON numbers that are floats in Pike: `decode("3.14")` returns `3.14` (float)
- JSON numbers that are ints: `decode("42")` returns `42` (int)

## Protocols.HTTP.Server — HTTP Server

```pike
// Basic HTTP server
void handle_request(Protocols.HTTP.Server.Request req) {
  write("Request: %s %s\n", req->request_type, req->not_query);
  write("Query: %O\n", req->variables);      // query string params
  write("Headers: %O\n", req->request_headers);
  write("Cookies: %O\n", req->cookies);
  write("Body: %O\n", req->body_raw);        // raw POST body
  write("IP: %s\n", req->get_ip());
  write("Protocol: %O\n", req->protocol);    // "HTTP/1.1"
  write("Full query: %O\n", req->full_query); // "/path?query=string"

  // Send response
  req->response_and_finish(([
    "error": 200,
    "data": "Hello World",
    "type": "text/html",
    "extra_heads": (["X-Custom": "value"])
  ]));
}

Protocols.HTTP.Server.Port port =
  Protocols.HTTP.Server.Port(handle_request, 8080);
// Args: (callback, port, bind_addr, reuse_port)

// HTTPS server
Protocols.HTTP.Server.SSLPort ssl_port =
  Protocols.HTTP.Server.SSLPort(handle_request, 8443,
    "server.key", "server.crt");

// Utility functions
Protocols.HTTP.Server.http_date(time());  // "Sun, 19 Apr 2026 14:47:44 GMT"
Protocols.HTTP.Server.http_decode_string("hello%20world");  // "hello world"
Protocols.HTTP.Server.extension_to_type("html");  // "text/html"
Protocols.HTTP.Server.filename_to_type("file.css"); // "text/css"

// Other server components
Protocols.HTTP.Server.Filesystem;  // serve static files
Protocols.HTTP.Server.Chained;     // middleware chain
Protocols.HTTP.Server.Proxy;       // reverse proxy
Protocols.HTTP.Server.HeaderParser; // HTTP header parser
```

#### Request Properties (45)
request_type ("GET"/"POST"/etc), not_query (path), full_query,
query (parsed query string), variables (query params mapping),
request_headers, cookies, body_raw, protocol, raw, misc,
get_ip(), connection_timeout_delay, max_request_size

#### Response Mapping Keys
- "error" — HTTP status code (int)
- "data" — response body (string)
- "type" — content type (string)
- "extra_heads" — additional headers (mapping)
- "len" — content length (auto-calculated if omitted)

#### Server Components
Port, SSLPort, Filesystem, Chained, Proxy, Request, HeaderParser

#### Gotchas
- `not_query` is the path WITHOUT query string; `full_query` includes it
- `variables` contains parsed query string params (GET)
- `body_raw` is the raw POST body — parse JSON/form data yourself
- `response_and_finish()` sends response and closes connection
- `http_date()` produces RFC 1123 format (GMT)
- `extension_to_type` and `filename_to_type` return MIME types

## Standards.UUID — UUID Generation

```pike
// UUID versions — all return UUID objects, cast to string
object v1 = Standards.UUID.make_version1(0);      // time-based
object v4 = Standards.UUID.make_version4();          // random
write("v1: %s\n", (string)v1);  // "b4a66d72-3bff-11f1-97d6-000000000000"
write("v4: %s\n", (string)v4);  // "b752baea-2122-4216-a1e5-f1e249796959"

// Namespace-based UUIDs (v3/v5 style)
object dns = Standards.UUID.make_dns("example.com");    // namespace + DNS name
object url = Standards.UUID.make_url("http://example.com");
object oid = Standards.UUID.make_oid("1.2.3.4");
object x500 = Standards.UUID.make_x500("cn=test");

// Null UUID
object nil = Standards.UUID.make_null();
write("null: %s\n", (string)nil);  // "00000000-0000-0000-0000-000000000000"

// Parse string to UUID object
object parsed = Standards.UUID.parse_uuid("b752baea-2122-4216-a1e5-f1e249796959");

// Format UUID object to string
string fmt = Standards.UUID.format_uuid((string)v4);

// UUID object methods
v4->version;           // int (4)
v4->variant;           // int
v4->str;               // string representation
v4->urn;               // "urn:uuid:..."
v4->validate();        // validation check
v4->timestamp;         // timestamp (v1 only)
v4->node;              // node ID (v1 only)
v4->clk_seq;           // clock sequence (v1 only)
v4->posix_time;        // POSIX time (v1)

// Namespace constants (UUID objects)
(string)Standards.UUID.NameSpace_DNS;   // "6ba7b810-9dad-11d1-80b4-00c04fd430c8"
(string)Standards.UUID.NameSpace_URL;   // "6ba7b811-9dad-11d1-80b4-00c04fd430c8"
(string)Standards.UUID.NameSpace_OID;   // "6ba7b812-9dad-11d1-80b4-00c04fd430c8"
(string)Standards.UUID.NameSpace_X500;  // "6ba7b814-9dad-11d1-80b4-00c04fd430c8"
(string)Standards.UUID.Nil_UUID;        // "00000000-0000-0000-0000-000000000000"
```

#### Gotchas
- **make_version1 requires int argument**: `make_version1(0)`, not `make_version1()`
- **UUID functions return objects, not strings**: always `(string)uuid` to get the string
- **make_version3/make_version5** take `(NameSpace_UUID, name_string)` — NameSpace must be UUID object
- **format_uuid takes string**, not UUID object: `format_uuid((string)uuid)`
- **parse_uuid returns string** (raw bytes), not UUID object — construct with `UUID(str)`

## Standards.BSON — BSON Serialization

```pike
// Encode mapping to BSON binary
mapping doc = ([
  "name": "test",
  "count": 42,
  "active": Val.true,
  "tags": ({"a", "b"}),
  "nested": (["key": "val"]),
  "nothing": Val.null
]);
string encoded = Standards.BSON.encode(doc);

// Decode BSON to mapping
mapping decoded = Standards.BSON.decode(encoded);
// (["name":"test", "count":42, "active":Val.true, ...])

// ObjectId
object oid = Standards.BSON.ObjectId();
write("%s\n", (string)oid);  // "69e4ecf4d11384aa9d000001"

// Special BSON types
object bin = Standards.BSON.Binary("data");     // binary data
object rx = Standards.BSON.Regex("pattern", "i"); // regex with flags
object sym = Standards.BSON.Symbol("name");       // symbol type

// BSON type constants (integers)
Standards.BSON.TYPE_STRING;    // 2
Standards.BSON.TYPE_INT32;     // 16
Standards.BSON.TYPE_INT64;     // 18
Standards.BSON.TYPE_FLOAT;     // 1
Standards.BSON.TYPE_BOOLEAN;   // 8
Standards.BSON.TYPE_NULL;      // 10
Standards.BSON.TYPE_ARRAY;     // 4
Standards.BSON.TYPE_DOCUMENT;  // 3
Standards.BSON.TYPE_BINARY;    // 5
Standards.BSON.TYPE_OBJECTID;  // 7
Standards.BSON.TYPE_DATETIME;  // 9

// Binary subtypes
Standards.BSON.BINARY_GENERIC; // 0
Standards.BSON.BINARY_UUID;    // 3
Standards.BSON.BINARY_MD5;     // 5
```

#### BSON Special Types
Binary, ObjectId, Regex(string pattern, string flags), Symbol,
Javascript, Timestamp, MinKey, MaxKey

#### Gotchas
- **BSON uses Val.null/Val.true/Val.false** for null, boolean values
- **ObjectId()** takes no arguments — auto-generates unique ID
- **Regex takes two string args**: pattern and flags (e.g. `"i"`)
- **Timestamp takes optional int**: `Timestamp()` or `Timestamp(n)`
- Decoded ints remain ints, decoded strings remain strings
- **encode_array** takes `array(mapping)`, not `array(mixed)`

## Standards.X509 — Certificate Handling

```pike
// Generate RSA key pair
object rsa = Crypto.RSA();
rsa->generate_key(2048);

// Create self-signed certificate
string cert = Standards.X509.make_selfsigned_certificate(
  rsa,                          // signing key
  3600*24*365,                  // TTL in seconds (1 year)
  ([ "commonName": "example.com" ])  // subject attributes
);

// Decode certificate
object tbs = Standards.X509.decode_certificate(cert);
tbs->subject;          // subject distinguished name
tbs->issuer;           // issuer DN
tbs->not_before;       // validity start (int timestamp)
tbs->not_after;        // validity end (int timestamp)
tbs->serial;           // serial number
tbs->version;          // X.509 version
tbs->public_key;       // public key info
tbs->extensions;       // extensions mapping
tbs->subject_str;      // formatted subject string
tbs->issuer_str;       // formatted issuer string

// Certificate verification
mixed result = Standards.X509.verify_certificate(cert, authorities);
// Returns TBSCertificate on success, throws on failure

// Chain verification
mapping chain_result = Standards.X509.verify_certificate_chain(
  ({cert1, cert2, cert3}),  // chain (leaf first)
  authorities               // trusted CAs
);

// PEM encode certificate
string pem = Standards.PEM.build("CERTIFICATE", cert);
// -----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----

// Parse private key from PEM
object key = Standards.X509.parse_private_key(pem_string);

// Certificate error constants
Standards.X509.CERT_BAD_SIGNATURE;     // 32
Standards.X509.CERT_INVALID;           // 4
Standards.X509.CERT_ROOT_UNTRUSTED;    // 16
Standards.X509.CERT_TOO_NEW;           // 2
Standards.X509.CERT_TOO_OLD;           // 1
Standards.X509.CERT_CHAIN_BROKEN;      // 8

// Key usage constants
Standards.X509.KU_digitalSignature;  // bit flag
Standards.X509.KU_keyCertSign;
Standards.X509.KU_cRLSign;
```

#### TBSCertificate Key Methods (56)
subject, issuer, not_before, not_after, serial, version,
public_key, extensions, subject_str, issuer_str,
ext_basicConstraints, ext_keyUsage, ext_subjectAltName_dNSName,
ext_subjectKeyIdentifier, ext_authorityKeyIdentifier,
keyinfo, validity, der, der_encode

#### Gotchas
- **RSA keygen**: `Crypto.RSA()->generate_key(2048)`, NOT `Crypto.RSA.generate(2048)`
- **make_selfsigned_certificate** takes `(key, ttl_seconds, attributes_mapping)`
- **verify_certificate** returns TBSCertificate object, NOT mapping
- **verify_certificate_chain** returns mapping with error info
- PEM functions: `Standards.PEM.build(label, der_data)` and `simple_decode(pem_string)`
- TBSCertificate has 56 methods for certificate introspection

## Standards.PEM — PEM Encoding

```pike
// Encode binary data as PEM
string pem = Standards.PEM.build("CERTIFICATE", der_data);
// -----BEGIN CERTIFICATE-----
// <base64 encoded data>
// -----END CERTIFICATE-----

// Decode PEM to binary
string der = Standards.PEM.simple_decode(pem_string);

// Message object for more control
object msg = Standards.PEM.Message(pem_string);

// Multiple PEM messages
object msgs = Standards.PEM.Messages(multi_pem_string);
```

#### Exports
Message, Messages, build, simple_decode, decrypt_body, derive_key

## Standards.ASN1 — ASN.1 Encoding

```pike
// OID encoding/decoding
string der = Standards.ASN1.encode_der_oid("1.2.840.113549.1.1.1");
string oid = Standards.ASN1.decode_der_oid(der);

// ASN.1 Types (37 types)
Standards.ASN1.Types.Integer;       // ASN.1 integer
Standards.ASN1.Types.Boolean;       // ASN.1 boolean
Standards.ASN1.Types.Sequence;      // ASN.1 sequence
Standards.ASN1.Types.Set;           // ASN.1 set
Standards.ASN1.Types.BitString;     // bit string
Standards.ASN1.Types.OctetString;   // octet string
Standards.ASN1.Types.Null;          // null
Standards.ASN1.Types.Identifier;    // OID
Standards.ASN1.Types.UTF8String;    // UTF-8 string
Standards.ASN1.Types.PrintableString; // printable string
Standards.ASN1.Types.IA5String;     // IA5 string
Standards.ASN1.Types.UTCTime;       // UTC time
Standards.ASN1.Types.GeneralizedTime; // generalized time
Standards.ASN1.Types.Object;        // generic object
Standards.ASN1.Types.Compound;      // compound type
```

#### Exports
Decode, Types (37 types), decode_der_oid, encode_der_oid

## System Module — OS Interface (Deep)

```pike
// === TIMING ===
object timer = System.Timer();
float elapsed = timer->get();     // seconds since creation (float)
float peek = timer->peek();       // peek without resetting

object time_obj = System.Time();
int sec = time_obj->sec;           // seconds since epoch
int usec = time_obj->usec;        // microseconds
int full = time_obj->usec_full;   // full microseconds since epoch

System.sleep(1);                   // sleep seconds (int or float)
System.usleep(100000);             // sleep microseconds
System.nanosleep(0, 500000000);   // nanosleep(sec, nsec)

// === PROCESS/USER INFO ===
System.getpid();                    // current PID
System.getppid();                   // parent PID
System.getuid();                    // real UID
System.getgid();                    // real GID
System.geteuid();                   // effective UID
System.getegid();                   // effective GID
System.getgroups();                 // supplementary groups
System.getpgrp();                   // process group
System.getsid();                    // session ID
System.gethostname();               // hostname string

// === SYSTEM INFO ===
mapping info = System.uname();
// (["sysname":"Linux", "nodename":"host", "release":"6.12...",
//   "version":"...", "machine":"x86_64"])

array load = System.getloadavg();  // ({1min, 5min, 15min})

// === RESOURCE LIMITS ===
System.getrlimit("RLIMIT_NOFILE");  // ({soft, hard}) or error
System.setrlimit("RLIMIT_NOFILE", ({1024, 65535}));
System.getrlimits();                // all limits

// === FILE OPERATIONS ===
System.chmod("/path/file", 0644);    // change permissions
System.chown("/path/file", uid, gid);
System.symlink("target", "linkpath");
System.readlink("linkpath");        // read symlink target
System.hardlink("source", "dest");  // create hard link
System.utime("file", atime, mtime); // set file times
System.sync();                       // sync filesystem buffers

// === DAEMON ===
System.daemon(1, 1);  // (nochdir, noclose) — daemonize process

// === SYSLOG ===
System.openlog("myapp", 0, 0);      // (ident, option, facility)
System.syslog(6, "message");        // (priority, message)
System.closelog();

// === ERRNO CONSTANTS (140+) ===
System.EACCES;    // 13 — permission denied
System.ENOENT;    // 2 — no such file
System.EEXIST;    // 17 — file exists
System.EINTR;     // 4 — interrupted
System.EINVAL;    // 22 — invalid argument
System.ENOMEM;    // 12 — out of memory
System.ECONNREFUSED; // 111 — connection refused
System.ETIMEDOUT;    // 110 — connection timed out
```

#### Timer Constants
- `System.CPU_TIME_IS_THREAD_LOCAL` — bool
- `System.CPU_TIME_RESOLUTION` — float, seconds
- `System.REAL_TIME_IS_MONOTONIC` — bool
- `System.REAL_TIME_RESOLUTION` — float, seconds

#### Gotchas
- `System.Timer.get()` returns elapsed since creation AND resets the timer
- `System.Timer.peek()` returns elapsed without resetting
- `System.Time.usec_full` is the full microsecond timestamp (not just fractional part)
- `System.sleep()` accepts float: `sleep(0.5)` for 500ms
- `System.getrlimit()` takes string like `"RLIMIT_NOFILE"`, not constant

## ADT.Table — Tabular Data

```pike
// Create table from array of rows + column names
object t = ADT.Table.table(
  ({ ({"Alice", "30"}), ({"Bob", "25"}), ({"Carol", "35"}) }),
  ({"name", "age"})
);

// Render as ASCII table
write("%s\n", (string)t);
//  name    age
// -------------
//  Alice   30
//  Bob     25
//  Carol   35
// -------------

// Column access returns column INDEX (int), not values
t->name;  // 0
t->age;   // 1

// Output formats
ADT.Table.ASCII;       // ASCII table formatter
ADT.Table.SQL;         // SQL insert statements
ADT.Table.Separated;   // delimited output
```

#### Gotchas
- **All values must be strings** in the data array — ints will error
- **Column access returns index**, not data: `t->name` returns 0
- **Cast to string** for ASCII rendering: `(string)t`
- Table is read-only after creation — no row/column manipulation

## Parser.XML — XML Parsing

```pike
// === Parser.XML.Tree (full DOM) ===
string xml = "<root><item id='1'>hello</item><item id='2'>world</item></root>";
object tree = Parser.XML.Tree.parse_input(xml);

// Node types
Parser.XML.Tree.XML_ELEMENT;   // element node type
Parser.XML.Tree.XML_TEXT;      // text node type
Parser.XML.Tree.XML_HEADER;    // XML header
Parser.XML.Tree.XML_COMMENT;   // comment
Parser.XML.Tree.XML_ROOT;      // root node
Parser.XML.Tree.XML_PI;        // processing instruction
Parser.XML.Tree.XML_DOCTYPE;   // doctype

// Tree node methods (65 total)
tree->get_tag_name();            // "root"
tree->get_children();            // array of child nodes
tree->get_elements("item");      // child elements by tag
tree->get_first_element("item"); // first matching child
tree->get_attributes();          // mapping of attributes
tree->get_text();                // text content
tree->get_parent();              // parent node
tree->get_root();                // root of tree
tree->add_child(node);           // add child node
tree->remove_child(node);        // remove child
tree->walk_inorder(func);        // inorder traversal
tree->walk_preorder(func);       // preorder traversal
tree->walk_postorder(func);      // postorder traversal
tree->render_xml();              // serialize to XML string
tree->clone();                   // deep clone
tree->count_children();          // number of children
tree->get_descendants();         // all descendants
tree->get_ancestors();           // all ancestors

// Parse flags
Parser.XML.Tree.PARSE_ENABLE_NAMESPACES;
Parser.XML.Tree.PARSE_FORCE_LOWERCASE;
Parser.XML.Tree.PARSE_WANT_ERROR_CONTEXT;
Parser.XML.Tree.PARSE_DISALLOW_RXML_ENTITIES;

// === Parser.XML.Simple (SAX-like) ===
object parser = Parser.XML.Simple();
parser->parse(xml,
  // tag callback
  lambda(string tag, mapping attrs, string content) {
    write("tag: %s attrs: %O\n", tag, attrs);
  },
  // content callback (optional)
  lambda(string content) {
    // text content
  }
);

// Simple parser methods
parser->define_entity("name", "value");
parser->set_default_attribute("tag", "attr", "value");
```

#### Node Types
ElementNode, TextNode, CommentNode, HeaderNode, PINode,
DoctypeNode, AttributeNode, RootNode
+ Simple variants: SimpleElementNode, SimpleTextNode, etc.

#### Gotchas
- `parse_input()` returns RootNode with tree structure
- `get_text()` on an element returns concatenated text of all child text nodes
- `get_elements("tag")` returns direct children only — use `get_descendants()` for recursive search
- `walk_inorder/preorder/postorder` call function for each node — return `STOP_WALK` to stop
- `Parser.XML.Simple()->parse()` takes callbacks, not returns tree

## Parser.Pike — Pike Source Tokenizer

```pike
// Split Pike source into tokens
string code = "int x = 1; string y = \"hello\";";
array tokens = Parser.Pike.split(code);
// ({"int", " ", "x", " ", "=", " ", "1", ";", " ", "string", ...})

// Group tokens into statements
array groups = Parser.Pike.group(tokens);

// Hide whitespace tokens
array clean = Parser.Pike.hide_whitespaces(tokens);

// Reconstitute with line numbers
string src = Parser.Pike.reconstitute_with_line_numbers(groups);

// Strip line statements (preprocessor)
string stripped = Parser.Pike.strip_line_statements(code);

// Token object type
Parser.Pike.Token;  // token object class
Parser.Pike.UnterminatedStringError;  // error for unclosed strings
```

#### Exports
split, group, hide_whitespaces, low_split,
reconstitute_with_line_numbers, simple_reconstitute,
strip_line_statements, tokenize, Token

## SSL.Context — TLS Configuration (57 methods)

```pike
// Create and configure TLS context
object ctx = SSL.Context();

// Add certificate
ctx->add_cert(Standards.X509.decode_certificate(cert_pem),
              ({private_key_der}), password);

// Or add cert directly
ctx->add_cert(cert_string, ({key_string}));

// Configure version range
ctx->min_version = SSL.Constants.PROTOCOL_TLS_1_2;
ctx->max_version = SSL.Constants.PROTOCOL_TLS_1_3;

// Configure cipher suites
ctx->preferred_suites = ({...});
ctx->configure_suite_b();  // Suite B compliance
ctx->filter_weak_suites(); // remove weak ciphers

// Session cache
ctx->use_cache = 1;
ctx->max_sessions = 1000;
ctx->session_lifetime = 3600;
ctx->active_sessions;  // current session count
ctx->forget_old_sessions();
ctx->purge_session(session_id);

// ECC curves
ctx->ecc_curves = ({...});

// Certificate verification
ctx->verify_certificates = 1;
ctx->require_trust = 1;
ctx->auth_level = SSL.Constants.AUTHLEVEL_ask;
ctx->set_trusted_issuers(({ca_cert_array}));
ctx->set_authorities(({ca_certs}));

// Random
ctx->random;  // random source object

// Key sizes
ctx->long_rsa = 4096;   // long-lived RSA key size
ctx->short_rsa = 2048;  // short-lived RSA key size
```

#### Key Methods (57)
add_cert, get_certificates, get_suites, sort_suites,
configure_suite_b, filter_weak_suites,
verify_certificates, require_trust, auth_level,
set_trusted_issuers, get_trusted_issuers,
set_authorities, get_authorities,
use_cache, max_sessions, session_lifetime,
ecc_curves, signature_algorithms,
min_version, max_version, random

#### SSL Classes
SSL.File(Stdio.File, SSL.Context) — TLS-wrapped file
SSL.Port(callback, SSL.Context) — TLS server port
SSL.Connection — TLS connection state
SSL.ClientConnection, SSL.ServerConnection
SSL.Session — session resumption
SSL.https — HTTPS helper

## Protocols.LDAP — LDAP Client (257 exports)

```pike
// Connect to LDAP server
object ldap = Protocols.LDAP.client("ldap://server:389");
ldap->bind("cn=admin,dc=example,dc=com", "password");

// Search
object result = ldap->search("(objectClass=person)",
  ({"cn", "mail"}),          // attributes to return
  Protocols.LDAP.SCOPE_SUB   // search scope
);

// Search scopes
Protocols.LDAP.SCOPE_BASE;  // 0 — base object only
Protocols.LDAP.SCOPE_ONE;   // 1 — one level
Protocols.LDAP.SCOPE_SUB;   // 2 — entire subtree

// Process results
int count = result->num_entries();
for (int i = 0; i < count; i++) {
  mapping entry = result->fetch(i);
  write("%O\n", entry);
}

// Add entry
ldap->add("cn=new,dc=example,dc=com", ([
  "objectClass": ({"person"}),
  "cn": ({"New User"}),
  "sn": ({"User"})
]));

// Modify entry
ldap->modify("cn=user,dc=example,dc=com", ({
  (["type": Protocols.LDAP.MODIFY_REPLACE,
    "attr": "mail",
    "vals": ({"new@email.com"})])
}));

// Modify operations
Protocols.LDAP.MODIFY_ADD;     // 0
Protocols.LDAP.MODIFY_DELETE;  // 1
Protocols.LDAP.MODIFY_REPLACE; // 2

// Delete entry
ldap->delete("cn=user,dc=example,dc=com");

// DN utilities
string canonical = Protocols.LDAP.canonicalize_dn("CN=User,DC=example,DC=com");
string encoded = Protocols.LDAP.encode_dn_value("special chars");
object filter = Protocols.LDAP.make_filter("(cn=John*)");
mapping url = Protocols.LDAP.parse_ldap_url("ldap://host/dc=example,dc=com??sub?(cn=*)");

// Error codes (50+ constants)
Protocols.LDAP.LDAP_SUCCESS;              // 0
Protocols.LDAP.LDAP_INVALID_CREDENTIALS;  // 49
Protocols.LDAP.LDAP_NO_SUCH_OBJECT;       // 32
Protocols.LDAP.LDAP_BUSY;                 // 51
Protocols.LDAP.LDAP_ALREADY_EXISTS;       // 68
Protocols.LDAP.LDAP_INSUFFICIENT_ACCESS;  // 50
Protocols.LDAP.LDAP_PROTOCOL_ERROR;       // 2
Protocols.LDAP.LDAP_SIZELIMIT_EXCEEDED;   // 4
Protocols.LDAP.LDAP_TIMELIMIT_EXCEEDED;   // 3
Protocols.LDAP.LDAP_REFERRAL;             // 10
```

#### LDAP Client Constructor
`Protocols.LDAP.client(string|void url, object|void context)`
— URL format: `ldap://host:port` or `ldaps://host:port` (SSL)

#### Key LDAP Methods
bind, unbind, search, add, modify, delete, compare,
set_scope, set_basedn, set_option, get_option,
start_tls, whoami, root_dse

#### Gotchas
- LDAP client constructor takes URL string: `"ldap://host:389"`
- Search returns a result object — use `result->num_entries()` and `result->fetch(i)`
- Attribute values are always arrays: `({"value"})`
- Filter syntax: `"(cn=John*)"` — parentheses required
- 257 exports total including attribute type descriptions, syntax definitions, and error codes

## Standards.IDNA — Internationalized Domain Names

```pike
// Convert internationalized domain to ASCII (Punycode)
string ascii = Standards.IDNA.to_ascii("münchen.de");
// "xn--m14nchen.de-87a4455h"

// Convert back to Unicode
string unicode = Standards.IDNA.to_unicode(ascii);

// Zone-level conversion
string zone_ascii = Standards.IDNA.zone_to_ascii("münchen.de");
string zone_uni = Standards.IDNA.zone_to_unicode(zone_ascii);

// Punycode encode/decode
string encoded = Standards.IDNA.Punycode.encode("münchen");
string decoded = Standards.IDNA.Punycode.decode(encoded);
```

#### Exports
to_ascii, to_unicode, zone_to_ascii, zone_to_unicode,
nameprep, Punycode (encode/decode)

## Standards.TLD — Top-Level Domain Data

```pike
// Country code TLDs (253 entries)
mapping cc = Standards.TLD.cc;
// Keys are ISO 3166 country codes: "US", "DE", "UK", etc.

// Generic TLDs
mixed generic = Standards.TLD.generic;
// Array of generic TLD strings: "com", "org", "net", etc.
```

#### Exports
cc (mapping), generic (array)

## Standards.ISO639_2 — Language Codes

```pike
// Look up language name by ISO 639-2 code
string name = Standards.ISO639_2.get_language("eng");  // "English"

// Map to ISO 639-1 (two-letter code)
string code = Standards.ISO639_2.map_to_639_1("eng");  // "en"

// List all languages (504 entries)
array langs = Standards.ISO639_2.list_languages();

// Convert between bibliographic and terminological codes
string t = Standards.ISO639_2.convert_b_to_t("ger");  // "deu"
string b = Standards.ISO639_2.convert_t_to_b("deu");  // "ger"
```

#### Exports
get_language, get_language_b, get_language_t,
list_languages, list_languages_b, list_languages_t,
convert_b_to_t, convert_t_to_b,
list_639_1, map_639_1, map_to_639_1

## Protocols.NNTP — Network News Transfer Protocol

```pike
// NNTP client
object nntp = Protocols.NNTP.client("news.example.com");
// Methods: connect, list_groups, group, article, head, body, post, quit

// Protocol helper
object proto = Protocols.NNTP.protocol("news.example.com", 119);

// Async protocol
object async = Protocols.NNTP.asyncprotocol("news.example.com", 119);
```

#### Exports
client, protocol, protocolhelper, asyncprotocol

## Protocols.TELNET — TELNET Protocol

```pike
// TELNET protocol handler
object telnet = Protocols.TELNET.protocol(Stdio.File fd);

// Key command constants
Protocols.TELNET.IAC;    // 255 — Interpret As Command
Protocols.TELNET.WILL;   // 251
Protocols.TELNET.WONT;   // 252
Protocols.TELNET.DO;     // 253
Protocols.TELNET.DONT;   // 254

// Telnet options
Protocols.TELNET.TEOPT_ECHO;         // 1
Protocols.TELNET.TEOPT_SGA;          // 3 — Suppress Go Ahead
Protocols.TELNET.TEOPT_TTYPE;        // 24 — Terminal Type
Protocols.TELNET.TEOPT_NAWS;         // 31 — Window Size
Protocols.TELNET.TEOPT_LINEMODE;     // 34
Protocols.TELNET.TEOPT_NEW_ENVIRON;  // 39
Protocols.TELNET.TEOPT_AUTHENTICATION; // 37

// Readline helper
object rl = Protocols.TELNET.Readline();
```

#### Exports (155)
155 TELNET protocol constants (commands, options, modes)
plus protocol class, Readline, LineMode helper

## Protocols.Bittorrent — BitTorrent Protocol

```pike
// Key classes
Protocols.Bittorrent.Torrent;    // torrent file handling
Protocols.Bittorrent.Peer;       // peer connection
Protocols.Bittorrent.Tracker;    // tracker client
Protocols.Bittorrent.Generator;  // torrent file creation
Protocols.Bittorrent.Bencoding;  // bencode/decode
Protocols.Bittorrent.PeerID;     // peer ID generation
Protocols.Bittorrent.Port;       // port management
```

#### Exports
Torrent, Peer, Tracker, Generator, Bencoding, PeerID, Port

## Protocols.Ident — RFC 1413 Identification

```pike
// Sync lookup — takes connection object
array result = Protocols.Ident.lookup(Stdio.File connection);
// ({"username", "os_name"})

// Async lookup
Protocols.Ident.lookup_async(connection, callback);

// Async class
object async = Protocols.Ident.AsyncLookup();
```

#### Exports
lookup, lookup_async, AsyncLookup

## Standards.EXIF — EXIF Image Metadata

```pike
// Read EXIF data from JPEG file
object f = Stdio.File("photo.jpg", "r");
mapping exif = Standards.EXIF.get_properties(f);
// Returns mapping with EXIF fields
f->close();
```

#### Exports
get_properties(File, void|mapping), components_config

## Standards.ID3 — ID3 Music Tags

```pike
// Read ID3v2 tag
object tag = Standards.ID3.Tag(Stdio.File("song.mp3", "r"));

// Read ID3v1 tag
object tagv1 = Standards.ID3.Tagv1(Stdio.File("song.mp3", "r"));

// Tag frames (ID3v2)
// APIC=attached picture, COMM=comment, TXXX=user text,
// TRCK=track number, TCON=genre, TCOP=copyright, etc.

// Utility functions
Standards.ID3.decode_string;     // decode ID3 string
Standards.ID3.encode_string;     // encode ID3 string
Standards.ID3.int_to_synchsafe;  // synchsafe integer
Standards.ID3.synchsafe_to_int;  // decode synchsafe
```

#### Exports (42)
Tag, Tagv1, Tagv2, TagHeader, Frame, Buffer,
+ 20+ frame types (Frame_APIC, Frame_COMM, Frame_TXXX, etc.)
+ utility functions (decode_string, encode_string, synchsafe)

## Protocols.IMAP — IMAP Server Framework

```pike
// IMAP server (framework for building IMAP servers)
object server = Protocols.IMAP.server(
  backend_object,    // your implementation
  port_number,       // port to listen on
  new_client_cb      // callback for new connections
);

// IMAP types for request/response formatting
Protocols.IMAP.types.imap_atom;       // atom type
Protocols.IMAP.types.imap_string;     // string type
Protocols.IMAP.types.imap_number;     // number type
Protocols.IMAP.types.imap_list;       // list type
Protocols.IMAP.types.imap_set;        // sequence set
Protocols.IMAP.types.imap_format;     // format helper

// Supported IMAP requests (27)
Protocols.IMAP.requests.append;       // APPEND message
Protocols.IMAP.requests.capability;   // CAPABILITY
Protocols.IMAP.requests.check;        // CHECK
Protocols.IMAP.requests.close;        // CLOSE
Protocols.IMAP.requests.copy;         // COPY messages
Protocols.IMAP.requests.create_mailbox; // CREATE mailbox
Protocols.IMAP.requests.delete;       // DELETE mailbox
Protocols.IMAP.requests.examine;      // EXAMINE
Protocols.IMAP.requests.expunge;      // EXPUNGE
Protocols.IMAP.requests.fetch;        // FETCH
Protocols.IMAP.requests.list;         // LIST
Protocols.IMAP.requests.login;        // LOGIN
Protocols.IMAP.requests.logout;       // LOGOUT
Protocols.IMAP.requests.lsub;         // LSUB
Protocols.IMAP.requests.noop;         // NOOP
Protocols.IMAP.requests.rename;       // RENAME
Protocols.IMAP.requests.search;       // SEARCH
Protocols.IMAP.requests.select;       // SELECT
Protocols.IMAP.requests.status;       // STATUS
Protocols.IMAP.requests.store;        // STORE
Protocols.IMAP.requests.subscribe;    // SUBSCRIBE
Protocols.IMAP.requests.uid;          // UID command
Protocols.IMAP.requests.unsubscribe;  // UNSUBSCRIBE
```

#### Exports
server, parser, types (11), requests (27)

#### Gotchas
- This is a **server framework**, not a client library
- Server constructor: `(backend, port, callback, void|int)`
- 27 request types cover RFC 3501 IMAP4rev1 commands
- `imap_server` is the low-level server, `server` is the high-level one

## Protocols.OBEX — OBEX Protocol

```pike
// OBEX client
object client = Protocols.OBEX.Client();

// OBEX over AT modem
object at_client = Protocols.OBEX.ATClient();

// Request types
Protocols.OBEX.REQ_CONNECT;     // 0x80
Protocols.OBEX.REQ_DISCONNECT;  // 0x81
Protocols.OBEX.REQ_PUT;         // 0x02
Protocols.OBEX.REQ_GET;         // 0x03
Protocols.OBEX.REQ_SETPATH;     // 0x05
Protocols.OBEX.REQ_SESSION;     // 0x07
Protocols.OBEX.REQ_ABORT;       // 0xFF
Protocols.OBEX.REQ_FINAL;       // final bit flag

// Header identifiers
Protocols.OBEX.HI_NAME;         // 0x01 — object name
Protocols.OBEX.HI_TYPE;         // 0x42 — object type
Protocols.OBEX.HI_LENGTH;       // 0xC3 — body length
Protocols.OBEX.HI_BODY;         // 0x48 — body data
Protocols.OBEX.HI_TARGET;       // 0x46 — target
Protocols.OBEX.HI_WHO;          // 0x4A — who
Protocols.OBEX.HI_DESCRIPTION;  // 0x05 — description

// Set path flags
Protocols.OBEX.SETPATH_BACKUP;    // navigate up
Protocols.OBEX.SETPATH_NOCREATE;  // don't create dir

// Header encoding
string encoded = Protocols.OBEX.encode_headers(headers_array);
array decoded = Protocols.OBEX.decode_headers(encoded);
array split = Protocols.OBEX.split_headers(data);
```

#### Exports (38)
Client, ATClient, Request, Headers, HeaderIdentifier,
14 header ID constants, 8 request constants,
setpath flags, encode/decode/split_headers

## Protocols.LPD — Line Printer Daemon

```pike
// LPD client for printing
object lpd = Protocols.LPD.client("printer.example.com", 515);
```

#### Exports
client

## Tools Module — Development Utilities

```pike
// Tools.Hilfe — Interactive Pike REPL
object hilfe = Tools.Hilfe.Evaluator();
Tools.Hilfe.StdinHilfe;  // stdin-based REPL
Tools.Hilfe.GenericHilfe;  // generic REPL
Tools.Hilfe.GenericAsyncHilfe;  // async REPL

// Hilfe commands
Tools.Hilfe.Command;        // command base class
Tools.Hilfe.CommandDump;    // dump state
Tools.Hilfe.CommandReset;   // reset evaluator

// Tools.AutoDoc — Documentation extraction
Tools.AutoDoc.CExtractor;     // C source doc extractor
Tools.AutoDoc.PikeExtractor;  // Pike source doc extractor
Tools.AutoDoc.DocParser;      // doc comment parser
Tools.AutoDoc.BMMLParser;     // BMML format parser

// Flags
Tools.AutoDoc.FLAG_QUIET;     // 0
Tools.AutoDoc.FLAG_NORMAL;    // 1
Tools.AutoDoc.FLAG_VERBOSE;   // 2
Tools.AutoDoc.FLAG_DEBUG;     // 4
Tools.AutoDoc.FLAG_KEEP_GOING; // continue on errors

// Tools.Testsuite — Test runner
object ts = Tools.Testsuite();

// Tools.sed — Stream editor
object sed = Tools.sed();

// Tools.X509 / Tools.PEM — Certificate utilities
object x509_tool = Tools.X509();
object pem_tool = Tools.PEM();

// Tools.Shoot — Benchmark suite (62 benchmarks)
Tools.Shoot.run;             // run benchmarks
Tools.Shoot.tests;           // list of test classes
Tools.Shoot.BinaryTrees;     // binary tree benchmark
Tools.Shoot.SortEqualInts;   // sorting benchmark
Tools.Shoot.RecursiveLoops;  // recursion benchmark
Tools.Shoot.Foreach;         // iteration benchmark
Tools.Shoot.Compile;         // compilation benchmark
```

#### Tools.Shoot Benchmarks (62)
Microbenchmarks: Ackermann, BinaryTrees, NestedLoops,
RecursiveLoops, SortEqualInts, SortOrderedInts,
Foreach, Compile, CompileExec, ArrayAdding, MatrixMult,
StringCreation, TagRemove variants, GC, etc.

#### Gotchas
- `Tools.Hilfe` is the interactive REPL used by `pike -x Hilfe`
- `Tools.Testsuite` runs the Pike test suite
- `Tools.Shoot` is for performance benchmarking, not functional testing

## Protocols.IPv6 — IPv6 Address Handling

```pike
// Parse IPv6 address to array of 8 integers
array(int) addr = Protocols.IPv6.parse_addr("2001:db8::1");
// ({8193, 3512, 0, 0, 0, 0, 0, 1})

// Normalize — expand :: notation
string full = Protocols.IPv6.normalize_addr_basic("2001:db8::1");
// "2001:db8:0:0:0:0:0:1"

string short = Protocols.IPv6.normalize_addr_short("2001:0db8:0000:0000:0000:0000:0000:0001");
// "2001:db8::1"

// Format array back to string
string formatted = Protocols.IPv6.format_addr_short(addr);
// "2001:db8::1"
```

#### Exports
parse_addr, normalize_addr_basic, normalize_addr_short, format_addr_short

## Protocols.Ports — IANA Port Number Lookup

```pike
// Look up service by name
array services = Protocols.Ports.lookup("http");
// ({Service(http 80/tcp), Service(http 80/udp)})

// Each service has properties
write("name: %s port: %d proto: %s\n",
  services[0]->name, services[0]->port, services[0]->proto);

// Look up by service name
Protocols.Ports.lookup("ssh");   // ({Service(ssh 22/tcp)})
Protocols.Ports.lookup("dns");   // ({Service(domain 53/tcp), Service(domain 53/udp)})

// Port database access
Protocols.Ports.tcp;       // mapping of TCP ports/services
Protocols.Ports.udp;       // mapping of UDP ports/services
Protocols.Ports.private_tcp;  // private/unassigned TCP
Protocols.Ports.private_udp;  // private/unassigned UDP
```

#### Exports
lookup, Service, tcp, udp, private_tcp, private_udp, port

#### Gotchas
- `lookup()` takes service name string, returns array of Service objects
- `port()` takes string port number, not int

## Function Module — Higher-Order Functions

```pike
// Y combinator — create recursive function from non-recursive
mixed factorial = Function.Y(lambda(function f, int n) {
  return n <= 1 ? 1 : n * f(n-1);
});
factorial(5);  // 120

// curry — partial application
function add5 = Function.curry(`+)(5);
add5(3);  // 8

// uncurry — reverse of curry
function uncurried = Function.uncurry(curried_func);

// splice_call — call function with args from array
mixed result = Function.splice_call(({1, 2, 3}), `+);  // 6
// First arg is array, second is function

// call_callback — safely call callback (handles 0/unset)
Function.call_callback(0, "arg1", "arg2");  // no-op, no error
Function.call_callback(my_callback, "arg");  // calls my_callback("arg")

// defined — get source location of function
string loc = Function.defined(main);
// "/path/to/file.pike:1"
```

#### Exports
Y, curry, uncurry, splice_call, call_callback, defined

#### Gotchas
- `Function.curry(func)(arg1)` returns a function that takes `arg2` → `func(arg1, arg2)`
- `splice_call` takes `(array, function)`, NOT `(function, array)`
- `call_callback(0, ...)` is safe — no error on null callback

## Program Module — Program Introspection

```pike
// Get all inherited programs
array inherits = Program.inherit_list(program);
array tree = Program.inherit_tree(program);
array all = Program.all_inherits(program);

// Check if program implements an interface
int does = Program.implements(prog_a, prog_b);  // 0 or 1

// Get source location
string loc = Program.defined(program);
// "/path/to/file.pike"
```

#### Exports
inherit_list, inherit_tree, all_inherits, implements, inherits, defined

## Standards.IIM — IPTC-NAA Metadata

```pike
// Read IPTC metadata from image file
object f = Stdio.File("photo.jpg", "r");
mapping info = Standards.IIM.get_information(f);
f->close();

// Available field definitions
Standards.IIM.fields;        // mapping of field names to IDs
Standards.IIM.binary_fields;  // binary field definitions
```

#### Exports
get_information(File), fields (mapping), binary_fields

## Protocols.LMTP — Local Mail Transfer Protocol

```pike
// LMTP server (like SMTP but for local delivery)
Protocols.LMTP.Server;         // server class
Protocols.LMTP.Connection;     // connection handler
Protocols.LMTP.Configuration;  // server configuration
```

#### Exports
Server, Connection, Configuration

## Predef Functions — Complete Reference

```pike
// === OUTPUT ===
write("format %s %d %f\n", "str", 42, 3.14);  // stdout
werror("error: %s\n", msg);                      // stderr

// === PATH MANIPULATION ===
combine_path("/a/b", "c");        // "/a/b/c"
combine_path("/a/b", "../c");    // "/a/c"
combine_path_nt("C:\\a", "b");   // Windows path
combine_path_unix("/a/b", "c");   // Unix path
dirname("/a/b/c");                // "/a/b"
basename("/a/b/c");               // "c"
explode_path("/a/b/c");           // ({"/", "a", "b", "c"})

// === COLLECTION OPERATIONS ===
map(({1, 2, 3}), `+, 1);                   // ({2, 3, 4})
filter(({1, 2, 3, 4}), `<, 3);             // ({1, 2})
sort(({3, 1, 2}));                          // ({1, 2, 3})
reverse(({1, 2, 3}));                       // ({3, 2, 1})
column(({({1,2}), ({3,4})}), 0);            // ({1, 3})
allocate(3, 0);                             // ({0, 0, 0})
enumerate(5);                                // ({0, 1, 2, 3, 4})
enumerate(3, 2, 10);                        // ({10, 12, 14})
// enumerate(count, step, start)

// === STRING SEARCH ===
search("hello world", "world");            // 6
search(({10, 20, 30}), 20);               // 1
has_prefix("hello", "hel");               // 1
has_suffix("hello", "llo");               // 1
has_value(({1, 2, 3}), 2);                // 1
replace("hello world", "world", "pike");  // "hello pike"

// === HASHING ===
hash("hello");                 // int hash value
hash_7_4("hello", 0, 0);       // compat hash

// === RANDOM ===
random(10);                    // random int 0..9
random_string(8);              // 8 random bytes

// === TIME ===
time();                        // seconds since epoch (int)
time(1);                       // seconds with fractional part (float)
gethrtime();                   // high-resolution nanoseconds (int)
gethrvtime();                  // virtual (CPU) time (float)
ctime(time());                 // "Sun Apr 19 17:21:56 2026\n"

mapping tm = localtime(time());
// (["year":126, "mon":3, "mday":19, "hour":17, "min":21, "sec":56,
//   "wday":0, "yday":108, "isdst":0, "timezone":0])

mapping gmtm = gmtime(time()); // same structure, UTC
int epoch = mktime(tm);        // convert mapping back to epoch

// localtime/gmtime fields: year (since 1900), mon (0-11), mday (1-31),
//   hour (0-23), min (0-59), sec (0-59), wday (0-6, Sun=0),
//   yday (0-365), isdst (0/1), timezone (offset from UTC)

// === PROCESS CONTROL ===
exit(0);                       // exit with code
_exit(0);                      // exit without cleanup
atexit(lambda() { /* cleanup */ });  // register exit handler

// === ERROR HANDLING ===
describe_error(err);            // error to string
describe_backtrace(bt);         // backtrace to string
get_backtrace(err);             // extract backtrace from error

// === TYPE CHECKS ===
zero_type(0);                   // 0 (0 is int, not zero type!)
zero_type(UNDEFINED);           // 1 (UNDEFINED has zero type)
undefinedp(UNDEFINED);          // 1
undefinedp(0);                  // 0
intp(42);                       // 1
floatp(3.14);                   // 1
stringp("hi");                  // 1
arrayp(({}));                   // 1
mappingp(([]));                 // 1
multisetp((<>));               // 1
objectp(Val.null);              // 1
programp(this_program);         // 1
functionp(write);               // 1
callablep(write);               // 1

// === OBJECT LIFECYCLE ===
destruct(obj);                  // destroy object
gc();                           // run garbage collector

// === SERIALIZATION ===
encode_value(data);             // serialize to string
decode_value(encoded);          // deserialize
encode_value_canonical(data);   // canonical serialization

// === MISC ===
all_constants();                 // all predefined constants
foo->bar = val;                 // arrow operator: dynamic member access
foo["bar"] = val;               // index operator: same for mappings
```

#### Gotchas
- `zero_type(0)` returns 0 — zero is int, not "zero type"
- `zero_type(UNDEFINED)` returns 1 — UNDEFINED is the zero type
- `undefinedp(UNDEFINED)` is the correct check for missing values
- `ctime()` returns string WITH trailing newline — use `ctime(t)[..<1]` to trim
- `localtime().year` is years since 1900 (e.g., 126 for 2026)
- `localtime().mon` is 0-indexed (January = 0)
- `enumerate(n)` returns `({0, 1, ..., n-1})`
- `enumerate(count, step, start)` returns `({start, start+step, ...})`
- `search()` returns -1 if not found (not 0 or false)
- `hash()` returns int, not hex string
- `column(arr_of_arrays, n)` extracts nth element from each sub-array
- `random(n)` returns int in range `[0, n)`

## master() — Master Object (138 methods)

```pike
object m = master();

// === PATH MANAGEMENT ===
m->pike_include_path;    // array of include directories
m->pike_module_path;     // array of module directories
m->pike_program_path;    // array of program directories
m->system_module_path;   // system module directories

m->add_include_path("/my/includes");
m->add_module_path("/my/modules");
m->add_program_path("/my/programs");
m->remove_include_path("/my/includes");
m->remove_module_path("/my/modules");
m->remove_program_path("/my/programs");

// === ENVIRONMENT ===
m->getenv("HOME");            // get env var
m->getenv();                  // get all env vars
m->putenv("VAR", "value");    // set env var

// === COMPILATION ===
m->compile_file("file.pike");       // compile file to program
m->compile_string("code", "name");  // compile string to program
m->compile_error(filename, line, msg);  // report compile error
m->compile_warning(filename, line, msg); // report compile warning

// === MODULE RESOLUTION ===
m->resolv("Stdio.File");       // resolve symbol name
m->cast_to_program("My.Module"); // resolve to program
m->cast_to_object("Stdio.File", 0); // resolve to object
m->findmodule("Stdio");        // find module file
m->handle_import("path", ...);  // import handler
m->handle_include(header, current, is_local);
m->handle_inherit(inherit_name, current_file);

// === PATH UTILITIES ===
m->normalize_path("/a/./b/../c");  // normalize path
m->is_absolute_path("/a");         // 1
m->is_absolute_path("a");          // 0

// === DESCRIPTION ===
m->describe_function(func);     // describe a function
m->describe_object(obj);        // describe an object
m->describe_program(prog);      // describe a program
m->describe_module(module);     // describe a module

// === PREDEFINES ===
m->add_predefine("NAME", "value");
m->remove_predefine("NAME");
m->get_predefines();            // current predefines

// === SERIALIZATION CODECS ===
m->Codec;     // serialization codec class
m->Encoder;   // encoder class
m->Decoder;   // decoder class

// === CACHES ===
m->fc;              // file cache (mapping)
m->programs;        // loaded programs (mapping)
m->objects;         // loaded objects (mapping)
m->enable_source_cache;  // bool

// === IDENTITY ===
m->_master_file_name;  // "/usr/local/pike/8.0.1116/lib/master.pike"
m->_pike_file_name;    // "pike"
m->is_pike_master;     // 1

// === VERSION ===
m->currentversion;    // version object
m->ver;               // version info

// === FILE I/O ===
m->master_read_file("path");    // read file from master FS
m->master_file_stat("path");    // stat file from master FS
m->master_get_dir("path");      // list dir from master FS
```

#### Key Master Method Groups
- **Path management** (8): add/remove include/module/program paths
- **Compilation** (5): compile_file/string, error/warning handlers
- **Resolution** (6): resolv, cast_to_program/object, findmodule
- **Description** (5): describe_function/object/program/module/backtrace
- **Environment** (3): getenv, putenv
- **Predefines** (4): add/remove/get predefines
- **Caches** (4): fc, programs, objects, source_cache

#### Gotchas
- `master()` returns the master controller object — controls Pike's module system
- `compile_string(code, name)` — name is used for error messages only
- `resolv("Stdio.File")` returns the same as indexing: `Stdio.File`
- `fc` (file cache) maps filenames to programs
- Master paths are arrays — `add_*_path()` appends, `remove_*_path()` removes
- `normalize_path` behavior is OS-dependent