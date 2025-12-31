---
name: rust-guidelines
description: |
  Comprehensive Rust best practices, idioms, and anti-patterns.
  Use when: writing new Rust code, refactoring existing Rust,
  reviewing Rust for issues, debugging ownership/lifetime errors,
  designing Rust APIs, or answering Rust design questions.
---

# Rust Coding Guidelines Skill

## Overview

This skill provides access to comprehensive Rust guidelines optimized for AI code generation. The guidelines cover idioms, patterns, anti-patterns, and best practices across all major Rust topics.

## When to Use This Skill

Activate this skill when the task involves:

- Writing new Rust code
- Refactoring existing Rust code
- Reviewing Rust code for issues
- Debugging borrow checker or lifetime errors
- Designing public APIs in Rust
- Choosing between Rust patterns (e.g., enum vs struct, async vs sync)
- Understanding Rust best practices

## Document Locations

All guideline documents are in: `[REPO_ROOT]/docs/`

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

## Workflow

### For Writing New Code

1. **Load anti-patterns first**: Read `11-anti-patterns.md` to know what to avoid
2. **Load core idioms**: Read `01-core-idioms.md` for standard patterns
3. **Load topic-specific docs**: Based on what you're building
4. **Write code**: Following the guidelines
5. **Self-review**: Check against anti-patterns before finishing

### For Refactoring

1. **Load anti-patterns**: Read `11-anti-patterns.md`
2. **Scan existing code**: Identify violations (note pattern IDs like AP-08)
3. **Load relevant docs**: For patterns you need to apply
4. **Refactor systematically**: Fix one pattern at a time
5. **Document changes**: Reference pattern IDs in commit messages

### For Code Review

1. **Load anti-patterns**: Read `11-anti-patterns.md`
2. **Check each pattern**: Go through AP-01 to AP-20
3. **Load topic docs**: Based on code content (async, unsafe, etc.)
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

## Pattern ID Reference

Each document uses a prefix for pattern IDs:

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

## Integration Notes

- Documents are markdown with code blocks using `rust` syntax
- Code examples show both ❌ BAD and ✅ GOOD patterns
- Cross-references use format `[document.md](document.md#section)`
- Clippy lints are referenced as `clippy::lint_name`
