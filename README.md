# Rust AI Guidelines

> A comprehensive, AI-optimized reference for writing idiomatic Rust code.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## Overview

This repository contains a curated collection of Rust best practices, idioms, patterns, and anti-patterns specifically structured for AI assistants (Claude, GPT, Copilot, etc.) to reference when generating, refactoring, or reviewing Rust code.

**Key Features:**

- **~150+ patterns** across 13 topic areas
- **Strength indicators** (MUST, SHOULD, CONSIDER, AVOID) for clear prioritization
- **Code examples** for every pattern (good and bad)
- **Anti-patterns section** — critical for preventing common AI-generated mistakes
- **Modular structure** — load only what you need to conserve context
- **Cross-referenced** — patterns link to related guidance

## Quick Start

### For Claude Code Users

1. Clone this repo into your project or as a skill:

   ```bash
   git clone https://github.com/oxur/rust-ai-guidelines.git
   ```

2. Reference in your `CLAUDE.md` or project instructions:

   ```markdown
   When writing Rust code, follow guidelines in `./rust-ai-guidelines/docs/`.
   Always check `11-anti-patterns.md` before writing new code.
   ```

### For Other AI Tools

Copy the relevant markdown files into your context or system prompt. Start with:

1. `11-anti-patterns.md` (what NOT to do)
2. `01-core-idioms.md` (essential patterns)
3. Topic-specific docs as needed

## Repository Structure

```
rust-ai-guidelines/
├── README.md                 # This file
├── LICENSE                   # MIT License
├── docs/                     # The guidelines
│   ├── README.md             # Document index
│   ├── 01-core-idioms.md     # Essential Rust idioms
│   ├── 02-api-design.md      # Public API design
│   ├── 03-error-handling.md  # Result, Option, error types
│   ├── 04-ownership-borrowing.md  # Lifetimes, borrowing
│   ├── 05-type-design.md     # Structs, enums, newtypes
│   ├── 06-traits.md          # Trait design patterns
│   ├── 07-concurrency-async.md    # Async, threading
│   ├── 08-performance.md     # Optimization patterns
│   ├── 09-unsafe-ffi.md      # Unsafe code, FFI
│   ├── 10-macros.md          # Declarative & proc macros
│   ├── 11-anti-patterns.md   # What NOT to do (critical!)
│   ├── 12-project-structure.md    # Crate organization
│   └── 13-documentation.md   # Doc comments, rustdoc
└── skills/                   # AI tool integrations
    └── claude-code/
        └── SKILL.md          # Claude Code skill definition
```

## Document Guide

| Document | Description | Priority |
|----------|-------------|----------|
| [01-core-idioms](docs/01-core-idioms.md) | Essential patterns for everyday Rust | 🔴 Critical |
| [02-api-design](docs/02-api-design.md) | Designing public APIs | 🔴 Critical |
| [03-error-handling](docs/03-error-handling.md) | Result, Option, thiserror, anyhow | 🔴 Critical |
| [04-ownership-borrowing](docs/04-ownership-borrowing.md) | Lifetimes, borrow checker strategies | 🔴 Critical |
| [05-type-design](docs/05-type-design.md) | Structs, enums, newtypes, generics | 🟡 Important |
| [06-traits](docs/06-traits.md) | Trait design and implementation | 🟡 Important |
| [07-concurrency-async](docs/07-concurrency-async.md) | Async/await, Send/Sync, threading | 🟡 Important |
| [08-performance](docs/08-performance.md) | Optimization without sacrificing clarity | 🟡 Important |
| [09-unsafe-ffi](docs/09-unsafe-ffi.md) | Safe wrappers around unsafe code | 🟠 Specialized |
| [10-macros](docs/10-macros.md) | macro_rules! and proc macros | 🟠 Specialized |
| [11-anti-patterns](docs/11-anti-patterns.md) | **What NOT to do** | 🔴 Critical |
| [12-project-structure](docs/12-project-structure.md) | Crate and module organization | 🟢 Reference |
| [13-documentation](docs/13-documentation.md) | Doc comments and rustdoc | 🟢 Reference |

## Usage Patterns

### For New Code

```
Load: 11-anti-patterns.md → 01-core-idioms.md → topic-specific docs
```

### For Refactoring

```
Load: 11-anti-patterns.md → scan for violations → apply fixes from relevant docs
```

### For Code Review

```
Load: 11-anti-patterns.md → check each pattern → report violations with IDs
```

### For API Design

```
Load: 02-api-design.md → 05-type-design.md → 03-error-handling.md
```

## Strength Indicators

Every pattern includes a strength indicator:

| Indicator | Meaning | Action |
|-----------|---------|--------|
| **MUST** | Required for correctness or safety | Always follow |
| **SHOULD** | Strong recommendation | Follow unless specific reason not to |
| **CONSIDER** | Good practice, context-dependent | Evaluate for your situation |
| **AVOID** | Anti-pattern | Do not use |

## Contributing

Contributions are welcome! Please:

1. Follow the existing pattern format
2. Include code examples (good AND bad where applicable)
3. Add appropriate strength indicators
4. Cross-reference related patterns
5. Update pattern counts in READMEs

## Sources & Acknowledgments

This reference synthesizes and builds upon the excellent work of many in the Rust community:

### Primary Sources

| Source | Description | License |
|--------|-------------|---------|
| [cheats.rs](https://cheats.rs/) | Comprehensive Rust cheat sheet by Ralf Biedert | CC BY-SA 4.0 |
| [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) | Unofficial Rust design patterns book | MPL-2.0 |
| [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) | Official Rust API design guidelines | MIT/Apache-2.0 |
| [Clippy Documentation](https://doc.rust-lang.org/clippy/) | Rust linter documentation | MIT/Apache-2.0 |
| [The Rust Style Guide](https://doc.rust-lang.org/nightly/style-guide/) | Official Rust formatting guide | MIT/Apache-2.0 |
| [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/) | Official async Rust book | MIT/Apache-2.0 |
| [The Little Book of Rust Macros](https://veykril.github.io/tlborm/) | Comprehensive macro guide | CC BY-SA 4.0 |

### Additional References

- [The Rust Programming Language](https://doc.rust-lang.org/book/) (The Book)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) (unsafe Rust)
- [Effective Rust](https://www.lurklurk.org/effective-rust/) by David Drysdale
- [Rust for Rustaceans](https://rust-for-rustaceans.com/) by Jon Gjengset

### Tools Used in Creation

This reference was created through a collaboration between human curation and AI synthesis:

- **Claude** (Anthropic) — Document synthesis, pattern extraction, and formatting
- **Claude Code** — PDF processing and cross-referencing

## Version History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2024-12-31 | Initial release |

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

The content synthesizes information from multiple sources with various licenses (see Sources above). Original pattern descriptions and code examples in this repository are MIT licensed. When in doubt, refer to the original sources for authoritative guidance.

---

## Star History

If you find this useful, please ⭐ the repo!

## Related Projects

- [rust-clippy](https://github.com/rust-lang/rust-clippy) — The official Rust linter
- [rustfmt](https://github.com/rust-lang/rustfmt) — The official Rust formatter
- [cargo-deny](https://github.com/EmbarkStudios/cargo-deny) — Cargo plugin for linting dependencies

---

*Built with ❤️ for the Rust community and AI-assisted development.*
