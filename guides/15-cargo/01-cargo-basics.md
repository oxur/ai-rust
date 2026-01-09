# Cargo Basics

Essential Cargo patterns for package creation, dependency management, and workspace organization. These patterns form the foundation for working effectively with Rust's package manager.

---

## CG-B-01: Use `cargo new` for New Projects

**Strength**: MUST

**Summary**: Always use `cargo new` to create new projects rather than manually creating directory structure.

```rust
// ✅ GOOD: Standard project creation
$ cargo new my-app              // Creates binary crate (default)
$ cargo new my-lib --lib        // Creates library crate
$ cargo new my-app --vcs none   // Skip git initialization

// Result:
// my-app/
// ├── Cargo.toml
// └── src/
//     └── main.rs  (or lib.rs for --lib)

// ❌ BAD: Manually creating structure
$ mkdir my-app
$ cd my-app
$ touch Cargo.toml src/main.rs  // Error-prone, misses conventions
```

**Rationale**: `cargo new` ensures correct directory structure, initializes git repository, creates properly formatted Cargo.toml with current edition, and follows all conventions. Manual creation is error-prone and often misses important defaults.

**See also**: CG-B-02 (cargo init for existing directories)

---

## CG-B-02: Use `cargo init` for Existing Directories

**Strength**: SHOULD

**Summary**: Use `cargo init` to convert an existing directory into a Cargo project.

```bash
# ✅ GOOD: Initialize in existing directory
$ cd existing-project
$ cargo init                    // Binary crate in current directory
$ cargo init --lib              // Library crate
$ cargo init --vcs git          // Explicitly initialize git

# ❌ BAD: Creating new directory then initializing
$ cargo new ../temp
$ mv ../temp/* .
$ rm -rf ../temp
```

**Rationale**: `cargo init` is designed specifically for existing directories, while `cargo new` creates a new directory. Using the right command communicates intent clearly and avoids unnecessary directory manipulation.

---

## CG-B-03: Follow Standard Package Layout

**Strength**: MUST

**Summary**: Organize code according to Cargo's standard directory structure for automatic target discovery.

```
my-package/
├── Cargo.toml
├── Cargo.lock
├── src/
│   ├── lib.rs              # Library root (for lib crates)
│   ├── main.rs             # Default binary (for bin crates)
│   └── bin/
│       ├── tool1.rs        # Additional binary
│       └── tool2/          # Multi-file binary
│           ├── main.rs
│           └── module.rs
├── tests/
│   ├── integration_test.rs
│   └── multi_file_test/
│       ├── main.rs
│       └── test_module.rs
├── examples/
│   ├── simple.rs
│   └── complex/
│       ├── main.rs
│       └── helper.rs
└── benches/
    └── benchmark.rs

// ✅ GOOD: Following conventions
// src/main.rs - found automatically as default binary
// src/bin/cli-tool.rs - found as additional binary

// ❌ BAD: Non-standard locations
// app/main.rs - won't be discovered
// src/binaries/tool.rs - requires manual Cargo.toml configuration
```

**Rationale**: Cargo automatically discovers targets in standard locations, reducing configuration. Deviating requires manual `[[bin]]`, `[[test]]`, `[[example]]`, or `[[bench]]` sections in Cargo.toml, adding unnecessary complexity.

**See also**: CG-B-04 (naming conventions)

---

## CG-B-04: Follow Naming Conventions for Targets

**Strength**: MUST

**Summary**: Use kebab-case for binaries, examples, benches, and integration tests; snake_case for modules.

```rust
// ✅ GOOD: Proper naming
src/bin/api-server.rs          // kebab-case for binary
src/bin/data-processor.rs      // kebab-case for binary
examples/hello-world.rs        // kebab-case for example
benches/parse-performance.rs   // kebab-case for benchmark
tests/end-to-end.rs            // kebab-case for integration test

// Inside files: snake_case for modules
mod database_connection;
mod http_handler;

// ❌ BAD: Wrong case
src/bin/apiServer.rs           // camelCase
src/bin/data_processor.rs      // snake_case for binary
examples/HelloWorld.rs         // PascalCase
mod DatabaseConnection;        // PascalCase for module
```

**Rationale**: Rust style guidelines (RFC 430) define clear casing rules. Binaries/executables use kebab-case matching Unix conventions, while Rust code uses snake_case. Consistency across the ecosystem aids readability and tooling.

**See also**: Rust API Guidelines C-CASE

---

## CG-B-05: Understand Cargo.toml vs Cargo.lock

**Strength**: MUST

**Summary**: Cargo.toml specifies dependency requirements broadly; Cargo.lock records exact versions used.

```toml
# Cargo.toml - YOUR specifications
[dependencies]
serde = "1.0"          # Means "^1.0" - any 1.x compatible version
regex = "1.10"         # Will accept 1.10.0, 1.10.1, 1.11.0, etc.
```

```toml
# Cargo.lock - CARGO's exact resolution (auto-generated)
[[package]]
name = "serde"
version = "1.0.193"    # Exact version locked
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "79fa87...

# ✅ GOOD: Commit Cargo.lock for applications and binaries
$ git add Cargo.lock
$ git commit -m "Lock dependency versions"

# ✅ GOOD: DON'T commit Cargo.lock for libraries
$ echo "Cargo.lock" >> .gitignore  # For library crates

# ❌ BAD: Manually editing Cargo.lock
# Never edit Cargo.lock by hand - it's managed by Cargo
```

**Rationale**: Cargo.toml lets you specify compatibility ranges (important for libraries to avoid over-constraining). Cargo.lock ensures reproducible builds by recording exact versions. Applications need reproducibility; libraries should allow flexibility. Manually editing Cargo.lock breaks Cargo's assumptions.

**See also**: CG-B-06 (when to commit Cargo.lock), CG-PUB-03 (SemVer compatibility)

---

## CG-B-06: Commit Cargo.lock for Applications, Not Libraries

**Strength**: MUST

**Summary**: Binary crates and applications should commit Cargo.lock; library crates should not.

```bash
# For applications (src/main.rs, binaries):
# ✅ GOOD: Track Cargo.lock
$ git add Cargo.lock
# Ensures reproducible builds across all environments

# For libraries (src/lib.rs, no binaries):
# ✅ GOOD: Ignore Cargo.lock
$ echo "Cargo.lock" >> .gitignore
# Allows users to get latest compatible dependencies

# ❌ BAD: Library committing Cargo.lock
#  Forces downstream users to use YOUR exact dependency versions
#  Defeats the purpose of SemVer compatibility ranges

# ❌ BAD: Application not committing Cargo.lock
#  Different developers get different versions
#  CI and production might use different versions than development
```

**Rationale**: Applications need reproducible builds - everyone building should get identical dependencies. Libraries need compatibility flexibility - downstream users should get the latest compatible versions within the SemVer range you specified. Committing a library's Cargo.lock forces unnecessary constraints on users.

**See also**: FAQ "Why have Cargo.lock in version control?"

---

## CG-B-07: Use Specific Version Requirements Appropriately

**Strength**: SHOULD

**Summary**: Choose dependency version requirements based on your needs and SemVer compatibility.

```toml
[dependencies]
# ✅ GOOD: Caret requirement (default) - accepts compatible updates
serde = "1.0"          # Same as "^1.0" - allows 1.x.y where x,y ≥ 0
tokio = "^1.35"        # Allows >= 1.35.0 and < 2.0.0

# ✅ GOOD: Tilde requirement - accepts patch updates only
regex = "~1.10.2"      # Allows >= 1.10.2 and < 1.11.0
# Useful when you need stability within a minor version

# ✅ GOOD: Wildcard - accepts any patch version
log = "0.4.*"          # Allows any 0.4.x version

# ✅ GOOD: Exact version - when you need specific behavior
some-sys = "=2.1.0"    # Exactly 2.1.0, no updates
# Use sparingly - only for known incompatibilities

# ✅ GOOD: Multiple requirements
serde_json = ">= 1.0.0, < 2.0.0"

# ❌ BAD: Overly restrictive for no reason
actix-web = "=4.4.0"   # Prevents security patches!
# Unless you have a specific bug to avoid

# ❌ BAD: Wildcard for major version
reqwest = "*"          # Accepts breaking changes!
some-crate = "1.*"     # Same as "^1.0" but less clear
```

**Rationale**: SemVer compatibility (`^`) is the default because it allows security patches and bug fixes while protecting against breaking changes. More restrictive requirements should have specific justifications. Exact versions (`=`) prevent patches and should be rare.

**See also**: CG-PUB-03 (SemVer rules), Specifying Dependencies documentation

---

## CG-B-08: Use Path Dependencies for Local Development

**Strength**: SHOULD

**Summary**: Use path dependencies for unpublished crates or local development, but not for published crates.

```toml
[dependencies]
# ✅ GOOD: Local dependency during development
my-utils = { path = "../my-utils" }

# ✅ GOOD: Path with version for publishability
serde = { version = "1.0", path = "../serde" }
# Cargo uses version when publishing, path during local dev

# Workspace members:
[dependencies]
# ✅ GOOD: Referencing workspace member
my-core = { path = "../my-core", version = "0.1" }

# ❌ BAD: Path-only dependency in published crate
[dependencies]
helper = { path = "/home/user/projects/helper" }
# Won't work for anyone else! Cargo will reject on publish

# ❌ BAD: Absolute paths
utils = { path = "/Users/alice/code/utils" }
# Not portable across machines
```

**Rationale**: Path dependencies enable working on multiple related crates simultaneously. When publishing, Cargo requires dependencies exist on crates.io (or a registry), so path-only dependencies will cause publish failures unless you include a `version` field.

**See also**: CG-B-09 (git dependencies), Workspaces

---

## CG-B-09: Use Git Dependencies Sparingly

**Strength**: CONSIDER

**Summary**: Git dependencies are useful for unreleased changes but hurt reproducibility; prefer released versions.

```toml
[dependencies]
# ✅ ACCEPTABLE: Temporary use of unreleased fix
regex = { git = "https://github.com/rust-lang/regex", rev = "9f9f693" }
# Always pin to specific rev/tag/branch for reproducibility

# ✅ GOOD: Pinned to a tag
serde = { git = "https://github.com/serde-rs/serde", tag = "v1.0.150" }

# ✅ GOOD: Pinned to specific commit
tokio = { git = "https://github.com/tokio-rs/tokio", rev = "abc123def456" }

# ❌ BAD: No rev/tag specified
actix-web = { git = "https://github.com/actix/actix-web" }
# Uses default branch HEAD - not reproducible!

# ❌ BAD: Branch without commit
reqwest = { git = "https://github.com/seanmonstar/reqwest", branch = "master" }
# Branch can change - breaks reproducibility

# ✅ BETTER: Once released, switch to registry version
serde = "1.0.150"
```

**Rationale**: Git dependencies bypass Cargo's dependency resolution and version checking. They make builds non-reproducible if not pinned and complicate dependency management. Use for temporary workarounds or pre-release testing, then switch to registry versions.

**See also**: CG-B-05 (Cargo.lock), Specifying Dependencies

---

## CG-B-10: Prefer Workspaces for Multi-Crate Projects

**Strength**: SHOULD

**Summary**: Use workspaces to manage multiple related crates in a single repository.

```toml
# Workspace root Cargo.toml
[workspace]
members = [
    "app",
    "core",
    "utils",
    "crates/*",    # Glob patterns supported
]

# ✅ GOOD: Workspace shares dependencies and target directory
# All members use same lockfile
# Unified `cargo build` builds everything
# Single target/ directory saves disk space

# Example structure:
# project-root/
# ├── Cargo.toml          # Workspace definition
# ├── Cargo.lock          # Shared lockfile
# ├── target/             # Shared build directory
# ├── app/
# │   ├── Cargo.toml
# │   └── src/main.rs
# ├── core/
# │   ├── Cargo.toml
# │   └── src/lib.rs
# └── utils/
#     ├── Cargo.toml
#     └── src/lib.rs

# Member crate Cargo.toml
[package]
name = "app"
version = "0.1.0"

[dependencies]
core = { path = "../core" }      # Reference other workspace members
utils = { path = "../utils" }

# ❌ BAD: Separate repos for tightly coupled crates
# - Harder to make coordinated changes
# - Version management complexity
# - Separate CI for each

# ❌ BAD: Single crate with too many responsibilities
# - Becomes monolithic
# - Harder to reuse components
# - Longer compile times
```

**Rationale**: Workspaces enable managing multiple crates as a single project. They share a Cargo.lock (consistent dependencies), share target/ directory (faster builds, less disk space), and enable atomic changes across crates. Perfect for applications with multiple binaries, shared libraries, or plugin architectures.

**See also**: CG-B-11 (workspace dependency inheritance), Workspaces documentation

---

## CG-B-11: Use Workspace Dependency Inheritance

**Strength**: SHOULD

**Summary**: Define common dependencies once at workspace level to ensure consistency across members.

```toml
# Workspace Cargo.toml
[workspace]
members = ["app", "core", "utils"]

[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1.35", features = ["full"] }
anyhow = "1.0"

[workspace.package]
version = "0.2.0"
edition = "2024"
license = "MIT"
authors = ["Team Name"]

# ✅ GOOD: Member inherits workspace dependencies
# app/Cargo.toml
[package]
name = "app"
version.workspace = true
edition.workspace = true
license.workspace = true

[dependencies]
serde.workspace = true           # Inherits from workspace
tokio.workspace = true
core = { path = "../core" }

# ✅ GOOD: Override when needed
serde = { workspace = true, features = ["derive", "rc"] }
# Adds extra features to workspace version

# ❌ BAD: Duplicating versions across members
# core/Cargo.toml
[dependencies]
serde = "1.0"    # Version drift risk!

# app/Cargo.toml
[dependencies]
serde = "1.0.193"  # Different micro version!

# utils/Cargo.toml
[dependencies]
serde = "1"        # Even more vague!
```

**Rationale**: Workspace dependency inheritance ensures all crates use consistent versions, reducing the chance of version conflicts. It makes updates easier (change once in workspace root) and prevents accidental version drift across members. The feature must be explicitly enabled but is worth the setup cost for multi-crate projects.

**See also**: CG-B-10 (workspace basics), Workspace inheritance documentation

---

## CG-B-12: Understand Binary vs Library Crates

**Strength**: MUST

**Summary**: Choose between binary and library crates based on whether you're building an executable or reusable code.

```rust
// BINARY CRATE (src/main.rs):
// ✅ GOOD: Application entry point
fn main() {
    println!("Hello, world!");
}
// - cargo run executes this
// - cargo install installs the binary
// - Cargo.lock SHOULD be committed

// LIBRARY CRATE (src/lib.rs):
// ✅ GOOD: Reusable code
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
// - Used by other crates via dependencies
// - cargo build creates rlib
// - Cargo.lock should NOT be committed

// HYBRID CRATE (both src/main.rs and src/lib.rs):
// src/lib.rs - reusable logic
pub fn process_data(input: &str) -> String {
    // ... implementation
    input.to_uppercase()
}

// src/main.rs - thin binary wrapper
fn main() {
    let input = std::env::args().nth(1).expect("Need input");
    let result = my_crate::process_data(&input);
    println!("{}", result);
}

// ✅ GOOD: Library with binary
// - Enables testing of library logic
// - Binary is thin wrapper
// - Other crates can use the library
```

```toml
# Binary-only crate:
[package]
name = "my-app"
version = "0.1.0"

# Automatically creates binary from src/main.rs

# Library-only crate:
[package]
name = "my-lib"
version = "0.1.0"

# Automatically creates library from src/lib.rs

# Hybrid crate:
[package]
name = "my-tool"
version = "0.1.0"

# Automatically creates:
# - library from src/lib.rs
# - binary from src/main.rs (named "my-tool")
```

**Rationale**: Binary crates are end products (applications, tools) that users run. Library crates provide functionality for other crates. Most complex applications benefit from being hybrid - library containing logic (testable, reusable) with a thin binary wrapper (CLI handling, main entry point).

**See also**: CG-B-03 (package layout), CG-B-06 (Cargo.lock strategy)

---

## Best Practices Summary

### Quick Reference Table

| Pattern | Strength | Key Insight |
|---------|----------|-------------|
| Use cargo new | MUST | Always use for new projects - ensures correct structure |
| Standard layout | MUST | Follow src/, tests/, examples/, benches/ convention |
| Naming conventions | MUST | kebab-case for binaries/targets, snake_case for modules |
| Cargo.toml vs Cargo.lock | MUST | Toml = your specs, Lock = exact versions |
| Commit Cargo.lock (apps) | MUST | Apps need reproducibility, libraries need flexibility |
| Version requirements | SHOULD | Use `^` (default) for compatibility, `=` only when needed |
| Path dependencies | SHOULD | Good for development, include version for publishing |
| Git dependencies | CONSIDER | Pin to rev/tag, prefer registry versions when available |
| Workspaces | SHOULD | Manage multi-crate projects efficiently |
| Workspace inheritance | SHOULD | Define common dependencies once at workspace level |
| Binary vs library | MUST | Choose based on executable vs reusable code needs |

---

## Related Guidelines

- **Build System**: See `02-cargo-build-system.md` for features, profiles, and build scripts
- **Publishing**: See `04-cargo-publishing.md` for crates.io workflow and SemVer
- **Configuration**: See `05-cargo-configuration.md` for .cargo/config.toml
- **Plugins**: See `03-cargo-plugins.md` for creating custom cargo subcommands

---

## External References

- [Cargo Book - Guide](https://doc.rust-lang.org/cargo/guide/)
- [Cargo Book - Manifest Format](https://doc.rust-lang.org/cargo/reference/manifest.html)
- [Cargo Book - Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html)
- [Cargo Book - Specifying Dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html)
- [SemVer Specification](https://semver.org/)
