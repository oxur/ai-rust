# Cargo Mastery Guides

Comprehensive best practices for Cargo, Rust's package manager. These guides provide actionable patterns for package management, builds, publishing, and configuration.

## 📚 Available Guides

| Guide | Topics Covered | When to Read |
|-------|---------------|--------------|
| **[01: Cargo Basics](./01-cargo-basics.md)** | Package creation, dependencies, workspaces, layouts | Start here if new to Cargo or creating a new project |
| **[02: Build System](./02-cargo-build-system.md)** | Features, profiles, build scripts, native libraries | When setting up features or optimizing builds |
| **[03: Cargo Plugins](./03-cargo-plugins.md)** | Custom subcommands, cargo metadata, plugin distribution | When building tools that extend cargo functionality |
| **[04: Publishing](./04-cargo-publishing.md)** | crates.io, SemVer, versioning, releases | Before publishing your first crate or making a release |
| **[05: Configuration](./05-cargo-configuration.md)** | .cargo/config.toml, environment variables, targets | When customizing Cargo for your project or machine |
| **[06: Advanced](./06-cargo-advanced.md)** | Build optimization, CI/CD, unstable features, diagnostics | When optimizing builds or setting up sophisticated workflows |

## 🎯 Quick Decision Tree

**I want to...**

### Create a new project
→ **[01: Cargo Basics](./01-cargo-basics.md)** - CG-B-01, CG-B-02, CG-B-03

### Add dependencies
→ **[01: Cargo Basics](./01-cargo-basics.md)** - CG-B-07, CG-B-08, CG-B-09

### Set up a workspace
→ **[01: Cargo Basics](./01-cargo-basics.md)** - CG-B-10, CG-B-11

### Make my crate configurable
→ **[02: Build System](./02-cargo-build-system.md)** - CG-BS-01 through CG-BS-05

### Optimize build times
→ **[02: Build System](./02-cargo-build-system.md)** - CG-BS-06, CG-BS-12  
→ **[06: Advanced](./06-cargo-advanced.md)** - CG-A-01, CG-A-02, CG-A-10

### Compile native code or link libraries
→ **[02: Build System](./02-cargo-build-system.md)** - CG-BS-08 through CG-BS-11

### Build a cargo plugin or subcommand
→ **[03: Cargo Plugins](./03-cargo-plugins.md)** - CG-P-01 through CG-P-12

### Publish to crates.io
→ **[04: Publishing](./04-cargo-publishing.md)** - CG-PUB-01, CG-PUB-02, CG-PUB-12

### Understand SemVer for Rust
→ **[04: Publishing](./04-cargo-publishing.md)** - CG-PUB-03, CG-PUB-04

### Make a breaking change
→ **[04: Publishing](./04-cargo-publishing.md)** - CG-PUB-04, CG-PUB-05, CG-PUB-07

### Configure Cargo for my project
→ **[05: Configuration](./05-cargo-configuration.md)** - CG-CF-01, CG-CF-03, CG-CF-04

### Set up cross-compilation
→ **[05: Configuration](./05-cargo-configuration.md)** - CG-CF-06, CG-CF-07

### Configure CI/CD
→ **[05: Configuration](./05-cargo-configuration.md)** - CG-CF-02, CG-CF-08, CG-CF-10  
→ **[06: Advanced](./06-cargo-advanced.md)** - CG-A-03, CG-A-08, CG-A-09

### Speed up my CI builds
→ **[06: Advanced](./06-cargo-advanced.md)** - CG-A-03, CG-A-04, CG-A-08

### Use unstable features safely
→ **[06: Advanced](./06-cargo-advanced.md)** - CG-A-06

### Manage dependencies strategically
→ **[06: Advanced](./06-cargo-advanced.md)** - CG-A-11, CG-A-12

## 📖 Pattern Naming Convention

All patterns follow the format: `[PREFIX]-[NUMBER]: [Pattern Name]`

### Prefixes

| Prefix | Guide | Focus |
|--------|-------|-------|
| `CG-B-XX` | Cargo Basics | Package management fundamentals |
| `CG-BS-XX` | Build System | Features, profiles, build scripts |
| `CG-P-XX` | Cargo Plugins | Custom subcommands and extensibility |
| `CG-PUB-XX` | Publishing | crates.io and versioning |
| `CG-CF-XX` | Configuration | Project and user configuration |
| `CG-A-XX` | Advanced | Optimization, CI/CD, diagnostics |

### Strength Indicators

Each pattern has a strength indicator:

- **MUST**: Required for correctness, safety, or ecosystem compatibility
- **SHOULD**: Strong recommendation, follow unless specific reason not to
- **CONSIDER**: Context-dependent, evaluate for your specific situation
- **AVOID**: Anti-pattern, don't do this

## 🚀 Quick Start Examples

### Creating a New Library

```bash
# Create the library
cargo new my-lib --lib

# Add dependencies
cd my-lib
cargo add serde --features derive
cargo add tokio --features full --optional

# Configure features
# Edit Cargo.toml:
[features]
default = []
async = ["dep:tokio"]
```

**Relevant patterns:** CG-B-01, CG-B-07, CG-BS-02, CG-BS-04

### Setting Up a Workspace

```bash
# Create workspace root
mkdir my-workspace && cd my-workspace

# Create Cargo.toml for workspace
cat > Cargo.toml << 'EOF'
[workspace]
members = ["app", "core", "utils"]

[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.35", features = ["full"] }
EOF

# Create member crates
cargo new app
cargo new core --lib
cargo new utils --lib
```

**Relevant patterns:** CG-B-10, CG-B-11

### Publishing Your First Crate

```bash
# 1. Complete metadata in Cargo.toml
# 2. Run pre-publish checks
cargo test
cargo clippy -- -D warnings
cargo doc --no-deps
cargo package --list

# 3. Dry run
cargo publish --dry-run

# 4. Actually publish
cargo publish
```

**Relevant patterns:** CG-PUB-01, CG-PUB-02, CG-PUB-12

### Optimizing Build Times

```toml
# .cargo/config.toml
[profile.dev]
debug = "line-tables-only"    # Minimal debug info

[profile.dev.package."*"]
opt-level = 2                 # Optimize dependencies
debug = false                 # No debug info for deps

[build]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]  # Fast linker
```

**Relevant patterns:** CG-BS-06, CG-CF-04

## 📝 Contributing

These guides are designed to be living documents. If you have:
- **Corrections**: Found an error or outdated information
- **Improvements**: Better examples or clearer explanations
- **New patterns**: Important patterns we're missing

Please provide feedback or contributions!

## 🔍 Common Issues and Solutions

### "My build is slow"
1. Check [CG-BS-06](./02-cargo-build-system.md#cg-bs-06) for dev profile optimization
2. Check [CG-BS-12](./02-cargo-build-system.md#cg-bs-12) for incremental compilation
3. Check [CG-A-01](./06-cargo-advanced.md#cg-a-01) for debug info reduction
4. Check [CG-A-02](./06-cargo-advanced.md#cg-a-02) for alternative linkers (mold, lld, zld)
5. Check [CG-A-10](./06-cargo-advanced.md#cg-a-10) to analyze build timings

### "cargo publish failed"
1. Check [CG-PUB-01](./04-cargo-publishing.md#cg-pub-01) for required metadata
2. Run `cargo publish --dry-run` to see what's wrong
3. Check [CG-PUB-12](./04-cargo-publishing.md#cg-pub-12) for crates.io requirements

### "I broke my users with a version update"
1. Review [CG-PUB-03](./04-cargo-publishing.md#cg-pub-03) for SemVer rules
2. Check [CG-PUB-04](./04-cargo-publishing.md#cg-pub-04) for subtle breaking changes
3. Consider yanking if severely broken: [CG-PUB-06](./04-cargo-publishing.md#cg-pub-06)

### "Cargo.lock is confusing"
1. Read [CG-B-05](./01-cargo-basics.md#cg-b-05) for Cargo.toml vs Cargo.lock
2. Read [CG-B-06](./01-cargo-basics.md#cg-b-06) for commit strategy

### "Features aren't working as expected"
1. Review [CG-BS-01](./02-cargo-build-system.md#cg-bs-01) for additive features principle
2. Check [CG-BS-02](./02-cargo-build-system.md#cg-bs-02) for optional dependencies
3. Check [CG-BS-03](./02-cargo-build-system.md#cg-bs-03) for naming conventions

### "My cargo plugin isn't working"
1. Check [CG-P-01](./03-cargo-plugins.md#cg-p-01) for naming convention (must be `cargo-*`)
2. Check [CG-P-02](./03-cargo-plugins.md#cg-p-02) for argument handling
3. Check [CG-P-03](./03-cargo-plugins.md#cg-p-03) for `--help` integration

### "CI builds are taking forever"
1. Check [CG-A-03](./06-cargo-advanced.md#cg-a-03) for proper CI caching
2. Check [CG-A-08](./06-cargo-advanced.md#cg-a-08) for CI pipeline design
3. Check [CG-A-04](./06-cargo-advanced.md#cg-a-04) for incremental compilation in CI

### "Getting future incompatibility warnings"
1. Check [CG-A-12](./06-cargo-advanced.md#cg-a-12) for handling future incompatibilities
2. Check [CG-A-11](./06-cargo-advanced.md#cg-a-11) for dependency update strategies

## 🌐 External Resources

### Official Documentation
- [The Cargo Book](https://doc.rust-lang.org/cargo/) - Official Cargo documentation
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) - API design best practices
- [The Rust Book](https://doc.rust-lang.org/book/) - Learn Rust programming

### Tools
- [cargo-edit](https://github.com/killercup/cargo-edit) - Add, remove, upgrade dependencies
- [cargo-release](https://github.com/crate-ci/cargo-release) - Automated release workflow
- [cargo-machete](https://github.com/bnjbvr/cargo-machete) - Find unused dependencies
- [cross](https://github.com/cross-rs/cross) - Easy cross-compilation

### Community
- [crates.io](https://crates.io/) - Rust package registry
- [lib.rs](https://lib.rs/) - Alternative crate discovery
- [Rust Users Forum](https://users.rust-lang.org/) - Get help from the community
- [r/rust](https://reddit.com/r/rust) - Rust subreddit

## 📜 License

These guides are provided as educational material for the Rust community.

---

**Last Updated**: 2026-01-09

**Version**: 2.0.0

**Guides Included**: 6 (Basics, Build System, Plugins, Publishing, Configuration, Advanced)
