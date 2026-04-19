# Pike Reference Documentation (refdoc/) — AGENTS.md

## Overview
Reference documentation system for Pike. Extracts documentation from source code comments via an autodoc system and renders it as HTML manuals and module references.

## Documentation Structure
- `refdoc/chapters/` — Manual chapters in XML (autodoc.xml defines the autodoc markup specification itself)
- `refdoc/src_images/` — Source images (logos, navigation icons)
- `refdoc/structure/` — HTML templates, CSS, JS, and XML structure definitions
- `refdoc/presentation/` — Pike scripts that render XML to HTML (make_html.pike, tree-split-autodoc.pike)
- `refdoc/Makefile` — Build system for documentation targets

## Build Targets
From the refdoc directory:
- `make modref` — Module reference (split into individual HTML pages under modref/)
- `make traditional` — Traditional multi-page manual
- `make one_page` — Single-page manual
- `make devel` — Re-copy CSS/JS/images without rebuilding (for style iteration)

From repo root: `make documentation` or `make doc`.

Output directories are generated at build time (`modref/`, `traditional_manual/`, `onepage/`).

## Autodoc System
Documentation is extracted from source code by Tools.AutoDoc.pmod:

### Pike Source (lib/modules/)
```
//! This is a function description.
//! @param x
//!   The x parameter.
//! @returns
//!   The result.
//! @note
//!   Important note.
//! @bugs
//!   Known bug.
//! @seealso
//!   related_function()
```

### C Source (src/)
```
/*! @decl string foo(int x)
 *! @param x
 *!   Description of x.
 *! @returns
 *!   The result string.
 */
```

### Autodoc Markup Rules
- Line-oriented: `//!` prefix for Pike, `/*!` / `*!` for C
- Keywords start with `@` (`@param`, `@returns`, `@note`, `@bugs`, `@seealso`, `@decl`, `@class`, `@endclass`, `@module`, `@endmodule`)
- Trailing `@` on a line continues to the next line
- Blank `//!` line produces a paragraph break
- Escape literal `@` as `@@`
- Full grammar is defined in `chapters/autodoc.xml`

## Key Files
| File | Purpose |
|---|---|
| Makefile | Build targets (modref, traditional, one_page) |
| chapters/autodoc.xml | Autodoc markup specification and grammar |
| chapters/*.xml | Manual chapters (introduction, data_types, control_structures, etc.) |
| structure/modref.html | HTML template for module reference pages |
| structure/chapters.html | HTML template for traditional manual |
| structure/modref.css | Stylesheet for documentation output |
| structure/modref.js | JavaScript for module reference navigation |
| presentation/make_html.pike | XML-to-HTML renderer |
| presentation/tree-split-autodoc.pike | Splits modref into individual HTML pages |
| doxygen.cfg | Doxygen configuration for C source |

## Contributing Documentation
1. Add `//!` comments to Pike source (lib/modules/)
2. Add `/*!` blocks to C source (src/)
3. Run `make documentation` from repo root (or `make modref` from refdoc/)
4. Check output in generated modref/ directory
5. Verify markup grammar against chapters/autodoc.xml
