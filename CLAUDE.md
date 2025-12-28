# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
make                    # Build release version
make debug              # Build debug version with sanitizers
make release            # Explicit release build
make reldebug           # Release with debug info (RelWithDebInfo)
GEN=ninja make          # Use Ninja for faster builds
```

To speed up builds:
- Install ccache (auto-detected)
- Use Ninja: `GEN=ninja make`
- Limit parallelism if needed: `CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make`

## Testing

```bash
make unit               # Run fast unit tests (~1 minute)
make allunit            # Run all unit tests (~1 hour)
./build/debug/test/unittest                    # Run fast tests directly
./build/debug/test/unittest "test/sql/path/to/test.test"  # Run single test
./build/debug/test/unittest "[groupname]"      # Run test group
```

Tests use the sqllogictest framework (`.test` files in `test/sql/`). Prefer sqllogictest over C++ tests. Name slow tests `.test_slow` to exclude them from fast unit tests.

## Code Formatting

```bash
make format-fix         # Format all code
make format-head        # Format only changed files
```

Requires clang-format 11.0.1: `pip install clang-format==11.0.1`

Conventions: tabs for indentation, spaces for alignment, 120 char line limit.

## Architecture Overview

DuckDB is a high-performance analytical database. The core source is in `src/`:

### Query Processing Pipeline

1. **Parser** (`src/parser/`) - Uses PostgreSQL's libpg_query parser, transforms tokens into `SQLStatements`, `Expressions`, and `TableRefs`

2. **Planner** (`src/planner/`) - Converts parse tree to logical query plan (`LogicalOperator` tree), includes the Binder which resolves symbols using the Catalog

3. **Optimizer** (`src/optimizer/`) - Transforms logical plan for better performance (predicate pushdown, expression rewriting, join ordering)

4. **Execution** (`src/execution/`) - Converts logical plan to physical plan (`PhysicalOperators`), uses push-based execution model

### Supporting Components

- **Catalog** (`src/catalog/`) - Tracks tables, schemas, functions; used by Binder for symbol resolution
- **Storage** (`src/storage/`) - Manages physical data in memory and on disk
- **Transaction** (`src/transaction/`) - Manages open transactions, COMMIT/ROLLBACK
- **Common** (`src/common/`) - Shared utilities, types, data structures
- **Function** (`src/function/`) - Built-in function implementations

### Extensions

Extensions live in `extension/`. In-tree extensions (parquet, json, icu) are in the main repo. Out-of-tree extensions are built separately.

Build with specific extensions:
```bash
DUCKDB_EXTENSIONS='json;icu;parquet' make
BUILD_TPCH=1 make                    # Include TPC-H extension
```

## C++ Guidelines

- Use `unique_ptr` over `shared_ptr`; avoid raw `new`/`delete`
- Use `[u]int(8|16|32|64)_t` not `int`/`long`; use `idx_t` for indices/counts
- Use `D_ASSERT` for programmer error assertions (never triggered by user input)
- All functions in `src/` must be in the `duckdb` namespace
- Use `override`/`final` for virtual methods, never repeat `virtual`
- Naming: CamelCase for types/functions, snake_case for variables/files

## Benchmarks

```bash
BUILD_BENCHMARK=1 BUILD_TPCH=1 make
./build/release/benchmark/benchmark_runner --list
./build/release/benchmark/benchmark_runner benchmark/micro/nulls/no_nulls_addition.benchmark
```
