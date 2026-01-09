# Rust Best Practices Guidelines

> Comprehensive collection of Rust idioms, patterns, and best practices for writing idiomatic, safe, and performant code.

**Last Updated**: 2026-01-09
**Total Patterns**: 561 across 15 collections

---

## Document Index

This collection is organized into 15 focused areas covering all aspects of Rust development:

| Document | Pattern Count | Pattern ID Range | Description |
|----------|--------------|------------------|-------------|
| [01. Core Idioms](./01-core-idioms.md) | 42 | ID-01 to ID-42 | Fundamental Rust patterns and idioms |
| [02. API Design](./02-api-design.md) | 59 | API-01 to API-59 | Designing ergonomic and idiomatic APIs |
| [03. Error Handling](./03-error-handling.md) | 32 | EH-01 to EH-32 | Robust error handling strategies |
| [04. Ownership & Borrowing](./04-ownership-borrowing.md) | 19 | OB-01 to OB-19 | Mastering Rust's ownership system |
| [05. Type Design](./05-type-design.md) | 27 | TD-01 to TD-27 | Effective type system usage |
| [06. Traits](./06-traits.md) | 26 | TR-01 to TR-26 | Trait design and implementation |
| [07. Concurrency & Async](./07-concurrency-async.md) | 20 | CA-01 to CA-20 | Safe concurrent and asynchronous code |
| [08. Performance](./08-performance.md) | 22 | PF-01 to PF-22 | Writing high-performance Rust |
| [09. Unsafe & FFI](./09-unsafe-ffi.md) | 22 | US-01 to US-22 | Working with unsafe code and FFI |
| [10. Macros](./10-macros.md) | 20 | MC-01 to MC-20 | Effective macro design and usage |
| [11. Anti-patterns](./11-anti-patterns.md) | 80 | AP-01 to AP-80 | Common pitfalls and how to avoid them |
| [12. Project Structure](./12-project-structure.md) | 31 | PS-01 to PS-31 | Organizing Rust projects |
| [13. Documentation](./13-documentation.md) | 35 | DC-01 to DC-35 | Writing excellent documentation |
| [14. CLI Tools](./14-cli-tools/) | 52 | CLI-01 to CLI-52 | Building command-line applications |
| [15. Cargo Mastery](./15-cargo/) | 74 | CG-* (6 guides) | Package management, builds, publishing, plugins |

---

## Cargo Mastery Guide Collection

The **15-cargo/** directory contains 6 specialized guides covering all aspects of Cargo:

| Guide | Patterns | Pattern Range | Use When |
|-------|----------|---------------|----------|
| [📦 Cargo Basics](./15-cargo/01-cargo-basics.md) | 12 | CG-B-01 to CG-B-12 | Creating packages, managing dependencies, workspaces |
| [⚙️ Build System](./15-cargo/02-cargo-build-system.md) | 12 | CG-BS-01 to CG-BS-12 | Features, profiles, build scripts, optimization |
| [🔌 Cargo Plugins](./15-cargo/03-cargo-plugins.md) | 12 | CG-P-01 to CG-P-12 | Building custom cargo subcommands and tools |
| [📤 Publishing](./15-cargo/04-cargo-publishing.md) | 13 | CG-PUB-01 to CG-PUB-13 | Publishing to crates.io, SemVer, versioning |
| [⚙️ Configuration](./15-cargo/05-cargo-configuration.md) | 12 | CG-CF-01 to CG-CF-12 | .cargo/config.toml, environment variables |
| [🚀 Advanced](./15-cargo/06-cargo-advanced.md) | 13 | CG-A-01 to CG-A-13 | CI/CD, optimization, unstable features |

**Quick Navigation**: See [15-cargo/README.md](./15-cargo/README.md) for a comprehensive decision tree and troubleshooting guide.

### When to Use Cargo Guides

Load cargo guides when working with:

- **Package creation**: CG-B-01, CG-B-02, CG-B-03 (cargo new, init, project setup)
- **Dependencies**: CG-B-07, CG-B-08, CG-B-09 (version specifications, workspace deps)
- **Workspaces**: CG-B-10, CG-B-11 (multi-crate projects, shared dependencies)
- **Features**: CG-BS-01 through CG-BS-05 (feature flags, optional deps)
- **Build scripts**: CG-BS-08 through CG-BS-11 (build.rs, native libraries)
- **Custom cargo commands**: CG-P-01 through CG-P-12 (plugin development)
- **Publishing**: CG-PUB-01, CG-PUB-02, CG-PUB-12 (preparing for crates.io)
- **SemVer compliance**: CG-PUB-03, CG-PUB-04 (versioning, breaking changes)
- **Build optimization**: CG-BS-06, CG-A-01, CG-A-02 (faster compilation)
- **CI/CD**: CG-CF-02, CG-A-03, CG-A-08 (continuous integration setup)

---

## CLI Tools Guide Collection

The **14-cli-tools/** directory contains comprehensive guidance for building command-line applications:

| Section | Patterns | Pattern Range | Topics Covered |
|---------|----------|---------------|----------------|
| [🚀 Project Setup](./14-cli-tools/01-project-setup.md) | 4 | CLI-01 to CLI-04 | Binary structure, Cargo.toml, lib/bin separation |
| [⚙️ Argument Parsing](./14-cli-tools/02-argument-parsing.md) | 11 | CLI-05 to CLI-15 | Clap derive, flags, subcommands, validation |
| [❌ Error Handling](./14-cli-tools/03-error-handling.md) | 5 | CLI-16 to CLI-20 | Exit codes, error messages, stderr |
| [🎨 Output & UX](./14-cli-tools/04-output-and-ux.md) | 8 | CLI-21 to CLI-28 | Human/machine output, progress, colors |
| [⚙️ Configuration](./14-cli-tools/05-configuration.md) | 5 | CLI-29 to CLI-33 | Config files, precedence, XDG directories |
| [🧪 Testing](./14-cli-tools/06-testing.md) | 5 | CLI-34 to CLI-38 | assert_cmd, integration tests, snapshots |
| [📦 Distribution](./14-cli-tools/07-distribution.md) | 4 | CLI-39 to CLI-42 | Binary size, cross-compilation, packaging |
| [🔧 Advanced Topics](./14-cli-tools/08-advanced-topics.md) | 6 | CLI-43 to CLI-48 | Signals, completions, plugins, async |
| [⚠️ Common Pitfalls](./14-cli-tools/09-common-pitfalls.md) | 4 | CLI-49 to CLI-52 | Anti-patterns to avoid |

**Quick Navigation**: See [14-cli-tools/README.md](./14-cli-tools/README.md) for complete pattern index and examples.

### When to Use CLI Tools Guides

Load CLI guides when building command-line applications:

- **Starting a CLI project**: CLI-01, CLI-02, CLI-03, CLI-04 (project structure, dependencies)
- **Parsing arguments**: CLI-05, CLI-06, CLI-07, CLI-08 (clap derive, flags, options)
- **Subcommands**: CLI-10 (subcommand patterns with enums)
- **Help and version**: CLI-14, CLI-15 (documentation, version info)
- **Error handling**: CLI-16, CLI-17, CLI-20 (exit codes, error messages, stderr)
- **User experience**: CLI-21, CLI-22, CLI-23 (output formats, progress, colors)
- **Configuration**: CLI-29, CLI-30, CLI-31 (config files, locations, precedence)
- **Testing CLIs**: CLI-34, CLI-35, CLI-37 (assert_cmd, integration tests)
- **Distribution**: CLI-39, CLI-40, CLI-42 (binary optimization, releases)
- **Advanced features**: CLI-43, CLI-44, CLI-48 (signals, completions, piping)

---

## Pattern Strength Indicators

Each pattern includes a strength indicator that guides how strictly it should be followed:

- **MUST**: Critical patterns that should always be followed (safety, correctness, or strong conventions)
- **SHOULD**: Strong recommendations for idiomatic code (best practices)
- **CONSIDER**: Useful patterns to consider based on context (trade-offs exist)
- **AVOID**: Anti-patterns that should be avoided (common mistakes)

---

## Pattern Format

Each pattern follows a consistent structure:

```markdown
## XX-NN: Pattern Name

**Strength**: MUST | SHOULD | CONSIDER | AVOID

**Summary**: One-line description of the pattern.

[Code examples demonstrating the pattern]

**Rationale**: Explanation of why this pattern matters.

**See also**: Cross-references to related patterns
```

---

## How to Use This Guide

### For New Rustaceans

Start with these foundational documents:

1. **01-core-idioms.md** - Learn fundamental Rust patterns
2. **04-ownership-borrowing.md** - Master Rust's unique ownership system
3. **03-error-handling.md** - Handle errors the Rust way
4. **14-cli-tools/01-project-setup.md** - Build your first CLI tool
5. **15-cargo/01-cargo-basics.md** - Create and manage Rust projects
6. **11-anti-patterns.md** - Avoid common mistakes

### For Intermediate Developers

Focus on these areas to level up:

1. **02-api-design.md** - Design better libraries and interfaces
2. **05-type-design.md** - Leverage the type system effectively
3. **06-traits.md** - Master trait-based design
4. **08-performance.md** - Optimize your code
5. **15-cargo/02-cargo-build-system.md** - Master features and build configuration

### For Advanced Practitioners

Explore specialized topics:

1. **07-concurrency-async.md** - Advanced async patterns
2. **09-unsafe-ffi.md** - Safe unsafe code and FFI bindings
3. **10-macros.md** - Metaprogramming with macros
4. **12-project-structure.md** - Architect large projects
5. **15-cargo/06-cargo-advanced.md** - Build optimization and CI/CD

### For Library Authors

Essential guides for publishing crates:

1. **02-api-design.md** - Design ergonomic public APIs
2. **13-documentation.md** - Document your crate effectively
3. **15-cargo/04-cargo-publishing.md** - Publish to crates.io
4. **15-cargo/03-cargo-plugins.md** - Build cargo extensions

### For CLI Tool Developers

Complete guide to building command-line applications:

1. **14-cli-tools/01-project-setup.md** - Set up CLI project structure
2. **14-cli-tools/02-argument-parsing.md** - Parse arguments with clap
3. **14-cli-tools/03-error-handling.md** - Handle CLI errors properly
4. **14-cli-tools/04-output-and-ux.md** - Create polished user experience
5. **14-cli-tools/06-testing.md** - Test CLI applications

### For All Developers

Essential reference materials:

1. **13-documentation.md** - Document your code effectively
2. **11-anti-patterns.md** - Comprehensive anti-pattern catalog (80 patterns!)
3. **15-cargo/README.md** - Quick reference for cargo workflows

---

## Quick Stats

- **Total Patterns**: 561
- **Document Collections**: 15 (13 single docs + 2 multi-guide collections)
- **Cargo Guide Patterns**: 74 across 6 specialized guides
- **CLI Tools Patterns**: 52 across 9 focused sections
- **Code Examples**: 620+ `rust` and `bash` code blocks
- **Pattern Categories**:
  - Core Idioms & Patterns: 42
  - API Design: 59
  - Error Handling: 32
  - Type System: 46 (Type Design + Traits)
  - Concurrency: 20
  - Performance: 22
  - Advanced Topics: 42 (Unsafe/FFI + Macros)
  - Anti-patterns: 80
  - Project Organization: 66 (Structure + Documentation)
  - **Cargo & Build System: 74** (Cargo Mastery collection)
  - **CLI Development: 52** (CLI Tools collection)

---

## Sources

This collection merges and consolidates patterns from multiple authoritative Rust resources:

- **Rust API Guidelines**: Official API design guidelines
- **Rust Design Patterns**: Community-maintained pattern catalog
- **Rust Performance Book**: Performance best practices
- **The Cargo Book**: Official Cargo documentation and best practices
- **Command Line Applications in Rust**: CLI development guide (rust-cli.github.io)
- **Clap Documentation**: Argument parsing best practices (clap-rs)
- **Clippy Lints**: Static analysis recommendations
- **Rust RFC discussions**: Language evolution insights
- **Production Rust codebases**: Real-world patterns
- **crates.io ecosystem**: Library publishing and maintenance patterns

All patterns have been deduplicated, merged, and organized for maximum clarity and utility.

---

## Contributing

These guidelines represent current best practices as of 2026-01-09. The Rust ecosystem evolves rapidly:

- Patterns marked **MUST** are stable and unlikely to change
- Patterns marked **SHOULD** represent current consensus
- Patterns marked **CONSIDER** may have evolving trade-offs
- Patterns marked **AVOID** identify well-established anti-patterns

When Rust evolves (new language features, edition changes, ecosystem shifts), these patterns should be reviewed and updated accordingly.

---

## Cross-References

Patterns frequently reference each other across documents. Look for **See also** sections to explore related concepts. Common cross-cutting themes include:

- **Zero-cost abstractions**: TR-04, PF-02, PF-06
- **Memory safety**: OB-*, US-*, EH-*
- **API ergonomics**: API-*, TD-*, TR-*
- **Compile-time guarantees**: TD-*, TR-*, MC-*
- **Build optimization**: PF-*, CG-BS-06, CG-A-01, CG-A-02
- **Publishing workflow**: CG-PUB-*, API-*, DC-*
- **Project organization**: PS-*, CG-B-10, CG-B-11
- **CLI development**: CLI-*, EH-*, API-*
- **Error handling patterns**: EH-*, CLI-16 through CLI-20

---

## License

This is a compilation and synthesis of publicly available Rust best practices. Individual patterns may reference specific sources. Use freely for learning and reference.

---

*For the latest Rust language features and edition-specific guidance, always consult the official [Rust documentation](https://doc.rust-lang.org/).*
