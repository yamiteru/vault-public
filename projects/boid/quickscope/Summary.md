# quickscope

Fast lexical scope extraction from source code ASTs using [tree-sitter](https://tree-sitter.github.io/).

## Purpose

This library powers the code analysis engine for an AI SaaS platform. Developers connect their repositories, and the system scans the entire git history commit-by-commit, analyzing code incrementally (only changed files per commit). This temporal analysis provides AI agents ("minions") with metrics trends over time, identifying potentially problematic files that need improvement. The minions wake periodically and collaborate via MAD (Multi-Agent Debate) to produce PRs that improve the codebase.

## Performance Requirements

Because we scan millions of files per second across thousands of repositories, every microsecond matters. Tree-sitter parsing dominates runtime, but even single-digit percentage improvements in the walker are significant at scale.

**Zero tolerance for:**
- Function pointers and unnecessary indirection
- Data copying/cloning
- Wasted space or CPU instructions
- Cache misses
- Data structures that don't fit cache lines or don't pack evenly

**This library must be the fastest in its category. No exceptions.**

## Design Constraints

- **Single-pass tree traversal** — One walk over the tree-sitter AST with O(1) inline reference resolution via live binding table.
- **No heap allocation during traversal** — Fixed-size inline stacks on the stack frame. Only result vectors and tree-sitter's parse tree allocate.
- **Cache-friendly data layout** — Parallel arrays (struct-of-arrays) for locations, metrics, and scores.
- **Aggressive inlining** — All hot-path methods (21 total) use `#[inline(always)]` to eliminate call overhead.

## Current Capabilities

Walks a syntax tree in a single pass and collects lexical scopes with their locations, metrics (depth, is_async, argument count, binding count, return count, throw count, branch count, jump count, try count, call count, await count, assignment count, nested scope count, depth limiting), and complexity scores.

## Roadmap

### Git Integration

- **Commit-by-commit traversal** — Built-in git history walker that processes repositories from first commit to HEAD.
- **Incremental processing** — Only changed files are analyzed, determined via git diffs. No redundant work.
- **Author tracking** — Records commit author(s) for each change to enable contributor-level metrics.

### Language Support

Target: **17 languages** across all major ecosystems.

| Language | Ecosystem |
|----------|-----------|
| JavaScript/TypeScript | Web |
| Python | General purpose |
| Java | Enterprise |
| C | Systems |
| C++ | Systems, games |
| C# | .NET, Unity |
| Go | Cloud, DevOps |
| Rust | Systems |
| PHP | Web |
| Ruby | Web |
| Swift | Apple |
| Kotlin | Android, JVM |
| Dart | Flutter mobile |
| Lua | Game scripting |
| Objective-C | Legacy Apple |
| Perl | Legacy enterprise |
| Zig | Modern systems |

**Current:** TypeScript/JavaScript only.

### Symbol Tracking

- **Declaration/reference tracking** — Track where symbols are declared and all their references.
- **Hoisting support** — Handle language-specific hoisting rules (e.g., JavaScript `var`, function declarations).
- **Scope variants** — Distinguish local vs global scope per language semantics.

### Import Analysis

- **Import tracking with counts** — Track all imports and how many times each is used.
- **Absolute path resolution** — Resolve relative imports to absolute paths.
- **Config file support** — Parse and merge language-specific config files (e.g., `tsconfig.json`, `pyproject.toml`) for path alias resolution.

### Server Integration

This library is designed for use in a **Rust (Axum) server**:

- **Thread-per-analysis** — The `analyze` function is called per-thread. The server distributes file analysis across multiple threads.
- **Batch inserts** — Processing threads collect results and batch-insert into ClickHouse. No per-file connections.
- **Graceful error handling** — All errors are handled explicitly. The library **cannot crash the server**. Every failure path returns a proper error type.
- **Pluggable logging** — Logging hooks integrate with the server's logging infrastructure.

### Storage

#### Data Model

Each analyzed file produces multiple **scopes** (functions, closures, methods, etc.), each with its own metrics. The storage layer maintains two levels of granularity:

- **Scope table (ClickHouse)** — Individual scope records with full metrics. This is the primary working set for queries.
- **File table (ClickHouse)** — Scope-aggregated metrics per file: min, max, and median values across all scopes in that file. Enables fast file-level queries without scanning all scopes.

#### Active vs Archived Data

On every new commit, the system determines which files are **irrelevant**:
- Files that no longer exist in the repository
- Files whose content changed (meaning their scopes changed)

Scopes from irrelevant files are moved to **Parquet** for archival. This ensures that:
- **Active queries only hit current data** — Search and aggregation over scopes always operates on files currently present in the repository from the perspective of the latest processed commit.
- **Historical data remains accessible** — Archived Parquet files preserve the full history for trend analysis and auditing.
- **ClickHouse stays lean** — The working set contains only relevant scopes, keeping queries fast.

#### Implementation

- **ClickHouse integration** — Initial implementation uses row-based inserts via standard client.
- **Column-oriented inserts** — Future optimization: feed data in column format mirroring the storage layout.
- **Batch collection** — Worker threads collect results; dedicated writer threads batch-insert to avoid connection proliferation.
- **Parquet archival** — Historical data archived in Parquet format for cold storage.

## Philosophy

### Raw Metrics Only

This library produces **raw metrics only**:
- Argument count, binding count, branch count, call count, etc.
- Line counts, depth, nesting levels

**Derived metrics are NOT computed here:**
- Cyclomatic complexity
- Maintainability index
- Cognitive complexity

These are computed by **ClickHouse** at query time. This provides flexibility to experiment with formulas without re-processing data.

### Compile-Time Optimization

This library will eventually be **open-sourced**, but extensibility is explicitly a non-goal:

- **No runtime extensibility** — Languages are defined at compile time.
- **Maximum compile-time optimization** — Code generation, inlining, and static dispatch everywhere.
- **Performance over flexibility** — Every design decision favors speed.

### Reliability

- **Thorough testing** — Comprehensive test coverage for all languages and edge cases.
- **Graceful degradation** — Malformed input returns errors, never panics.
- **Explicit error handling** — All error paths are typed and documented.

## Usage

Add to your `Cargo.toml`:

```toml
[dependencies]
quickscope = "0.1"
```

### Built-in TypeScript support

```rust
use quickscope::{Walker, TypeScript};

fn main() {
    let source = r#"
function greet(name) {
    const msg = "hello";
    return msg + name;
}

const add = (a, b) => a + b;
"#;

    let mut walker = Walker::new();
    walker.add(TypeScript);

    let scopes = walker.walk(".ts", source).unwrap();

    for (loc, metrics, score) in &scopes {
        println!(
            "rows {}-{}: own {:.2}, max {:.2}, {} args, {} bindings",
            loc.start_row, loc.end_row, score.own, score.max,
            metrics.argument_count, metrics.binding_count,
        );
    }
}
```

The Walker owns parsers and routes calls by extension. Multiple `walk` calls reuse the same parser instances. TypeScript supports `.ts`, `.mts`, `.cts`, and `.tsx` extensions.

### Defining a custom language

Implement the `Language` trait directly:

```rust
use quickscope::{Context, Language, ParserMapping, Walker};
use tree_sitter::Node;

pub struct Rust;

impl Language for Rust {
    fn mappings() -> Vec<ParserMapping> {
        vec![ParserMapping {
            extensions: &[".rs"],
            language: tree_sitter_rust::LANGUAGE_RUST.into(),
        }]
    }

    fn visit(node: Node, ctx: &mut Context) {
        match node.kind() {
            "function_item" => {
                ctx.increment_binding();
                ctx.create_scope(node);
            }
            "closure_expression" => {
                ctx.create_scope(node);
            }
            "parameter" => {
                ctx.increment_argument();
                ctx.increment_binding();
            }
            "let_declaration" => {
                ctx.increment_binding();
            }
            _ => {}
        }
    }
}

fn main() {
    let mut walker = Walker::new();
    walker.add(Rust);
    let scopes = walker.walk(".rs", "fn main() { let x = 1; }").unwrap();
    assert_eq!(scopes.len(), 1);
}
```

## Project structure

```
quickscope/
├── Cargo.toml                          (workspace root)
└── quickscope/
    ├── Cargo.toml                      (main crate)
    ├── src/
    │   ├── lib.rs                      (core types, Walker, Language trait)
    │   └── languages/
    │       ├── mod.rs                  (re-exports all built-in languages)
    │       └── typescript/
    │           ├── mod.rs              (language definition)
    │           └── tests.rs            (language-specific tests)
    └── tests/
        └── integration.rs              (integration tests)
```

### Adding a built-in language

Built-in languages implement the `Language` trait. To add one:

1. Add the grammar crate to `quickscope/Cargo.toml` dependencies.

2. Create `quickscope/src/languages/<name>/mod.rs`:

```rust
pub struct Rust;

impl crate::Language for Rust {
    fn mappings() -> Vec<crate::ParserMapping> {
        vec![crate::ParserMapping {
            extensions: &[".rs"],
            language: tree_sitter_rust::LANGUAGE_RUST.into(),
        }]
    }

    fn visit(node: tree_sitter::Node, ctx: &mut crate::Context) {
        match node.kind() {
            "function_item" => {
                ctx.increment_binding();
                ctx.create_scope(node);
            }
            "closure_expression" => {
                ctx.create_scope(node);
            }
            _ => {}
        }
    }
}

#[cfg(test)]
mod tests;
```

3. Create `quickscope/src/languages/<name>/tests.rs` with language-specific tests.

4. Add `pub mod <name>;` and `pub use <name>::<Name>;` to `quickscope/src/languages/mod.rs`.

## API

| Type | Description |
|------|-------------|
| `Scopes` | Parallel arrays of `Location`, `Metrics`, and `Score` for all extracted lexical scopes |
| `Location` | Where a lexical scope lives — `start_row` and `end_row` (both `u16`) |
| `Score` | Complexity score — `own: f64` (direct complexity), `max: f64` (peak complexity in subtree including self) |
| `Metrics` | Metrics for a lexical scope — `argument_count: u8`, `depth: u8`, `is_async: bool`, `depth_limited: bool`, `binding_count: u8`, `return_count: u8`, `throw_count: u8`, `branch_count: u8`, `jump_count: u8`, `try_count: u8`, `call_count: u8`, `await_count: u8`, `nested_scope_count: u8`, `assignment_count: u8`, `total_lines: u16`, `code_lines: u16`, `comment_lines: u16`, `whitespace_lines: u16` |
| `Context` | Passed to language visitors — provides `create_scope`, `set_async`, `increment_argument`, `increment_binding`, `increment_return`, `increment_throw`, `increment_branch`, `increment_jump`, `increment_try`, `increment_call`, `increment_await`, `increment_assignment` |
| `Language` | Trait: `mappings() -> Vec<ParserMapping>`, `visit(node: Node, ctx: &mut Context)` |
| `ParserMapping` | Maps file extensions to a `tree_sitter::Language` |
| `Walker` | Owns parsers, routes `walk(ext, source)` by extension, returns `Option<Scopes>` |
| `TypeScript` | Built-in TypeScript language (`.ts`, `.mts`, `.cts`, `.tsx`) |

## Benchmarking

See [BENCHMARKING.md](./BENCHMARKING.md) for the full benchmarking system documentation.

Quick start:
```bash
./scripts/bench.sh --remote    # Run on RPI 5 (full perf counters)
./scripts/bench.sh             # Run in Docker (timing + memory)
./scripts/bench.sh --local     # Run locally
```

## Non-goals

- **Type analysis** — No type inference, type-aware callback detection, or generic resolution. The walker sees syntax, not semantics.
- **Semantic analysis** — No control flow graph construction, data flow analysis, or points-to analysis. We extract structural metrics only.
- **Cross-file analysis in single pass** — Dependency graphs are built by aggregating single-file results. Each file is analyzed in isolation.
- **Derived metrics** — No cyclomatic complexity, maintainability index, or cognitive complexity. These are computed by ClickHouse for flexibility.
- **Runtime extensibility** — No plugin system or dynamic language loading. All optimization happens at compile time.

## License

MIT
