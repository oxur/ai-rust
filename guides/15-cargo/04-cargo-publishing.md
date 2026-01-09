# Cargo Publishing

Essential patterns for publishing crates to registries, managing versions with SemVer, and maintaining backward compatibility. These patterns ensure smooth releases and happy users.

---

## CG-PUB-01: Complete Package Metadata Before Publishing

**Strength**: MUST

**Summary**: Fill in all required and recommended metadata fields before your first publish.

```toml
[package]
name = "my-awesome-crate"
version = "0.1.0"
edition = "2024"

# ✅ REQUIRED for crates.io:
license = "MIT OR Apache-2.0"     # or license-file = "LICENSE.txt"
description = "A short description of what this crate does"

# ✅ STRONGLY RECOMMENDED:
repository = "https://github.com/username/my-awesome-crate"
readme = "README.md"
homepage = "https://my-awesome-crate.rs"
documentation = "https://docs.rs/my-awesome-crate"

# ✅ RECOMMENDED for discoverability:
keywords = ["cli", "parser", "utility"]  # Max 5, helps search
categories = ["command-line-utilities"]   # From crates.io list

# ✅ OPTIONAL but useful:
authors = ["Your Name <you@example.com>"]
rust-version = "1.70"                     # MSRV

# ❌ BAD: Missing required fields
# [package]
# name = "my-crate"
# version = "0.1.0"
# # cargo publish will reject this!

# ❌ BAD: Vague description
description = "A utility"  # Too vague! What does it do?

# ❌ BAD: Too many keywords
keywords = ["cli", "parser", "utility", "awesome", "fast", "best"]
# Max is 5!
```

**Rationale**: crates.io requires license and description. Other fields dramatically improve discoverability and give users confidence. A well-documented crate gets more users and contributors. The metadata becomes part of the public registry forever, so get it right before publishing.

**See also**: CG-PUB-02 (pre-publish checklist), Package metadata documentation

---

## CG-PUB-02: Follow Pre-Publish Checklist

**Strength**: SHOULD

**Summary**: Always run `cargo publish --dry-run` and verify contents before publishing.

```bash
# ✅ GOOD: Pre-publish workflow
$ cargo test                      # All tests pass
$ cargo clippy -- -D warnings     # No warnings
$ cargo doc --no-deps             # Docs build correctly
$ cargo package --list            # Check included files
$ cargo publish --dry-run         # Verify without publishing
$ cargo publish                   # Actually publish

# Check what files will be published:
$ cargo package --list
Cargo.lock
Cargo.toml
LICENSE-APACHE
LICENSE-MIT
README.md
src/lib.rs
src/utils.rs

# ❌ BAD: Publishing without checking
$ cargo publish
# Oops! Included 500MB of test data, or forgot the README!

# ✅ GOOD: Exclude unnecessary files
[package]
exclude = [
    "benches/",
    "examples/large-dataset.bin",
    ".github/",
    "*.png",  # Exclude large images
]

# Or use include (more restrictive):
[package]
include = [
    "src/**/*.rs",
    "Cargo.toml",
    "LICENSE-*",
    "README.md",
]
```

**Rationale**: `--dry-run` catches errors before they're permanent. crates.io has a 10MB size limit and versions cannot be deleted (only yanked). Excluding development files keeps package size small and avoids leaking sensitive information.

**See also**: CG-PUB-01 (metadata), CG-PUB-06 (yanking)

---

## CG-PUB-03: Follow Rust SemVer Rules

**Strength**: MUST

**Summary**: Version changes must follow SemVer rules for Rust API compatibility.

```toml
# Version format: MAJOR.MINOR.PATCH
# 1.2.3 means:
# - MAJOR (1): Breaking changes
# - MINOR (2): New features, backward compatible
# - PATCH (3): Bug fixes only

# ✅ MINOR changes (bump minor version):
# - Adding new public items (functions, types, modules)
# - Adding trait methods with default implementations
# - Adding new variants to #[non_exhaustive] enums
# - Adding new fields to #[non_exhaustive] structs
# - Loosening generic bounds
# - Adding optional dependencies/features

# ✅ PATCH changes (bump patch version):
# - Bug fixes that don't change API
# - Performance improvements
# - Documentation updates
# - Internal refactoring

# ❌ MAJOR changes (require major version bump):
# - Removing or renaming public items
# - Changing function signatures
# - Adding variants to exhaustive enums
# - Adding fields to exhaustive structs
# - Tightening generic bounds
# - Changing error types
# - Removing features
```

```rust
// Example version transitions:

// 1.0.0 → 1.1.0 (MINOR: added function)
// Before:
pub fn existing_fn() {}

// After:
pub fn existing_fn() {}
pub fn new_fn() {}  // ✅ New item is minor change

// 1.0.0 → 2.0.0 (MAJOR: changed signature)
// Before:
pub fn process(data: &str) {}

// After:
pub fn process(data: &str, options: Options) {}  // ❌ Added parameter is breaking

// 1.0.0 → 1.0.1 (PATCH: fix bug)
// Before:
pub fn calculate(x: i32) -> i32 {
    x * 2  // Bug: should be x * 3
}

// After:
pub fn calculate(x: i32) -> i32 {
    x * 3  // ✅ Bug fix is patch
}

// Using #[non_exhaustive] for future flexibility:
#[non_exhaustive]
pub enum Error {
    NotFound,
    PermissionDenied,
    // Can add variants in minor releases!
}

// 1.0.0 → 1.1.0 (MINOR: added variant)
#[non_exhaustive]
pub enum Error {
    NotFound,
    PermissionDenied,
    Timeout,  // ✅ OK because #[non_exhaustive]
}
```

**Rationale**: SemVer allows Cargo to automatically update dependencies without breaking builds. Violating SemVer breaks users' CI and production systems. Rust has specific SemVer rules (adding enum variants is usually breaking unless using `#[non_exhaustive]`).

**See also**: CG-PUB-04 (breaking changes), SemVer Compatibility chapter

---

## CG-PUB-04: Identify Breaking Changes Correctly

**Strength**: MUST

**Summary**: Recognize subtle breaking changes that require major version bumps.

```rust
// ❌ BREAKING: Adding trait bounds
// Before (1.0.0):
pub fn process<T>(value: T) {}

// After (trying 1.1.0):
pub fn process<T: Clone>(value: T) {}  // BREAKING! Needs 2.0.0

// ❌ BREAKING: Changing trait item signature
// Before (1.0.0):
pub trait Handler {
    fn handle(&self);
}

// After (trying 1.1.0):
pub trait Handler {
    fn handle(&mut self);  // BREAKING! &self → &mut self
}

// ❌ BREAKING: Adding non-defaulted trait method
// Before (1.0.0):
pub trait Plugin {
    fn name(&self) -> &str;
}

// After (trying 1.1.0):
pub trait Plugin {
    fn name(&self) -> &str;
    fn version(&self) -> &str;  // BREAKING! No default impl
}

// ✅ NON-BREAKING: Adding defaulted trait method (minor change)
pub trait Plugin {
    fn name(&self) -> &str;
    fn version(&self) -> &str {
        "1.0"  // Has default, so existing impls still work
    }
}

// ❌ BREAKING: Removing public items
// Before (1.0.0):
pub fn old_function() {}

// After (trying 1.1.0):
// Removed old_function  // BREAKING! Needs 2.0.0

// ✅ NON-BREAKING: Deprecating then removing later
// Version 1.1.0:
#[deprecated(since = "1.1.0", note = "use new_function instead")]
pub fn old_function() {}
pub fn new_function() {}

// Version 2.0.0:
// Now safe to remove old_function

// ❌ BREAKING: Changing struct to non_exhaustive
// Before (1.0.0):
pub struct Config {
    pub timeout: u64,
}

// After (trying 1.1.0):
#[non_exhaustive]  // BREAKING! Users can no longer construct
pub struct Config {
    pub timeout: u64,
}
```

**Rationale**: Many changes seem harmless but break compilation or behavior. Adding trait bounds breaks generic code. Changing traits breaks implementations. Understanding Rust-specific breaking changes prevents accidental SemVer violations.

**See also**: CG-PUB-03 (SemVer rules), SemVer Compatibility chapter

---

## CG-PUB-05: Use Deprecation Before Removal

**Strength**: SHOULD

**Summary**: Mark items as deprecated in a minor release before removing them in a major release.

```rust
// ✅ GOOD: Gradual deprecation process

// Version 1.0.0: Original API
pub fn process_data(data: &str) -> Result<(), Error> {
    // old implementation
    Ok(())
}

// Version 1.5.0: Introduce replacement, deprecate old
#[deprecated(
    since = "1.5.0",
    note = "use `process_data_v2` instead, which handles errors better"
)]
pub fn process_data(data: &str) -> Result<(), Error> {
    // Keep working for backward compatibility
    Ok(())
}

pub fn process_data_v2(data: &str) -> Result<String, Error> {
    // Better API
    Ok(data.to_uppercase())
}

// Version 2.0.0: Remove deprecated item
// Now it's safe to remove process_data entirely

// ✅ GOOD: Deprecation with migration path
#[deprecated(
    since = "2.1.0",
    note = "use `Config::builder()` instead. \
            Example: Config::builder().timeout(30).build()"
)]
pub fn new(timeout: u64) -> Config {
    Config { timeout }
}

pub fn builder() -> ConfigBuilder {
    ConfigBuilder::default()
}

// ❌ BAD: Removing without deprecation warning
// Version 1.5.0:
pub fn old_api() {}

// Version 1.6.0:
// Removed old_api without warning!  // Users' builds break unexpectedly
```

**Rationale**: Deprecation warnings give users time to migrate before the removal. A typical timeline: deprecate in minor release, wait 6-12 months or 2-3 minor releases, then remove in next major. This respects users' time and maintains ecosystem stability.

**See also**: CG-PUB-04 (breaking changes), rust-version field

---

## CG-PUB-06: Yank vs New Version Strategically

**Strength**: MUST

**Summary**: Yank broken versions; release new versions for intentional changes.

```bash
# ✅ GOOD: Yank for serious bugs
$ cargo yank --version 1.2.3
# Use when:
# - Version doesn't compile
# - Has critical security flaw
# - Has data corruption bug
# - Was published by mistake

# ✅ GOOD: Undo a yank if needed
$ cargo yank --version 1.2.3 --undo

# Yanking does NOT delete code:
# - Existing Cargo.lock files still work
# - New projects won't select yanked version
# - Can't yank if other crates depend on it in their Cargo.lock

# ❌ BAD: Yanking for intentional changes
$ cargo yank --version 1.2.3  # Don't yank to "force" an update
$ cargo publish # Publishing 1.2.4 with API changes
# Just publish 1.2.4 directly!

# ❌ BAD: Trying to hide mistakes
# You accidentally published internal secrets
$ cargo yank --version 1.2.3
# NOT ENOUGH! Yank doesn't delete code!
# Anyone who downloaded it still has it
# → Immediately rotate credentials/secrets

# ✅ GOOD: For bugs without security implications
# Found a bug in 1.2.3, fix it in 1.2.4
$ cargo publish # Publishing 1.2.4
# No need to yank 1.2.3 unless it's severely broken
```

**Rationale**: Yanking prevents new dependents but doesn't delete code - it's not for hiding mistakes. Use yanking sparingly for truly broken releases. For normal bug fixes, just publish a new version. Users with Cargo.lock won't automatically update anyway.

**See also**: CG-PUB-02 (pre-publish checks), cargo yank documentation

---

## CG-PUB-07: Document Breaking Changes in CHANGELOG

**Strength**: SHOULD

**Summary**: Maintain a CHANGELOG that clearly documents breaking changes and migration paths.

```markdown
# CHANGELOG.md

## [2.0.0] - 2026-01-09

### Breaking Changes
- **BREAKING**: `process_data` now returns `Result<String, Error>` instead of 
  `Result<(), Error>` to provide processed data to callers.
  
  **Migration**: If you were ignoring the return value, no changes needed. 
  If you need the old behavior, use `process_data_v1` (deprecated).
  
  ```rust
  // Before:
  process_data(&input)?;
  
  // After:
  let result = process_data(&input)?;
  // or if you don't need the result:
  let _ = process_data(&input)?;
  ```

- **BREAKING**: Removed deprecated `old_function` (deprecated since 1.5.0).
  Use `new_function` instead.

### Added
- New `Config::builder()` API for more flexible configuration
- Support for async operations with `async-std` feature

### Fixed
- Fixed panic when handling empty input strings (#123)

## [1.5.0] - 2025-12-01

### Deprecated
- `process_data` - use `process_data_v2` instead (will be removed in 2.0.0)

### Added
- New `process_data_v2` with improved error handling

// ✅ GOOD: Clear migration instructions
// ✅ GOOD: Links to issues/PRs
// ✅ GOOD: Separate sections for breaking vs non-breaking

// ❌ BAD: Vague changelog
// ## [2.0.0]
// - Various improvements
// - Fixed bugs
// - Breaking changes
// (What broke? How do I migrate?)
```

**Rationale**: A good CHANGELOG is documentation for migrations. Users deciding whether to upgrade need to know what breaks and how to fix it. Link to detailed migration guides for complex changes. Follow Keep a Changelog format for consistency.

**See also**: CG-PUB-05 (deprecation), Keep a Changelog format

---

## CG-PUB-08: Tag Releases in Version Control

**Strength**: SHOULD

**Summary**: Create git tags for each published version to enable reproducible builds.

```bash
# ✅ GOOD: Tag when publishing
$ git tag v1.2.3
$ git push origin v1.2.3
$ cargo publish

# Or with annotation:
$ git tag -a v1.2.3 -m "Release version 1.2.3"
$ git push origin v1.2.3

# ✅ GOOD: Automated release process
# Use tools like cargo-release, release-plz, or cargo-smart-release
$ cargo release 1.2.3

# ❌ BAD: Publishing without tagging
$ cargo publish
# Now: which commit was 1.2.3? Hard to reproduce!

# ❌ BAD: Tag doesn't match published version
$ cargo publish  # Publishing 1.2.3
$ git tag v1.2.4 # Wrong tag!
```

**Rationale**: Tags create an audit trail mapping crate versions to source code. This enables: reproducing builds, investigating issues in specific versions, checking out old versions, and verifying published code matches source. Automated release tools can handle tagging for you.

**See also**: CG-PUB-09 (automation), cargo-release

---

## CG-PUB-09: Automate Release Process

**Strength**: CONSIDER

**Summary**: Use release automation tools to reduce human error and streamline publishing.

```toml
# Using cargo-release
[package.metadata.release]
pre-release-commit-message = "chore: Release {{crate_name}} v{{version}}"
tag-message = "chore: Release {{crate_name}} v{{version}}"
tag-name = "v{{version}}"
```

```bash
# ✅ GOOD: Using cargo-release
$ cargo install cargo-release
$ cargo release patch  # Bumps patch, tags, publishes

# Workflow:
# 1. Updates version in Cargo.toml
# 2. Updates CHANGELOG.md
# 3. Commits changes
# 4. Creates git tag
# 5. Pushes to remote
# 6. Publishes to crates.io

# Alternative tools:
$ cargo install release-plz  # Automated releases from CI
$ cargo install cargo-smart-release  # For workspaces

# ✅ GOOD: GitHub Actions automation
name: Release
on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo publish --token ${{ secrets.CARGO_TOKEN }}

# ❌ BAD: Manual and error-prone
# 1. Edit Cargo.toml
# 2. Edit CHANGELOG
# 3. git commit
# 4. git tag (maybe forget this)
# 5. git push (maybe forget tags)
# 6. cargo publish (maybe use wrong version)
```

**Rationale**: Automation reduces mistakes in the multi-step release process. Tools ensure version consistency across Cargo.toml, tags, and CHANGELOG. CI integration prevents publishing from dirty working directories or uncommitted changes.

**See also**: CG-PUB-08 (tagging), CI/CD tools

---

## CG-PUB-10: Handle Pre-1.0 Versions Correctly

**Strength**: MUST

**Summary**: Understand 0.x version semantics: 0.y.z treats y as major, z as minor.

```toml
# For pre-1.0 releases:
# 0.y.z format where:
# - 0: Still in initial development
# - y: MAJOR changes (like 1.x.y → 2.x.y)
# - z: MINOR changes (like 1.x.y → 1.x+1.y)

# ✅ GOOD: Pre-1.0 version progression
version = "0.1.0"   # Initial release
version = "0.1.1"   # Bug fix (minor)
version = "0.1.2"   # Add features (minor)
version = "0.2.0"   # Breaking change!
version = "0.2.1"   # Bug fix
version = "0.3.0"   # Another breaking change
version = "1.0.0"   # Stable API!

# Cargo treats these as compatible:
# ^0.1.2 → 0.1.2, 0.1.3, 0.1.4, ...
# ^0.2.0 → 0.2.0, 0.2.1, 0.2.2, ...
# But NOT: 0.1.x → 0.2.x (breaking!)

# ❌ BAD: Jumping to 1.0.0 too soon
version = "0.1.0"   # Still figuring out API
version = "1.0.0"   # Committed to stability!
# Now you're stuck with this API forever

# ✅ GOOD: Using 0.x while evolving
# Stay on 0.x until API is stable and battle-tested
version = "0.8.0"   # Many iterations
version = "0.9.0"   # Almost there
version = "1.0.0"   # Now we're confident!

# ✅ GOOD: 0.0.x for experimental crates
version = "0.0.1"   # Extremely unstable
version = "0.0.2"   # Each patch can break
# Every version is potentially breaking!
```

**Rationale**: 0.x versions signal "not yet stable" and allow breaking changes more frequently. Cargo's SemVer implementation treats 0.y as the major version. Don't rush to 1.0.0 - it's a promise of stability. Use 0.x liberally during development.

**See also**: CG-PUB-03 (SemVer rules), SemVer spec section 4

---

## CG-PUB-11: Specify rust-version (MSRV)

**Strength**: SHOULD

**Summary**: Document Minimum Supported Rust Version to help users plan upgrades.

```toml
[package]
name = "my-crate"
version = "1.5.0"
rust-version = "1.70.0"  # MSRV

# ✅ GOOD: Document MSRV in README too
# README.md:
# ## Minimum Supported Rust Version (MSRV)
# This crate requires Rust 1.70.0 or later.

# Cargo respects rust-version in dependency resolution

# ✅ GOOD: MSRV policy
# - Increasing MSRV is a minor breaking change (bump minor)
# - Document MSRV policy in README:
#   "We support Rust versions from the past 6 months"
#   or "We support the latest stable and 2 previous versions"

# ✅ GOOD: Test MSRV in CI
# .github/workflows/ci.yml
matrix:
  rust:
    - 1.70.0  # MSRV
    - stable
    - beta

# ❌ BAD: Not specifying MSRV
# Users don't know which Rust version they need

# ❌ BAD: Increasing MSRV without communication
# Version 1.5.0: rust-version = "1.65"
# Version 1.5.1: rust-version = "1.75"  # Breaks users on 1.65!
# Should be 1.6.0 (minor bump) or 2.0.0 (if major change)
```

**Rationale**: `rust-version` documents MSRV and helps Cargo's resolver avoid incompatible versions. Bumping MSRV is a compatibility concern - users on older Rust can't upgrade. Have a policy (like "last 6 months" or "last 3 releases") and document it.

**See also**: CG-PUB-03 (SemVer), rust-version field documentation

---

## CG-PUB-12: Prepare for crates.io Requirements

**Strength**: MUST

**Summary**: Understand crates.io-specific requirements and policies before publishing.

```toml
# ✅ crates.io requirements:
[package]
name = "my-crate"  # Must be unique, lowercase, ASCII, max 64 chars
version = "1.0.0"  # Must be valid SemVer
license = "MIT"    # REQUIRED - or license-file
description = "A short description"  # REQUIRED

# ❌ crates.io will REJECT:
# - Names with capital letters, spaces, underscores
# - Names of existing crates (case-insensitive)
# - Reserved names (rust, cargo, etc.)
# - Packages > 10MB
# - No license specified
# - No description
# - Invalid SemVer version

# ✅ GOOD: Check name availability first
$ cargo search my-crate
# If results appear, name is taken

# ✅ GOOD: Namespace in name if generic
# Instead of: json, parser, utils
# Use: myproject-json, myproject-parser, myproject-utils

# ✅ GOOD: License options
license = "MIT"                         # SPDX identifier
license = "MIT OR Apache-2.0"           # Dual license (common for Rust)
license = "Apache-2.0 WITH LLVM-exception"
license-file = "LICENSE.txt"            # Custom license

# ❌ BAD: Path dependencies without version
[dependencies]
my-local-crate = { path = "../local" }
# cargo publish will reject! Add version:
my-local-crate = { version = "1.0", path = "../local" }
```

**Rationale**: crates.io has strict requirements to maintain quality and prevent abuse. Checking these before `cargo publish` prevents frustrating rejections. Names are permanent - choose wisely. Some policies (like the 10MB limit) have historical reasons but are firm.

**See also**: CG-PUB-01 (metadata), crates.io policies

---

## Best Practices Summary

### Quick Reference Table

| Pattern | Strength | Key Insight |
|---------|----------|-------------|
| Complete metadata | MUST | license and description required, rest improves discoverability |
| Pre-publish checklist | SHOULD | Always --dry-run, check file list, verify tests pass |
| SemVer rules | MUST | MAJOR=breaking, MINOR=features, PATCH=fixes |
| Identify breaking changes | MUST | Many changes are subtly breaking in Rust |
| Deprecation before removal | SHOULD | Warn in minor, remove in major, give users time |
| Yank vs new version | MUST | Yank for broken builds, new version for fixes |
| CHANGELOG | SHOULD | Document breaking changes with migration instructions |
| Tag releases | SHOULD | Git tags enable reproducibility and audit trail |
| Automate releases | CONSIDER | Reduce human error with cargo-release or CI |
| Pre-1.0 versions | MUST | 0.y.z: y is major, z is minor |
| Specify rust-version | SHOULD | Document MSRV, have upgrade policy |
| crates.io requirements | MUST | Check name, license, size limits before publishing |

---

## Related Guidelines

- **Basics**: See `01-cargo-basics.md` for package creation and dependencies
- **Build System**: See `02-cargo-build-system.md` for features and SemVer interaction
- **Configuration**: See `05-cargo-configuration.md` for registry configuration
- **Advanced**: See `06-cargo-advanced.md` for CI/CD publishing workflows

---

## External References

- [Cargo Book - Publishing](https://doc.rust-lang.org/cargo/reference/publishing.html)
- [Cargo Book - SemVer Compatibility](https://doc.rust-lang.org/cargo/reference/semver.html)
- [SemVer Specification](https://semver.org/)
- [API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Keep a Changelog](https://keepachangelog.com/)
- [crates.io Policies](https://crates.io/policies)
