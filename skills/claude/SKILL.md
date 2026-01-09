---
name: rust-guidelines
description: |
  Comprehensive Rust best practices, idioms, and anti-patterns.
  Use when: writing new Rust code, refactoring existing Rust,
  reviewing Rust for issues, debugging ownership/lifetime errors,
  designing Rust APIs, building CLI tools, managing Cargo projects,
  or answering Rust design questions.
---

# Rust Coding Guidelines Skill

## Overview

This skill provides access to comprehensive Rust guidelines optimized for AI code generation. The guidelines cover idioms, patterns, anti-patterns, and best practices across all major Rust topics, including CLI development and Cargo mastery.

## When to Use This Skill

Activate this skill when the task involves:

- Writing new Rust code
- Refactoring existing Rust code
- Reviewing Rust code for issues
- Debugging borrow checker or lifetime errors
- Designing public APIs in Rust
- Choosing between Rust patterns (e.g., enum vs struct, async vs sync)
- Understanding Rust best practices
- **Building command-line applications**
- **Managing Cargo projects and workspaces**
- **Publishing crates to crates.io**
- **Creating cargo plugins or subcommands**

## Document Locations

All guideline documents are in: `[REPO_ROOT]/guides/`

**Collections:**
- Core guides: `guides/01-*.md` through `guides/13-*.md`
- CLI Tools: `guides/14-cli-tools/` (9 focused sections)
- Cargo Mastery: `guides/15-cargo/` (6 specialized guides)

## Document Selection Guide

Load documents based on the task:

| Task | Load These Documents |
|------|---------------------|
| **Any Rust code** | `11-anti-patterns.md` (always load first) |
| **New code** | `01-core-idioms.md`, `11-anti-patterns.md` |
| **API design** | `02-api-design.md`, `05-type-design.md`, `06-traits.md` |
| **Error handling** | `03-error-handling.md` |
| **Ownership/lifetime issues** | `04-ownership-borrowing.md` |
| **Async code** | `07-concurrency-async.md` |
| **Performance optimization** | `08-performance.md` |
| **Refactoring** | `11-anti-patterns.md`, then topic-specific |
| **Code review** | `11-anti-patterns.md` (scan all patterns) |
| **FFI / unsafe** | `09-unsafe-ffi.md` |
| **Macros** | `10-macros.md` |
| **Project setup** | `12-project-structure.md` |
| **Documentation** | `13-documentation.md` |
| **CLI application** | `14-cli-tools/README.md`, then section-specific |
| **Cargo/dependencies** | `15-cargo/01-cargo-basics.md` |
| **Build scripts/features** | `15-cargo/02-cargo-build-system.md` |
| **Cargo plugin** | `15-cargo/03-cargo-plugins.md` |
| **Publishing crate** | `15-cargo/04-cargo-publishing.md` |
| **Cargo configuration** | `15-cargo/05-cargo-configuration.md` |
| **CI/CD or build optimization** | `15-cargo/06-cargo-advanced.md` |

### CLI Tools - When to Load Specific Sections

| Task | Load These Sections |
|------|---------------------|
| **Starting CLI project** | `14-cli-tools/01-project-setup.md` |
| **Argument parsing** | `14-cli-tools/02-argument-parsing.md` |
| **CLI error handling** | `14-cli-tools/03-error-handling.md` |
| **Output formatting** | `14-cli-tools/04-output-and-ux.md` |
| **Config files** | `14-cli-tools/05-configuration.md` |
| **Testing CLI** | `14-cli-tools/06-testing.md` |
| **Distributing CLI** | `14-cli-tools/07-distribution.md` |
| **Shell completions/signals** | `14-cli-tools/08-advanced-topics.md` |
| **Avoiding CLI pitfalls** | `14-cli-tools/09-common-pitfalls.md` |

### Cargo - When to Load Specific Guides

| Task | Load These Guides |
|------|---------------------|
| **Package creation** | `15-cargo/01-cargo-basics.md` (CG-B-01, CG-B-02, CG-B-03) |
| **Dependencies** | `15-cargo/01-cargo-basics.md` (CG-B-07, CG-B-08, CG-B-09) |
| **Workspaces** | `15-cargo/01-cargo-basics.md` (CG-B-10, CG-B-11) |
| **Features** | `15-cargo/02-cargo-build-system.md` (CG-BS-01 to CG-BS-05) |
| **Build scripts** | `15-cargo/02-cargo-build-system.md` (CG-BS-08 to CG-BS-11) |
| **Custom cargo commands** | `15-cargo/03-cargo-plugins.md` (all patterns) |
| **Publishing** | `15-cargo/04-cargo-publishing.md` (CG-PUB-01, CG-PUB-02) |
| **SemVer** | `15-cargo/04-cargo-publishing.md` (CG-PUB-03, CG-PUB-04) |
| **Config files** | `15-cargo/05-cargo-configuration.md` (all patterns) |
| **Build optimization** | `15-cargo/06-cargo-advanced.md` (CG-A-01, CG-A-02) |
| **CI/CD** | `15-cargo/06-cargo-advanced.md` (CG-A-03, CG-A-08) |

## Workflow

### For Writing New Code

1. **Load anti-patterns first**: Read `11-anti-patterns.md` to know what to avoid
2. **Load core idioms**: Read `01-core-idioms.md` for standard patterns
3. **Load topic-specific docs**: Based on what you're building
4. **Write code**: Following the guidelines
5. **Self-review**: Check against anti-patterns before finishing

### For Building a CLI Application

1. **Start with overview**: Read `14-cli-tools/README.md` for complete picture
2. **Setup project**: Read `14-cli-tools/01-project-setup.md`
3. **Implement arguments**: Read `14-cli-tools/02-argument-parsing.md`
4. **Handle errors properly**: Read `14-cli-tools/03-error-handling.md`
5. **Polish UX**: Read `14-cli-tools/04-output-and-ux.md`
6. **Add tests**: Read `14-cli-tools/06-testing.md`

### For Managing Cargo Projects

1. **Start with basics**: Read `15-cargo/README.md` for navigation
2. **Load specific guide**: Based on task (see table above)
3. **Check anti-patterns**: Review relevant patterns
4. **Apply best practices**: Follow pattern recommendations

### For Refactoring

1. **Load anti-patterns**: Read `11-anti-patterns.md`
2. **Scan existing code**: Identify violations (note pattern IDs like AP-08)
3. **Load relevant docs**: For patterns you need to apply
4. **Refactor systematically**: Fix one pattern at a time
5. **Document changes**: Reference pattern IDs in commit messages

### For Code Review

1. **Load anti-patterns**: Read `11-anti-patterns.md`
2. **Check each pattern**: Go through AP-01 to AP-80
3. **Load topic docs**: Based on code content (async, unsafe, CLI, etc.)
4. **Report findings**: Use pattern IDs for clarity

## Critical Rules (Always Apply)

These rules should be followed in ALL Rust code without needing to load documents:

### Parameters

```rust
// ❌ AVOID
fn process(data: &String, items: &Vec<i32>)

// ✅ PREFER
fn process(data: &str, items: &[i32])
```

### Derives

```rust
// ✅ Most types should have at minimum:
#[derive(Debug, Clone, PartialEq)]
struct MyType { /* ... */ }
```

### Error Handling

```rust
// ❌ AVOID in library code
let value = something.unwrap();

// ✅ PREFER
let value = something?;
// or
let value = something.ok_or(MyError::NotFound)?;
```

### Construction

```rust
// ✅ Provide both:
impl MyType {
    pub fn new(/* required args */) -> Self { /* ... */ }
}

impl Default for MyType {
    fn default() -> Self { /* ... */ }
}
```

### Async

```rust
// ❌ NEVER block in async
async fn bad() {
    std::fs::read_to_string("file.txt"); // BLOCKS!
}

// ✅ Use async I/O
async fn good() {
    tokio::fs::read_to_string("file.txt").await;
}
```

### CLI Applications

```rust
// ❌ AVOID - direct argument access
use std::env;
let args: Vec<String> = env::args().collect();

// ✅ PREFER - use clap derive
use clap::Parser;

#[derive(Parser)]
struct Cli {
    #[arg(short, long)]
    name: String,
}

fn main() {
    let cli = Cli::parse();
}
```

## Pattern ID Reference

Each document uses a prefix for pattern IDs:

### Core Guides

| Prefix | Document |
|--------|----------|
| ID-XX | 01-core-idioms.md |
| API-XX | 02-api-design.md |
| EH-XX | 03-error-handling.md |
| OB-XX | 04-ownership-borrowing.md |
| TD-XX | 05-type-design.md |
| TR-XX | 06-traits.md |
| CA-XX | 07-concurrency-async.md |
| PF-XX | 08-performance.md |
| US-XX | 09-unsafe-ffi.md |
| MC-XX | 10-macros.md |
| AP-XX | 11-anti-patterns.md |
| PS-XX | 12-project-structure.md |
| DC-XX | 13-documentation.md |

### CLI Tools Collection

| Prefix | Document |
|--------|----------|
| CLI-XX | 14-cli-tools/*.md (52 patterns across 9 sections) |

Specific ranges:
- CLI-01 to CLI-04: Project Setup
- CLI-05 to CLI-15: Argument Parsing
- CLI-16 to CLI-20: Error Handling
- CLI-21 to CLI-28: Output and UX
- CLI-29 to CLI-33: Configuration
- CLI-34 to CLI-38: Testing
- CLI-39 to CLI-42: Distribution
- CLI-43 to CLI-48: Advanced Topics
- CLI-49 to CLI-52: Common Pitfalls

### Cargo Mastery Collection

| Prefix | Document |
|--------|----------|
| CG-B-XX | 15-cargo/01-cargo-basics.md |
| CG-BS-XX | 15-cargo/02-cargo-build-system.md |
| CG-P-XX | 15-cargo/03-cargo-plugins.md |
| CG-PUB-XX | 15-cargo/04-cargo-publishing.md |
| CG-CF-XX | 15-cargo/05-cargo-configuration.md |
| CG-A-XX | 15-cargo/06-cargo-advanced.md |

## Strength Indicators

When reading guidelines, note the strength:

| Indicator | Meaning | Action |
|-----------|---------|--------|
| **MUST** | Required for correctness/safety | Always follow |
| **SHOULD** | Strong recommendation | Follow unless specific reason not to |
| **CONSIDER** | Context-dependent | Evaluate for situation |
| **AVOID** | Anti-pattern | Do not use |

## Example Usage

### Task: "Write a function to parse configuration from a file"

1. Load: `11-anti-patterns.md`, `01-core-idioms.md`, `03-error-handling.md`
2. Apply:
   - EH-03: Define custom error type
   - EH-02: Use `?` for propagation
   - ID-02: Provide `new()` constructor
   - AP-06: Don't use `unwrap()`
   - AP-08: Accept `impl AsRef<Path>` not `&PathBuf`

### Task: "Review this async code for issues"

1. Load: `11-anti-patterns.md`, `07-concurrency-async.md`
2. Check:
   - AP-18: No sync I/O in async functions
   - CA-03: No blocking in async code
   - CA-01: Proper Send/Sync bounds
   - CA-07: Cancellation safety

### Task: "Build a CLI tool that processes files"

1. Load: `14-cli-tools/README.md` for overview
2. Load: `14-cli-tools/01-project-setup.md`, `14-cli-tools/02-argument-parsing.md`
3. Apply:
   - CLI-01: Binary crate structure
   - CLI-02: Cargo.toml dependencies (clap with derive)
   - CLI-05: Use clap derive for all arguments
   - CLI-06: Positional arguments for files
   - CLI-16: Exit codes (0 for success)
   - CLI-17: User-friendly error messages

### Task: "Create a cargo subcommand plugin"

1. Load: `15-cargo/03-cargo-plugins.md`
2. Apply:
   - CG-P-01: Binary name must be `cargo-{name}`
   - CG-P-02: Handle `--help` properly
   - CG-P-03: Use `cargo_metadata` for workspace info
   - CG-P-06: Error handling for cargo plugins
   - CG-P-12: Distribution and installation

### Task: "Prepare crate for publishing"

1. Load: `15-cargo/04-cargo-publishing.md`
2. Check:
   - CG-PUB-01: Complete Cargo.toml metadata
   - CG-PUB-02: Pre-publish checklist
   - CG-PUB-12: crates.io requirements
   - Also review: `02-api-design.md`, `13-documentation.md`

### Task: "Optimize build times in CI"

1. Load: `15-cargo/06-cargo-advanced.md`
2. Apply:
   - CG-A-01: Reduce debug info
   - CG-A-02: Use faster linker (mold/lld)
   - CG-A-03: CI caching strategies
   - CG-A-04: Incremental compilation settings
   - CG-A-08: CI pipeline design

## Integration Notes

- Documents are markdown with code blocks using `rust` syntax
- Code examples show both ❌ BAD and ✅ GOOD patterns
- Cross-references use format `[document.md](document.md#section)`
- Clippy lints are referenced as `clippy::lint_name`
- CLI and Cargo collections have their own READMEs with decision trees
- Pattern IDs are consistent: PREFIX-NUMBER format

## Quick Reference for Common Tasks

| I want to... | Read this |
|--------------|-----------|
| Write any Rust code | Start with `11-anti-patterns.md` |
| Build a CLI app | `14-cli-tools/README.md` → section-specific |
| Parse CLI arguments | `14-cli-tools/02-argument-parsing.md` |
| Create a new project | `15-cargo/01-cargo-basics.md` (CG-B-01 to CG-B-03) |
| Add dependencies | `15-cargo/01-cargo-basics.md` (CG-B-07 to CG-B-09) |
| Set up workspace | `15-cargo/01-cargo-basics.md` (CG-B-10, CG-B-11) |
| Configure features | `15-cargo/02-cargo-build-system.md` (CG-BS-01 to CG-BS-05) |
| Write build.rs | `15-cargo/02-cargo-build-system.md` (CG-BS-08 to CG-BS-11) |
| Create cargo plugin | `15-cargo/03-cargo-plugins.md` |
| Publish to crates.io | `15-cargo/04-cargo-publishing.md` |
| Optimize builds | `15-cargo/06-cargo-advanced.md` (CG-A-01, CG-A-02) |
| Design an API | `02-api-design.md` |
| Handle errors | `03-error-handling.md` and maybe `14-cli-tools/03-error-handling.md` |
| Fix ownership issues | `04-ownership-borrowing.md` |
| Write async code | `07-concurrency-async.md` |
| Optimize performance | `08-performance.md` |
| Use unsafe/FFI | `09-unsafe-ffi.md` |
| Write macros | `10-macros.md` |
