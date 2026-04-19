---
name: pike-build-system
description: Building Pike from source, configure options, module builds, and troubleshooting
globs:
  - Makefile
  - "**/Makefile"
  - configure
  - src/run_autoconfig
metadata:
  version: "1.0.0"
  organization: pike-lang
---

# Pike Build System

## Overview

Pike uses an autoconf-based build system with a meta-Makefile at the repository root. The meta-Makefile delegates to `src/Makefile` and handles OS/arch detection, configure runs, and parallel builds. Build artifacts land in `build/<os-arch>/`.

## Key Concepts

- **Meta-Makefile** — The top-level `Makefile` orchestrates the entire build. Never `cd src` and run make directly.
- **Auto-detection** — `configure` probes for libraries and headers. If a dependency is missing, the corresponding module is silently omitted from the build; this is not an error.
- **Build directory** — All compiled output goes under `build/<os-arch>/` (e.g., `build/linux-x86_64/`). The built binary is `build/<os-arch>/pike`.
- **CONFIGUREARGS** — Pass configure flags through this variable on the make command line.
- **MAKE_PARALLEL** — Pass parallel-build flags through this variable.

## Rules

### 1. Run make from the repo root, never from src/

The top-level Makefile handles configure, build directory setup, and module resolution. Running make inside `src/` skips these steps and produces an incomplete or broken build.

**WRONG:**
```bash
cd src && ./configure && make
```

**CORRECT:**
```bash
make CONFIGUREARGS="--prefix=/opt/pike"
```

### 2. Missing dependencies silently skip modules

If configure cannot find a library (MySQL, GMP, etc.), it disables that module without failing the build. Check configure output for "checking for..." lines to see what was detected.

**WRONG:**
```bash
# Build succeeds, then wondering why Mysql module is missing
pike -e 'Sql.Sql; // Error: no Mysql support'
```

**CORRECT:**
```bash
# Verify module availability after build
pike -e 'foreach(indices(master()->resolv("Mysql")), m) werror("%O\n", m)'
# Or check configure output:
make CONFIGUREARGS="--with-mysql" 2>&1 | grep -i mysql
```

### 3. Pass configure flags via CONFIGUREARGS, not as bare arguments

The meta-Makefile wraps the configure step. Flags must go through `CONFIGUREARGS` so the meta-Makefile forwards them correctly.

**WRONG:**
```bash
./configure --with-debug
```

**CORRECT:**
```bash
make CONFIGUREARGS="--with-debug"
```

### 4. Use MAKE_PARALLEL for parallel builds

Parallelism is not enabled by default. Pass `-j` through `MAKE_PARALLEL` to utilize multiple cores.

**WRONG:**
```bash
make -j$(nproc)
```

**CORRECT:**
```bash
make MAKE_PARALLEL="-j$(nproc)"
```

### 5. Clean builds with spotless, not manual deletion

The build tree contains generated files that simple `rm` or `make clean` may not fully remove. Use `make spotless` to reset to a pristine state.

**WRONG:**
```bash
rm -rf build/
make clean
```

**CORRECT:**
```bash
make spotless
```

### 6. Find the built binary in the build directory

After a successful build, the Pike executable is at `build/<os-arch>/pike`, not in `src/` or the repo root.

**WRONG:**
```bash
./pike --version
```

**CORRECT:**
```bash
build/linux-x86_64/pike --version
# or after install:
pike --version
```
