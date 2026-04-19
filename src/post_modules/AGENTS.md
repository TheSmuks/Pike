# Pike Post Modules (src/post_modules/) — AGENTS.md

## Overview
Optional native modules built after Pike itself. These have external library dependencies and are skipped if dependencies are missing.

## Modules
| Module | Description | Dependency |
|---|---|---|
| JSON | JSON encode/decode (Standards.JSON) | Built-in |
| SQLite | SQLite3 database | libsqlite3 |
| Nettle | Cryptographic primitives | libnettle |
| _ADT | C-implemented ADTs (History, CritBit, etc.) | Built-in |
| GL | OpenGL bindings | libGL |
| GLUT | GLUT windowing | libglut |
| GTK1 | GTK1 bindings | gtk1 |
| GTK2 | GTK2 bindings | gtk2 |
| _Regexp_PCRE | PCRE regex | libpcre |
| Unicode | Unicode normalization | ICU or built-in |
| BSON | BSON serialization | Built-in |
| Bz2 | Bzip2 compression | libbz2 |
| CritBit | CritBit tree implementation | Built-in |
| COM | Windows COM | Windows |
| GSSAPI | GSSAPI authentication | libgssapi |
| Kerberos | Kerberos | libkrb5 |
| _Image_SVG | SVG image support | librsvg |
| _Image_WebP | WebP image support | libwebp |
| _Sass | Sass/SCSS compilation | libsass |
| SDL | SDL bindings | libSDL |
| Shuffler | Data shuffling | Built-in |
| VCDiff | VCDiff encoding | Built-in |
| ZXID | SAML/Liberty Alliance | zxid |

## Key Differences from src/modules/
1. **Optional**: Missing dependencies → module silently skipped
2. **Post-build**: Compiled after Pike binary is available
3. **Independent configure**: Each module checks its own dependencies
4. **Dynamic loading**: Most load as shared objects (.so/.dylib)

## Build
- `make` from repo root handles post_modules automatically
- Individual module: `cd src/post_modules/ModuleName && ./configure && make`
- If a module fails to configure, it's skipped (not an error)

## Important Module Notes

### JSON (Standards.JSON)
- encode(data) / decode(string)
- Flags: ASCII_ONLY(1), HUMAN_READABLE(2), PIKE_CANONICAL(4)
- NaN/Infinity → JSON null (silent conversion)
- Mapping keys must be strings for JSON

### SQLite
- Use via Sql.Sql("sqlite://path/to/db")
- num_rows() throws (SQLite doesn't know count)
- Binary data via (<"data">) multiset wrapper

### Nettle/Crypto
- Nettle is low-level C bindings
- Crypto module (in lib/modules/) is the high-level Pike wrapper
- Hash: Nettle.SHA256, Nettle.MD5, etc.
- Cipher: Nettle.AES, Nettle.DES, etc.
- HMAC: Nettle.HMAC(hash)
