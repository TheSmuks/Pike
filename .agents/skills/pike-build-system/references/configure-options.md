# Configure Options Reference

## Build Commands

Run all commands from the repository root.

```bash
make                                    # build everything
make CONFIGUREARGS="--prefix=/opt/pike" # configure with flags
make MAKE_PARALLEL="-j$(nproc)"         # parallel build
make spotless                           # complete clean
make install                            # install to prefix
make verify                             # run test suite
make documentation                      # build docs
make pike                               # build just the pike binary
```

## Key Configure Flags

| Flag | Purpose |
|------|---------|
| `--prefix=/path` | Installation prefix |
| `--with-dmalloc` | Enable memory debugging |
| `--without-copt` | Disable C compiler optimizations |
| `--with-debug` | Enable debug build |
| `--with-static-linking` | Static linking |
| `--with-thread-library=/path` | Specify thread library path |
| `--without-threads` | Disable threading support |
| `--without-zlib` | Disable Gz module |
| `--with-include-dir=/path` | Additional include search path |
| `--with-lib-dir=/path` | Additional library search path |

## Module-Specific Flags

Each module can be enabled or disabled explicitly. When neither flag is given, configure auto-detects based on available libraries.

| Flag | Module |
|------|--------|
| `--with-mysql` / `--without-mysql` | MySQL database |
| `--with-pgsql` / `--without-pgsql` | PostgreSQL database |
| `--with-gdbm` / `--without-gdbm` | GDBM database |
| `--with-gmp` / `--without-gmp` | GMP bignum |
| `--with-image` / `--without-image` | Image module |

### Module Dependency Paths

For module-specific include and library paths:

```bash
--with-mysql-include-dir=/path/to/mysql/include
--with-mysql-lib-dir=/path/to/mysql/lib
```

The pattern `--with-<module>-include-dir` and `--with-<module>-lib-dir` applies to most modules with external dependencies.

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `PIKE_MODULE_PATH` | Runtime module search path (colon-separated) |
| `PIKE_INCLUDE_PATH` | Runtime include search path (colon-separated) |
| `PIKE_BUILD_OS` | Override OS detection for cross-builds |
| `CFLAGS` | Extra C compiler flags |
| `LDFLAGS` | Extra linker flags |
| `CC` | C compiler override |

## Troubleshooting

### Missing Module After Build

Configure probes for each module's dependencies. If a probe fails, the module is silently skipped.

1. Check configure output for "checking for..." lines related to the missing module.
2. Install the missing dev package (e.g., `libmysqlclient-dev`, `libgmp-dev`).
3. Re-run: `make spotless && make CONFIGUREARGS="--with-<module>"`

### Build Errors After Pulling Changes

Generated files may be stale. Do a full clean:

```bash
make spotless && make
```

### Module Not Loading at Runtime

Check that `PIKE_MODULE_PATH` includes the directory containing the compiled `.so` files:

```bash
pike -e 'werror("%O\n", master()->_pike_module_path)'
```

### Autoconf Issues

If `configure` itself fails or is outdated:

```bash
cd src && ./run_autoconfig .
```

This regenerates the autoconf files from `configure.in` and the `.cmod` sources.
