# Pike Native Modules (src/modules/) — AGENTS.md

## Overview
Core native modules written in C. These are built and installed with Pike by default.

## Modules
|Module|Description|
|---|---|
|_Stdio|File I/O (Stdio.File, Stdio.Port, Stdio.UDP, Stdio.Buffer, Stdio.Stat)|
|_math|Math efuns (sin, cos, sqrt, etc.)|
|system|System calls (getenv, spawn_command, etc.)|
|Image|Image processing (Image.Image, codecs for JPEG/PNG/GIF/TIFF)|
|Parser|HTML/XML/C/Pike/CSV parsing|
|Gmp|Arbitrary precision integers (GNU MP wrapper)|
|MIME|MIME message parsing/encoding|
|_Charset|Character set conversion (100+ charsets)|
|Regexp|POSIX regular expressions|
|Gz|Zlib compression|
|Gdbm|GDBM database|
|Mysql|MySQL client library|
|Postgres|PostgreSQL client library|
|Oracle|Oracle client library|
|Odbc|ODBC client library|
|HTTPLoop|HTTP server loop|
|Pipe|Process pipe management|
|PDF|PDF generation|
|spider|Utility functions (parse_html, etc.)|
|_Roxen|Roxen WebServer support|
|Java|Java JNI bridge|
|Gettext|Internationalization|
|Mird|Mird database|
|Math|Extended math functions|
|CommonLog|Common log format parsing|
|_Image_FreeType|FreeType font rendering|
|_Image_GIF|GIF image codec|
|_Image_JPEG|JPEG image codec|
|_Image_TIFF|TIFF image codec|
|_Image_TTF|TrueType font rendering|
|_Image_XFace|X-Face image support|
|DVB|Digital Video Broadcasting|
|_Ffmpeg|FFmpeg video/audio bridge|
|Fuse|FUSE filesystem bindings|
|Inotify|Linux inotify file monitoring|
|FSEvents|macOS FSEvents monitoring|
|Msql|mSQL database|
|_Protocols_DNS_SD|DNS-SD (Bonjour/Avahi)|
|SANE|Scanner Access (SANE)|
|_WhiteFish|Full-text search engine|
|Wnotify|Windows directory change notifications|
|Yp|NIS/YP directory service|
|sybase|Sybase database|

## Module Structure
Each module has:
- `configure.in` or uses shared `module_configure.in`
- `Makefile.in` or uses shared `static_module_makefile.in` / `dynamic_module_makefile.in`
- C source files (`.c` or `.cmod`)

## .cmod File Convention
Pike C modules use `.cmod` files with special macros:

```c
/* Standard module template */
#include "module.h"
#include "config.h"

PIKECLASS MyClass
{
  CVAR struct my_storage {
    int count;
  };

  PIKEFUN void create(int x)
  {
    THIS->count = x;
  }

  PIKEFUN int get_count()
    flags ID_STATIC;
  {
    RETURN THIS->count;
  }

  PIKEFUN void set_count(int x)
  {
    THIS->count = x;
    pop_n_elems(args);
    push_int(0);
  }

  INIT
  {
    THIS->count = 0;
  }

  EXIT
  {
    /* cleanup */
  }
}

PIKE_MODULE_INIT
{
  INIT;
}

PIKE_MODULE_EXIT
{
  EXIT;
}
```

## Key Macros
- PIKECLASS: Define a Pike class in C
- PIKEFUN: Define a Pike-accessible function
- CVAR: Define per-object C storage
- INIT/EXIT: Object lifecycle hooks
- PIKE_MODULE_INIT/EXIT: Module lifecycle hooks
- THIS: Pointer to object's C storage
- RETURN: Return a value from PIKEFUN
- pop_n_elems(args): Pop arguments from stack
- push_*: Push values to stack (push_int, push_text, push_array, etc.)

## Build
- Each module has its own configure check for library dependencies
- Modules with missing dependencies are silently skipped
- Use `--with-*-include-dir` and `--with-*-lib-dir` configure flags

## Stdio-Specific Notes
- Stdio.File open modes: 'r', 'w', 'rw', 'rwc', 'a' — NOT C fopen modes
- set_nonblocking() requires a running backend
- Stdio.Buffer has locked state during subbuffer views
- sendfile() for zero-copy file transmission
