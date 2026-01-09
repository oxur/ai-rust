# Advanced Cargo Techniques

This guide covers performance optimization, unstable features, CI/CD integration, and advanced cargo workflows for experienced users.

---

## CG-A-01: Optimize Debug Builds for Faster Iteration

**Strength**: SHOULD

**Summary**: Reduce debug info generation to speed up development builds without sacrificing useful diagnostics.

Development builds generate full debug information by default, but you often only need basic backtraces. Reducing debug info significantly speeds up compilation and linking.

```toml
# ❌ BAD: Using defaults (slow dev builds)
# Cargo.toml with no profile customization
# Dev builds generate full debug info for all dependencies

# ✅ GOOD: Optimized dev profile
[profile.dev]
# Generate minimal debug info for your code (useful backtraces)
debug = "line-tables-only"

[profile.dev.package."*"]
# No debug info for dependencies (faster builds)
debug = false

# ✅ GOOD: Separate debugging profile when needed
[profile.debugging]
inherits = "dev"
debug = true  # Full debug info when actually debugging

# Usage: cargo build --profile debugging
```

```toml
# ✅ GOOD: Additional dev optimizations
[profile.dev]
debug = "line-tables-only"
incremental = true        # Enable incremental compilation
split-debuginfo = "unpacked"  # Faster on macOS

[profile.dev.package."*"]
opt-level = 0
debug = false

# Optimize specific slow dependencies
[profile.dev.package.serde]
opt-level = 2

[profile.dev.package.regex]
opt-level = 2
```

**Rationale**: Full debug information dramatically increases build and link times, especially for large projects with many dependencies. `line-tables-only` provides useful panic backtraces with file names and line numbers, which is sufficient for most development work. Disabling debug info for dependencies saves significant time since you rarely debug into third-party code. When you actually need to debug, use `--profile debugging` to get full debug symbols.

**See also**: CG-A-02 (alternative linkers), CG-A-03 (build cache), CG-BS-07 (profile configuration)

---

## CG-A-02: Use Alternative Linkers for Faster Linking

**Strength**: SHOULD

**Summary**: Configure faster linkers like `mold`, `lld`, or `zld` to reduce link times.

Linking can dominate build times, especially for incremental builds where only linking changes. Alternative linkers are significantly faster than the default system linker.

```toml
# ❌ BAD: Using default linker (slow)
# No linker configuration

# ✅ GOOD: Using mold on Linux
# .cargo/config.toml
[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold"]

# ✅ GOOD: Using lld on Linux
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]

# ✅ GOOD: Using zld on macOS
[target.x86_64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]

[target.aarch64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/opt/homebrew/bin/zld"]

# ✅ GOOD: Cross-platform configuration
[target.'cfg(target_os = "linux")']
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold"]

[target.'cfg(target_os = "macos")']
rustflags = ["-C", "link-arg=-fuse-ld=zld"]

# ✅ GOOD: Windows with lld-link
[target.'cfg(target_os = "windows")']
linker = "rust-lld.exe"
```

**Installation commands:**

```bash
# Linux (mold - fastest)
sudo apt install mold  # Ubuntu/Debian
brew install mold      # Homebrew

# Linux (lld)
sudo apt install lld   # Ubuntu/Debian

# macOS (zld)
brew install michaeleisel/zld/zld
```

**Rationale**: Default linkers (especially on Linux non-x86_64 targets) are slow. Modern alternative linkers like `mold` can be 2-10x faster. For large projects, this can save minutes on each build. The fastest linker varies by platform: `mold` on Linux, `zld` on macOS, `lld` on Windows. The configuration is per-developer (in `$HOME/.cargo/config.toml`) or per-project (in `.cargo/config.toml`), and doesn't affect release builds sent to users.

**See also**: CG-A-01 (dev profile optimization), CG-CF-04 (target configuration)

---

## CG-A-03: Leverage Build Caching in CI

**Strength**: MUST

**Summary**: Cache cargo build artifacts correctly to speed up CI pipelines without wasting space.

Naive caching of the entire `target/` directory is inefficient. Cargo's build cache has specific requirements for effective CI caching.

```yaml
# ❌ BAD: Caching entire CARGO_HOME or target/
- name: Cache cargo
  uses: actions/cache@v3
  with:
    path: |
      ~/.cargo           # Too much! Includes registry sources twice
      target/            # Too much! Includes incremental artifacts
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

# ✅ GOOD: Selective cargo caching (GitHub Actions)
- name: Install Rust
  uses: dtolnay/rust-toolchain@stable
  
- name: Cache cargo registry and build artifacts
  uses: actions/cache@v3
  with:
    path: |
      ~/.cargo/bin/
      ~/.cargo/registry/index/
      ~/.cargo/registry/cache/
      ~/.cargo/git/db/
      target/
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    restore-keys: |
      ${{ runner.os }}-cargo-

# Note: Excludes registry/src/ (extracted sources, redundant with cache/)

# ✅ GOOD: Using rust-cache action (recommended)
- name: Install Rust
  uses: dtolnay/rust-toolchain@stable
  
- name: Cache cargo
  uses: Swatinem/rust-cache@v2
  with:
    shared-key: "build"
    save-if: ${{ github.ref == 'refs/heads/main' }}

# ✅ GOOD: GitLab CI caching
cache:
  key:
    files:
      - Cargo.lock
  paths:
    - .cargo/bin/
    - .cargo/registry/index/
    - .cargo/registry/cache/
    - .cargo/git/db/
    - target/

# ✅ GOOD: CircleCI caching
- restore_cache:
    keys:
      - cargo-cache-{{ arch }}-{{ checksum "Cargo.lock" }}
      - cargo-cache-{{ arch }}-
      
- save_cache:
    key: cargo-cache-{{ arch }}-{{ checksum "Cargo.lock" }}
    paths:
      - ~/.cargo/registry/index
      - ~/.cargo/registry/cache
      - ~/.cargo/git/db
      - target
```

**Rationale**: Cargo's registry structure stores downloaded crates twice: compressed in `registry/cache/` and extracted in `registry/src/`. Caching both wastes bandwidth and storage. The `registry/cache/` directory is sufficient—cargo recreates `registry/src/` from it. The `target/` directory contains build artifacts that can be reused across builds. Using specialized actions like `rust-cache` handles all these details correctly and implements smart cache invalidation.

**See also**: CG-A-04 (incremental compilation), CG-A-08 (CI optimization)

---

## CG-A-04: Understand Incremental Compilation Trade-offs

**Strength**: CONSIDER

**Summary**: Incremental compilation speeds up rebuilds but increases disk usage and can have subtle bugs.

Incremental compilation is enabled by default in dev mode, but understanding its behavior helps optimize your workflow.

```toml
# ✅ GOOD: Default (incremental enabled for dev)
[profile.dev]
incremental = true  # Default, usually leave enabled

[profile.release]
incremental = false  # Default, don't enable for release

# ⚠️ CONSIDER: Disabling in CI for reproducibility
# In CI config file or .cargo/config.toml
[build]
# Disable incremental in CI for clean builds
incremental = false

# ✅ GOOD: Cleaning incremental cache when needed
```

```bash
# If you encounter weird compilation errors:
cargo clean -p your_crate  # Clean specific package
rm -rf target/debug/incremental/  # Nuclear option: clear incremental cache

# Regular incremental cache maintenance
cargo clean --release  # Keep dev cache, clear release
```

```toml
# ✅ GOOD: CI-specific configuration
# .cargo/config.toml (gitignored, CI sets this)
[profile.dev]
incremental = false  # Clean builds in CI

[profile.release]
incremental = false  # Always false for release
```

**Rationale**: Incremental compilation saves previous compilation results to avoid re-analyzing unchanged code. This dramatically speeds up rebuilds during development (typically 2-5x faster). However, incremental artifacts are large (gigabytes), non-portable across machines, and occasionally become corrupted, causing mysterious compile errors. In CI, incremental compilation doesn't help (clean builds) and wastes cache space. If you encounter strange compilation errors that `cargo clean` fixes, incremental corruption is the likely culprit. For reproducible release builds, always disable incremental.

**See also**: CG-A-03 (CI caching), CG-A-01 (dev optimization)

---

## CG-A-05: Use Workspace Feature Unification Carefully

**Strength**: CONSIDER

**Summary**: Workspace feature unification can reduce rebuilds but may mask dependency bugs.

Feature unification resolves features across the entire workspace, which can reduce rebuilds but has implications for correctness.

```toml
# ⚠️ CONSIDER: Workspace feature unification (unstable)
# .cargo/config.toml
[unstable]
resolver = "3"  # Requires nightly

# Or in Cargo.toml (future)
[workspace]
resolver = "3"

# ❌ BAD: Unaware of feature unification implications
# workspace-member-a/Cargo.toml
[dependencies]
serde = "1"  # Doesn't enable derive

# workspace-member-b/Cargo.toml
[dependencies]
serde = { version = "1", features = ["derive"] }

# With feature unification, member-a gets derive too!
# This can mask bugs where member-a should have enabled derive itself.

# ✅ GOOD: Explicit features even with unification
# workspace-member-a/Cargo.toml
[dependencies]
serde = { version = "1", features = ["derive"] }
# Explicit about what features we need

# workspace-member-b/Cargo.toml
[dependencies]
serde = { version = "1", features = ["derive"] }
# Same features, will reuse compiled artifacts
```

```bash
# Testing without feature unification (to catch missing feature declarations)
cd workspace-member-a
cargo build --locked
# If this fails but workspace build succeeds, you have missing features
```

**Rationale**: By default, cargo builds each workspace member independently, which can lead to the same dependency being built multiple times with different feature sets. Workspace feature unification (resolver v3) builds each dependency once with the union of all requested features, reducing redundant builds. However, this can hide bugs where a workspace member fails to declare needed features because another member enabled them. If feature unification is disabled, the bug surfaces. For large workspaces, the build time savings can be significant, but test individual members to ensure correct feature declarations.

**See also**: CG-B-08 (workspace inheritance), CG-BS-01 (feature design)

---

## CG-A-06: Leverage Unstable Features Responsibly

**Strength**: CONSIDER

**Summary**: Use unstable cargo features for productivity but understand the maintenance cost.

Cargo's unstable features (enabled with `-Z` flags) can provide significant benefits but require nightly Rust and may change.

```toml
# ⚠️ CONSIDER: Using unstable features
# Requires nightly: rustup default nightly

# .cargo/config.toml
[unstable]
# Faster parallel frontend
build-std = false
timings = ["html"]  # Build timing information

# Enable specific unstable features
[build]
rustflags = ["-Zthreads=8"]  # Parallel frontend

# ✅ GOOD: Document nightly requirement clearly
```

```markdown
# README.md
## Requirements

This project uses unstable Cargo features for development:
- Nightly Rust: `rustup default nightly`
- Enable unstable features: Set in `.cargo/config.toml`

For stable Rust users, remove `.cargo/config.toml` (builds will be slower).
```

```bash
# ✅ GOOD: CI configuration with nightly
# .github/workflows/ci.yml
- name: Install nightly Rust
  uses: dtolnay/rust-toolchain@nightly

# ❌ BAD: Requiring unstable features in published crates
# Don't use unstable cargo features in Cargo.toml of published crates
```

```toml
# ✅ GOOD: Unstable features in local config only
# Project .cargo/config.toml (for developers)
[unstable]
timings = ["html"]

[build]
rustflags = ["-Zthreads=8"]

# Cargo.toml (published) - no unstable features
[package]
name = "my-crate"
# Standard configuration only
```

**Useful unstable features:**

```bash
# Build timings (helpful for optimization)
cargo build -Ztimings

# Parallel frontend (faster compilation)
RUSTFLAGS="-Zthreads=8" cargo build

# Alternative codegen backend (faster debug builds)
cargo build -Zcodegen-backend=cranelift
```

**Rationale**: Unstable features can significantly improve developer productivity: parallel compilation, build timings, and alternative codegen backends can save hours of development time. However, they require nightly Rust and may break without warning. Use them in `.cargo/config.toml` (which is typically not committed or is gitignored) for local development speedups, but never require them in published crates. Document nightly requirements clearly if your project needs them. For CI, weigh the benefits against the maintenance burden of tracking nightly changes.

**See also**: CG-A-01 (dev optimization), CG-A-02 (alternative linkers)

---

## CG-A-07: Optimize Release Builds for Size or Speed

**Strength**: SHOULD

**Summary**: Customize release profiles based on your deployment constraints.

The default release profile balances various factors, but you can optimize specifically for binary size or maximum runtime performance.

```toml
# ❌ BAD: Using defaults without considering requirements
[profile.release]
# Defaults: opt-level = 3, lto = false, codegen-units = 16

# ✅ GOOD: Optimizing for speed
[profile.release]
opt-level = 3           # Maximum optimization
lto = "thin"            # Link-time optimization (thin is faster than "fat")
codegen-units = 1       # Better optimization, slower compile
panic = "abort"         # Smaller binary, no unwinding

# ✅ GOOD: Optimizing for binary size
[profile.release-small]
inherits = "release"
opt-level = "z"         # Optimize for size
lto = "fat"             # Aggressive LTO
codegen-units = 1       # Single codegen unit
strip = "symbols"       # Strip symbols
panic = "abort"         # No unwinding

# Usage: cargo build --profile release-small

# ✅ GOOD: Balanced release profile
[profile.release]
opt-level = 3
lto = "thin"            # Good performance, reasonable compile time
codegen-units = 16      # Parallel compilation
strip = "debuginfo"     # Keep symbols, strip debug info

# ✅ GOOD: Fast release builds for testing
[profile.release-fast]
inherits = "release"
debug = false
opt-level = 2           # Fast compilation, good performance
lto = false
codegen-units = 16
```

**Size comparison example:**

```bash
# Default release: ~8MB binary, 2min compile
cargo build --release

# Size-optimized: ~3MB binary, 5min compile
cargo build --profile release-small

# Fast release: ~12MB binary, 1min compile
cargo build --profile release-fast
```

**Rationale**: Different deployment scenarios have different constraints. WebAssembly and embedded systems prioritize size; servers prioritize speed; development workflows need fast compile times. The default release profile is a compromise that may not fit your needs. `lto = "thin"` provides most LTO benefits with much faster compile times than `"fat"`. `codegen-units = 1` enables better optimization but disables parallel codegen. Profile your actual workload—sometimes `opt-level = 2` is faster than `3` due to cache effects. Custom release profiles let you optimize for different scenarios without modifying main release settings.

**See also**: CG-BS-07 (profile configuration), CG-BS-10 (PGO)

---

## CG-A-08: Design CI Pipelines for Cargo Projects

**Strength**: SHOULD

**Summary**: Structure CI to leverage cargo's features and catch issues early.

Effective CI for Rust projects uses cargo's built-in checks and balances speed with coverage.

```yaml
# ✅ GOOD: Comprehensive GitHub Actions CI
name: CI

on:
  push:
    branches: [main]
  pull_request:

env:
  CARGO_TERM_COLOR: always
  RUST_BACKTRACE: 1

jobs:
  check:
    name: Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo check --all-features --locked

  test:
    name: Test Suite
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        rust: [stable, beta]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.rust }}
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --all-features

  fmt:
    name: Rustfmt
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      - run: cargo fmt --all -- --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo clippy --all-features --all-targets -- -D warnings

  doc:
    name: Documentation
    runs-on: ubuntu-latest
    env:
      RUSTDOCFLAGS: -D warnings
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo doc --no-deps --all-features

  coverage:
    name: Code Coverage
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - name: Install cargo-tarpaulin
        run: cargo install cargo-tarpaulin
      - run: cargo tarpaulin --out xml
      - uses: codecov/codecov-action@v3

# ✅ GOOD: Optimized for speed
jobs:
  quick-check:
    name: Quick Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      
      # Fastest feedback
      - name: Check (no features)
        run: cargo check
        
      # Then formatting (fast, fails fast)
      - name: Format check
        run: cargo fmt --all -- --check
      
      # Then clippy (catches more than check)
      - name: Clippy
        run: cargo clippy -- -D warnings
      
      # Finally tests (slowest)
      - name: Test
        run: cargo test

  # Full validation on main branch only
  full-validation:
    if: github.ref == 'refs/heads/main'
    name: Full Validation
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --all-features --all-targets
```

```yaml
# ✅ GOOD: Dependency audit
jobs:
  audit:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rustsec/audit-check@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

**Rationale**: Cargo provides multiple validation steps: `check` (fastest, type-checking only), `clippy` (linting), `test` (correctness), `fmt` (style), and `doc` (documentation). Ordering matters: fast checks should run first to fail fast. `cargo check` is much faster than `cargo build` and catches most errors. Running on multiple OSes catches platform-specific issues. Using `rust-cache` is essential for reasonable CI times. `--locked` ensures reproducible builds from Cargo.lock. Separate quick checks (every PR) from exhaustive validation (main branch only) to balance thoroughness with speed.

**See also**: CG-A-03 (CI caching), CG-A-09 (MSRV testing)

---

## CG-A-09: Test Against Multiple Rust Versions

**Strength**: SHOULD

**Summary**: Test on stable, beta, and MSRV to catch compatibility issues early.

Libraries should work across a range of Rust versions, while applications typically target stable.

```yaml
# ✅ GOOD: Testing multiple Rust versions (libraries)
name: CI

on: [push, pull_request]

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        rust:
          - stable
          - beta
          - nightly
          - 1.75.0  # MSRV
      fail-fast: false  # Continue other versions if one fails
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.rust }}
      - uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.rust }}
      - run: cargo test --all-features
      
      # Allow nightly failures
      - if: matrix.rust == 'nightly'
        continue-on-error: true

# ✅ GOOD: MSRV verification
jobs:
  msrv:
    name: MSRV Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: 1.75.0  # Your MSRV
      - run: cargo check --all-features
      
# ✅ GOOD: Using cargo-msrv for verification
  - name: Verify MSRV
    run: |
      cargo install cargo-msrv
      cargo msrv verify
```

```toml
# ✅ GOOD: Declaring MSRV in Cargo.toml
[package]
name = "my-library"
version = "0.1.0"
rust-version = "1.75.0"  # MSRV declaration

# This is checked by cargo when building
```

```yaml
# ✅ GOOD: Application CI (simpler, stable only)
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --locked
```

**Rationale**: Libraries are used by many projects with different Rust versions. Testing on stable ensures current compatibility; beta catches upcoming breaking changes; nightly provides early warning of future issues. Testing on MSRV (Minimum Supported Rust Version) ensures you don't accidentally use newer features. Applications typically only need to test on stable since you control deployment. The `rust-version` field in Cargo.toml documents and enforces MSRV. Use `cargo-msrv` to find the actual minimum version. Testing nightly can be marked as `continue-on-error` since nightly breakage shouldn't fail your CI.

**See also**: CG-PUB-10 (MSRV management), CG-A-08 (CI design)

---

## CG-A-10: Analyze Build Times with Cargo Timings

**Strength**: CONSIDER

**Summary**: Use `cargo build -Ztimings` to identify compilation bottlenecks.

Cargo's built-in timing analysis helps pinpoint which dependencies and codegen units are slowing down builds.

```bash
# Generate timing report (requires nightly)
cargo +nightly build -Ztimings

# Opens cargo-timing.html in browser with visual timeline

# ✅ GOOD: Analyzing timing report
# Look for:
# 1. Dependencies that take longest to compile
# 2. Codegen units that block other work
# 3. Parallel vs sequential build phases
# 4. Link time proportion
```

```toml
# ✅ GOOD: Optimizing based on timing analysis
# After identifying slow dependencies:

[profile.dev.package.slow-dependency]
opt-level = 2  # Pre-optimize slow dependencies

[profile.dev]
# Adjust based on timing data
split-debuginfo = "unpacked"  # If linking is slow

# ✅ GOOD: Build-time analysis script
```

```bash
#!/bin/bash
# build-analysis.sh

# Clean build with timing
cargo clean
cargo +nightly build -Ztimings

# Compare incremental build
touch src/main.rs
cargo +nightly build -Ztimings

echo "Timing reports generated:"
echo "  cargo-timing.html (clean build)"
echo "  cargo-timing-incremental.html (incremental)"

# Report slowest dependencies
echo ""
echo "Slowest dependencies:"
grep -A 1 "Compile time" cargo-timing.html | \
  sort -rn -k4 | \
  head -10
```

**What to look for in timing reports:**

- **Long sequential chains**: Indicates poor parallelization
- **Thick horizontal bars**: Dependencies that take long to compile
- **Large gap between codegen and link**: Consider alternative linker
- **Many thin bars**: Codegen units might be too fragmented

**Common optimizations based on timing:**

```toml
# If linking dominates (thick bar at end):
[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold"]

# If dependency compilation dominates:
[profile.dev.package.slow-dep-1]
opt-level = 2  # Pre-optimize

[profile.dev.package.slow-dep-2]
opt-level = 2

# If build parallelization is poor:
[build]
jobs = 8  # Explicit job count
```

**Rationale**: Build times accumulate over a project's lifetime, wasting developer time. Cargo timings reveals exactly where time is spent: dependency compilation, codegen, or linking. This data-driven approach prevents premature optimization. The HTML report shows a visual timeline of what compiled in parallel vs sequentially, making bottlenecks obvious. Once you identify slow dependencies, you can pre-optimize them in dev builds or consider alternatives. Tracking timing changes across versions helps catch performance regressions.

**See also**: CG-A-01 (dev optimization), CG-A-02 (linker selection)

---

## CG-A-11: Handle Dependency Updates Strategically

**Strength**: SHOULD

**Summary**: Update dependencies regularly but carefully to balance security with stability.

Dependency updates can introduce breaking changes even in patch versions. A systematic approach minimizes disruption.

```bash
# ❌ BAD: Blind cargo update
cargo update  # Updates all dependencies, may break build

# ✅ GOOD: Targeted updates
cargo update -p serde  # Update specific dependency
cargo update -p serde --precise 1.0.195  # Update to specific version

# ✅ GOOD: Check for outdated dependencies
cargo outdated  # Requires: cargo install cargo-outdated

# Shows:
# Name        Project    Compat   Latest
# serde       1.0.190    1.0.195  1.0.195
# tokio       1.32.0     1.32.0   1.35.1

# ✅ GOOD: Audit for security vulnerabilities
cargo audit  # Requires: cargo install cargo-audit

# Reports known CVEs in dependencies
```

```yaml
# ✅ GOOD: Automated dependency updates (Dependabot)
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "cargo"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      # Group patch updates together
      patch-updates:
        patterns:
          - "*"
        update-types:
          - "patch"
```

```toml
# ✅ GOOD: Pin problematic dependencies
[dependencies]
# Pin dependencies that frequently break changes
serde = "=1.0.190"  # Exact version

# Use ~ for patch-level compatibility
tokio = "~1.32"  # Equivalent to 1.32.x

# Use ^ for semver compatibility (default)
anyhow = "^1.0"  # Compatible with 1.x

# ✅ GOOD: Separate direct and dev dependencies
[dependencies]
# Production dependencies: more conservative
serde = "1.0"

[dev-dependencies]
# Test dependencies: can be more aggressive
criterion = "0.5"  # Latest within 0.5
proptest = "1"
```

```bash
# ✅ GOOD: Update workflow
# 1. Check for updates
cargo outdated

# 2. Check for security issues
cargo audit

# 3. Update conservatively
cargo update -p vulnerable-crate  # Security fixes first

# 4. Test thoroughly
cargo test --all-features

# 5. Check for breaking changes
cargo test --all-targets

# 6. Update Cargo.lock
git add Cargo.lock
git commit -m "Update dependencies"
```

**Rationale**: Dependencies can have security vulnerabilities requiring updates, but updates can also break builds or change behavior. SemVer provides compatibility guarantees, but violations happen. Patch updates (1.0.1 → 1.0.2) should be safe but aren't always. Minor updates (1.0 → 1.1) may add features with bugs. Major updates (1.x → 2.x) are breaking changes. `cargo audit` catches known vulnerabilities automatically. `cargo outdated` shows available updates. Dependabot automates updates with PRs you can review. Grouping patch updates reduces PR noise. Test thoroughly after updates—breakage often appears in edge cases not covered by the dependency's tests.

**See also**: CG-B-05 (version requirements), CG-A-08 (CI testing)

---

## CG-A-12: Diagnose and Fix Future Incompatibility Warnings

**Strength**: SHOULD

**Summary**: Address future incompatibility reports to prevent breakage on Rust updates.

Cargo tracks breaking changes coming in future Rust versions and reports which dependencies will be affected.

```bash
# ✅ GOOD: Check for future incompatibilities
cargo report future-incompatibilities --color always

# Example output:
# warning: the following packages contain code that will be rejected by a future version of Rust
#   - old-crate v0.1.0
#     - method `foo` is never used (warn(dead_code) will be deny(dead_code))
#     - this will become a hard error in Rust 1.75.0

# ✅ GOOD: Generate detailed report
cargo report future-incompat --id 1

# Shows specific warnings and affected code
```

```toml
# ✅ GOOD: Proactive future-incompat checking in CI
# .github/workflows/ci.yml
- name: Check future compatibility
  run: |
    cargo check --all-features 2>&1 | tee build.log
    if cargo report future-incompatibilities 2>&1 | grep -q "future incompatibilities"; then
      echo "⚠️ Future incompatibility warnings found"
      cargo report future-incompatibilities
      exit 1
    fi
```

```bash
# ❌ BAD: Ignoring warnings
cargo build
# Warnings about future incompatibilities scroll by
# Later: build breaks on new Rust version

# ✅ GOOD: Addressing warnings proactively
# 1. Identify affected dependencies
cargo report future-incompatibilities

# 2. Check if updates available
cargo outdated

# 3. Update affected dependencies
cargo update -p affected-crate

# 4. If no update, file issue with upstream
# Or consider alternative dependencies
```

```rust
// ✅ GOOD: Common future incompatibility fixes

// Old code (will break):
trait MyTrait {
    fn method(&self) -> u32;
}

impl MyTrait for () {
    fn method(&self) -> u32 { 0 }
}

// Warning: trait method is never used

// Fix: Actually use the method or remove trait
impl () {
    fn method(&self) -> u32 { 0 }  // Direct impl instead
}
```

**When future incompatibilities appear:**

1. **Your code**: Fix immediately—you control this
2. **Direct dependencies**: Update or file issues
3. **Transitive dependencies**: Update top-level deps to get new transitives
4. **Abandoned dependencies**: Consider alternatives
5. **Upstream won't fix**: Consider forking or vendoring

**Rationale**: Rust evolves, and some changes are breaking. The Rust team provides advance warning through future incompatibility lints, giving you time to adapt before code actually breaks. These warnings indicate that your dependencies (or your code) use patterns that will be errors in future Rust versions. Addressing them proactively prevents surprise breakage during routine Rust updates. Most issues are in dependencies, not your code—updating dependencies usually fixes them. For abandoned dependencies, you may need to switch to maintained alternatives.

**See also**: CG-A-11 (dependency updates), CG-A-09 (version testing)

---

## Best Practices Summary

| Pattern | Key Takeaway | Strength |
|---------|-------------|----------|
| CG-A-01 | Reduce debug info for faster dev builds | SHOULD |
| CG-A-02 | Use alternative linkers (mold, lld, zld) | SHOULD |
| CG-A-03 | Cache cargo artifacts correctly in CI | MUST |
| CG-A-04 | Understand incremental compilation trade-offs | CONSIDER |
| CG-A-05 | Use workspace feature unification carefully | CONSIDER |
| CG-A-06 | Leverage unstable features responsibly | CONSIDER |
| CG-A-07 | Optimize release profiles for your needs | SHOULD |
| CG-A-08 | Design effective CI pipelines | SHOULD |
| CG-A-09 | Test against multiple Rust versions | SHOULD |
| CG-A-10 | Analyze build times with cargo timings | CONSIDER |
| CG-A-11 | Update dependencies strategically | SHOULD |
| CG-A-12 | Fix future incompatibility warnings | SHOULD |

## Related Guidelines

- **01-cargo-basics.md**: Foundation for understanding cargo's dependency system
- **02-cargo-build-system.md**: Build scripts and feature configuration
- **05-cargo-configuration.md**: Configuration hierarchy and environment variables

## External References

- [Cargo Book: Optimizing Build Performance](https://doc.rust-lang.org/cargo/guide/build-cache.html)
- [Cargo Book: Continuous Integration](https://doc.rust-lang.org/cargo/guide/continuous-integration.html)
- [Cargo Timings Documentation](https://doc.rust-lang.org/nightly/cargo/reference/timings.html)
- [rust-cache GitHub Action](https://github.com/Swatinem/rust-cache)
- [cargo-audit](https://github.com/rustsec/rustsec/tree/main/cargo-audit)
- [Minimum Supported Rust Version](https://rust-lang.github.io/rfcs/2495-min-rust-version.html)
