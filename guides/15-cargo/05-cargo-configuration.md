# Cargo Configuration

Essential patterns for Cargo configuration through .cargo/config.toml, environment variables, and command-line options. These patterns enable project-specific and user-specific customization.

---

## CG-CF-01: Understand Configuration Hierarchy

**Strength**: MUST

**Summary**: Configuration is loaded hierarchically from project up to home directory, with specific precedence rules.

```bash
# Cargo searches for config files in this order (higher = higher precedence):
# 1. Command line: --config KEY=VALUE
# 2. Environment variables: CARGO_BUILD_JOBS=4
# 3. Project configs (searched upward from pwd):
#    /my/project/.cargo/config.toml
#    /my/.cargo/config.toml
#    /.cargo/config.toml
# 4. User config: $CARGO_HOME/config.toml (~/.cargo/config.toml)

# Example hierarchy:
# project/
# ├── .cargo/
# │   └── config.toml      # Project-specific (highest priority)
# └── subdir/
#     └── .cargo/
#         └── config.toml  # NOT read if cargo invoked from project/

# ~/.cargo/config.toml     # User-specific (lowest priority)

# ✅ GOOD: Project config for team settings
# project/.cargo/config.toml
[build]
target = "x86_64-unknown-linux-gnu"
rustflags = ["-C", "link-arg=-fuse-ld=lld"]

[alias]
b = "build"
t = "test --all-features"

# ✅ GOOD: User config for personal preferences
# ~/.cargo/config.toml
[build]
jobs = 8                    # Your machine has 8 cores

[term]
color = "always"            # You always want colored output

# ❌ BAD: Sensitive data in project config
# project/.cargo/config.toml
[registries.company]
token = "secret-token"      # NEVER! Use environment variables
# Better: CARGO_REGISTRIES_COMPANY_TOKEN=secret-token
```

**Rationale**: The hierarchy enables both team conventions (project config) and personal preferences (user config). Command-line and environment variables override files for CI flexibility. Understanding precedence prevents confusion when settings don't apply as expected.

**See also**: CG-CF-02 (environment variables), CG-CF-03 (project vs user config)

---

## CG-CF-02: Use Environment Variables for Dynamic Configuration

**Strength**: SHOULD

**Summary**: Use environment variables for CI/CD, secrets, and machine-specific overrides.

```bash
# ✅ GOOD: CI/CD configuration
export CARGO_INCREMENTAL=0              # Disable incremental in CI
export CARGO_TERM_COLOR=always          # Force colors in CI logs
export CARGO_BUILD_JOBS=2               # Limit parallelism on CI
export CARGO_NET_GIT_FETCH_WITH_CLI=true

# ✅ GOOD: Secrets and tokens
export CARGO_REGISTRY_TOKEN="$SECRET"       # crates.io token
export CARGO_REGISTRIES_COMPANY_TOKEN="$SECRET"  # Private registry

# ✅ GOOD: Per-command override
CARGO_BUILD_TARGET=wasm32-unknown-unknown cargo build

# Environment variable naming: CARGO_<SECTION>_<KEY>
# Example conversions:
# build.jobs          → CARGO_BUILD_JOBS
# build.target-dir    → CARGO_BUILD_TARGET_DIR
# target.x86_64-unknown-linux-gnu.runner → CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_RUNNER

# ❌ BAD: Hardcoding machine-specific paths
# ~/.cargo/config.toml
[build]
rustc = "/home/alice/.cargo/bin/rustc"  # Breaks on other machines!

# ✅ BETTER: Use environment variable
export CARGO_BUILD_RUSTC="$HOME/.cargo/bin/rustc"
```

**Rationale**: Environment variables override config files, perfect for CI where you can't modify .cargo/config.toml. They're also the only secure way to handle credentials. Use them for machine-specific, user-specific, or build-specific overrides.

**See also**: CG-CF-01 (hierarchy), CG-CF-08 (CI configuration)

---

## CG-CF-03: Choose Project vs User Config Appropriately

**Strength**: MUST

**Summary**: Use project config for team settings, user config for personal preferences.

```toml
# ✅ GOOD: project/.cargo/config.toml (commit to git)
[build]
target = "x86_64-unknown-linux-gnu"     # Team standard
rustflags = ["-C", "link-arg=-fuse-ld=lld"]  # Team uses lld

[alias]
check-all = "check --workspace --all-features"
test-all = "test --workspace --all-features"

[target.x86_64-unknown-linux-gnu]
runner = "qemu-system-x86_64"           # Project uses QEMU

# Keep team conventions here

# ✅ GOOD: ~/.cargo/config.toml (personal, not committed)
[build]
jobs = 16                               # Your machine specs

[cargo-new]
vcs = "git"                             # Your preference

[term]
color = "always"                        # Your terminal
verbose = false                         # You prefer terse output

# Keep personal preferences here

# ❌ BAD: Personal preferences in project config
# project/.cargo/config.toml (committed)
[build]
jobs = 4                                # Forces everyone to 4 jobs!
# Different machines have different capabilities

# ❌ BAD: Team settings in user config
# ~/.cargo/config.toml
[build]
target = "x86_64-pc-windows-gnu"        # Only you want Windows target
# Confuses you when working on different projects
```

**Rationale**: Project config is shared via version control - use for team conventions. User config is personal - use for machine-specific or preference settings. Mixing them causes confusion and conflicts.

**See also**: CG-CF-01 (hierarchy), CG-CF-04 (what to configure)

---

## CG-CF-04: Configure Common Build Settings Wisely

**Strength**: SHOULD

**Summary**: Configure build settings to optimize for your development workflow.

```toml
# ✅ GOOD: Development-optimized config
# project/.cargo/config.toml
[build]
jobs = 4                                # Reasonable default

[profile.dev]
split-debuginfo = "unpacked"            # Faster linking on macOS/Linux
debug = "line-tables-only"              # Faster compile, some debug info

[profile.dev.package."*"]
opt-level = 2                           # Optimize dependencies
debug = false                           # Skip debug for dependencies

# ✅ GOOD: Cross-compilation setup
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
runner = "qemu-aarch64"

[target.wasm32-unknown-unknown]
runner = "wasm-pack test"

# ✅ GOOD: Custom linker for faster linking
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=mold"]
# Or: linker = "clang"
#     rustflags = ["-C", "link-arg=-fuse-ld=mold"]

# ❌ BAD: Overly aggressive optimization in dev
[profile.dev]
opt-level = 3                           # Slow dev builds!
lto = true                              # Very slow linking!

# ❌ BAD: Hardcoded paths that don't work for others
[target.x86_64-unknown-linux-gnu]
runner = "/home/alice/bin/my-runner"    # Only works for Alice!
```

**Rationale**: Build settings dramatically affect developer experience. Optimize for iteration speed in development. Use target-specific settings for cross-compilation. Avoid hardcoded paths that break portability.

**See also**: CG-BS-06 (dev profile), CG-CF-07 (cross-compilation)

---

## CG-CF-05: Use Command Aliases for Common Workflows

**Strength**: SHOULD

**Summary**: Define aliases for frequently-used command combinations to save typing.

```toml
# ✅ GOOD: Useful aliases
[alias]
b = "build"
c = "check"
t = "test"
r = "run"

# Workspace operations
cw = "check --workspace"
tw = "test --workspace"
bw = "build --workspace"

# All features
ca = "check --all-features"
ta = "test --all-features"

# Release checks
cr = "check --release"
br = "build --release"

# Common combinations
check-all = "check --workspace --all-features"
test-all = "test --workspace --all-features --"
clippy-all = "clippy --workspace --all-features -- -D warnings"

# CI-style check
ci = "clippy --all-targets --all-features -- -D warnings"

# Custom workflows
prebuild = ["fmt", "--check", "&&", "clippy", "--", "-D", "warnings"]

# ✅ GOOD: Project-specific aliases
# For a web server project:
dev = "run --features dev-server"
prod = "run --release --features production"

# ❌ BAD: Aliasing built-in commands (not allowed)
[alias]
build = "build --release"               # Error: can't redefine built-in

# ❌ BAD: Too clever/confusing aliases
[alias]
go = "run"                              # Unclear
x = "test"                              # Too terse
🚀 = "build --release"                  # Emoji aliases are confusing
```

**Rationale**: Aliases reduce typing for frequent operations. They can encode team workflows and conventions. Keep them mnemonic and not too clever. Document complex aliases in project README.

**See also**: CG-CF-03 (project config), alias documentation

---

## CG-CF-06: Configure Target-Specific Settings

**Strength**: SHOULD

**Summary**: Use `[target.<triple>]` for platform-specific linkers, runners, and flags.

```toml
# ✅ GOOD: Cross-compilation targets
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
runner = "qemu-aarch64 -L /usr/aarch64-linux-gnu"

[target.x86_64-pc-windows-gnu]
linker = "x86_64-w64-mingw32-gcc"
runner = "wine"

[target.wasm32-unknown-unknown]
runner = "wasm-pack test --node"

# ✅ GOOD: Platform-specific optimizations
[target.x86_64-unknown-linux-gnu]
rustflags = [
    "-C", "link-arg=-fuse-ld=mold",     # Fast linker
    "-C", "target-cpu=native",          # Optimize for this CPU
]

# ✅ GOOD: Embedded targets
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "probe-run --chip STM32F103C8"
rustflags = [
    "-C", "link-arg=-Tlink.x",
    "-C", "linker=rust-lld",
]

# ✅ GOOD: Test runner wrapper
[target.x86_64-unknown-linux-gnu]
runner = ["valgrind", "--leak-check=full"]

# ❌ BAD: Global rustflags affecting all targets
[build]
rustflags = ["-C", "target-cpu=native"]  # Breaks cross-compilation!

# ✅ BETTER: Target-specific
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "target-cpu=native"]  # Only for native builds
```

**Rationale**: Target-specific configuration enables cross-compilation and platform-specific optimizations. Separating by target prevents conflicts when building for multiple platforms. Use cfg() syntax for flexible target matching.

**See also**: CG-CF-07 (cross-compilation), target configuration

---

## CG-CF-07: Set Up Cross-Compilation Correctly

**Strength**: SHOULD

**Summary**: Configure linkers, sysroots, and runners for each cross-compilation target.

```toml
# ✅ GOOD: Complete cross-compilation setup
# Install cross-compilation tools first:
# $ sudo apt install gcc-aarch64-linux-gnu qemu-user

[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
runner = "qemu-aarch64 -L /usr/aarch64-linux-gnu"

[target.armv7-unknown-linux-gnueabihf]
linker = "arm-linux-gnueabihf-gcc"
runner = "qemu-arm -L /usr/arm-linux-gnueabihf"

# ✅ GOOD: Using cross tool (easier setup)
# Install: cargo install cross
# Then just use: cross build --target aarch64-unknown-linux-gnu
# cross handles toolchain and Docker automatically

# ✅ GOOD: Windows cross-compilation from Linux
[target.x86_64-pc-windows-gnu]
linker = "x86_64-w64-mingw32-gcc"
runner = "wine"

[target.i686-pc-windows-gnu]
linker = "i686-w64-mingw32-gcc"

# ✅ GOOD: macOS cross-compilation
[target.x86_64-apple-darwin]
linker = "x86_64-apple-darwin-clang"
ar = "x86_64-apple-darwin-ar"

# ❌ BAD: Missing linker configuration
# Just setting target without linker config
$ cargo build --target aarch64-unknown-linux-gnu
# Error: linker `cc` not found

# ❌ BAD: Hardcoded absolute paths
[target.aarch64-unknown-linux-gnu]
linker = "/home/user/tools/gcc"         # Not portable!
```

**Rationale**: Cross-compilation requires proper toolchain configuration. Each target needs appropriate linker, potentially a runner for testing, and possibly additional flags. Use the `cross` tool for simpler setup, or configure manually for fine control.

**See also**: CG-CF-06 (target-specific), cross tool

---

## CG-CF-08: Optimize Configuration for CI

**Strength**: SHOULD

**Summary**: Use CI-specific configuration via environment variables to optimize build performance and reliability.

```yaml
# ✅ GOOD: GitHub Actions CI configuration
# .github/workflows/ci.yml
name: CI

env:
  CARGO_TERM_COLOR: always              # Colored output in logs
  CARGO_INCREMENTAL: 0                  # No incremental (no cache benefit)
  RUST_BACKTRACE: 1                     # Full backtraces on panic
  
  # Optional: Faster crates.io protocol
  # CARGO_REGISTRIES_CRATES_IO_PROTOCOL: sparse

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: dtolnay/rust-toolchain@stable
      
      - name: Cache cargo registry
        uses: actions/cache@v3
        with:
          path: ~/.cargo/registry
          key: ${{ runner.os }}-cargo-registry-${{ hashFiles('**/Cargo.lock') }}
      
      - name: Cache cargo index
        uses: actions/cache@v3
        with:
          path: ~/.cargo/git
          key: ${{ runner.os }}-cargo-index-${{ hashFiles('**/Cargo.lock') }}
      
      - name: Cache cargo build
        uses: actions/cache@v3
        with:
          path: target
          key: ${{ runner.os }}-cargo-build-target-${{ hashFiles('**/Cargo.lock') }}
      
      - run: cargo test --all-features
      - run: cargo clippy --all-features -- -D warnings

# ✅ GOOD: GitLab CI with environment config
# .gitlab-ci.yml
variables:
  CARGO_TERM_COLOR: "always"
  CARGO_INCREMENTAL: "0"
  CARGO_HOME: "${CI_PROJECT_DIR}/.cargo"

cache:
  paths:
    - .cargo/
    - target/

test:
  script:
    - cargo test --all-features

# ❌ BAD: Not disabling incremental in CI
# Wastes time and space - no reuse between runs
# (If not set, defaults to enabled)
```

**Rationale**: CI environments differ from development. Disable incremental compilation (no benefit across runs). Enable colors for readable logs. Cache appropriately. Use environment variables since you can't easily modify .cargo/config.toml in CI.

**See also**: CG-CF-02 (environment variables), CI documentation

---

## CG-CF-09: Use --config for Temporary Overrides

**Strength**: CONSIDER

**Summary**: Use command-line --config for one-off overrides without modifying files.

```bash
# ✅ GOOD: Temporary profile override
$ cargo build --config profile.dev.opt-level=3

# ✅ GOOD: Temporary network configuration
$ cargo build --config net.git-fetch-with-cli=true

# ✅ GOOD: Temporary target
$ cargo build --config 'build.target="wasm32-unknown-unknown"'

# ✅ GOOD: Multiple overrides
$ cargo build \
    --config profile.dev.opt-level=2 \
    --config 'build.rustflags=["-C", "target-cpu=native"]'

# ✅ GOOD: Using a config file
$ cargo build --config ./custom-config.toml

# ✅ GOOD: Testing with different configurations
$ cargo test --config profile.dev.package.image.opt-level=3

# ❌ BAD: Using --config for regular builds
$ cargo build --config profile.dev.opt-level=2  # Every time!
# Just put this in .cargo/config.toml instead
```

**Rationale**: `--config` is perfect for experiments and one-off builds. It's more convenient than temporarily editing config files. However, for regular builds, put configuration in .cargo/config.toml. Command-line overrides have highest precedence.

**See also**: CG-CF-01 (precedence), --config documentation

---

## CG-CF-10: Handle Credentials Securely

**Strength**: MUST

**Summary**: Never commit credentials; use environment variables or credential helpers.

```toml
# ❌ NEVER: Credentials in config files
# .cargo/config.toml (committed to git)
[registry]
token = "abc123secret"                   # SECURITY RISK!

[registries.company]
token = "company-secret-token"           # ANYONE CAN SEE THIS!

# ✅ GOOD: Use environment variables
# .github/workflows/publish.yml
env:
  CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_TOKEN }}

# Or locally:
$ export CARGO_REGISTRY_TOKEN="secret-from-password-manager"
$ cargo publish

# ✅ GOOD: Use credentials.toml (never commit this!)
# ~/.cargo/credentials.toml (in .gitignore)
[registry]
token = "abc123secret"

# Created automatically by:
$ cargo login

# ✅ GOOD: For organization use credential providers
# .cargo/config.toml (committed - no secrets!)
[registry]
credential-provider = "cargo:token-from-vault"

# Then set up vault in CI/CD environment

# ✅ GOOD: .gitignore credentials
# .gitignore
.cargo/credentials
.cargo/credentials.toml

# ❌ BAD: Committing credentials by accident
$ git add .
$ git commit -m "Update config"
# Oops! Committed .cargo/credentials.toml
# Must: Revoke token, remove from history, regenerate
```

**Rationale**: Committed credentials are a critical security vulnerability. Anyone with repository access gets your credentials. Use environment variables in CI, credentials.toml locally (never commit), or credential providers for organizations.

**See also**: CG-CF-02 (environment variables), credential providers

---

## CG-CF-11: Configure Network Settings for Reliability

**Strength**: CONSIDER

**Summary**: Configure network timeouts, retries, and proxies for reliable dependency fetching.

```toml
# ✅ GOOD: Network configuration
[http]
timeout = 30                            # Seconds per HTTP request
low-speed-limit = 10                    # Min bytes/sec before timeout
check-revoke = true                     # Check SSL certificate revocation
multiplexing = true                     # HTTP/2 (default)
cainfo = "/path/to/custom/ca-cert.pem"  # Custom CA bundle

[http.proxy]
# Proxy configuration
proxy = "http://proxy.company.com:8080"

[net]
retry = 3                               # Number of times to retry failed requests
git-fetch-with-cli = true               # Use git CLI instead of libgit2
offline = false                         # Allow network access

# ✅ GOOD: Corporate environment
[http]
proxy = "http://corporate-proxy:8080"
cainfo = "/etc/ssl/certs/corporate-ca.pem"

[net]
git-fetch-with-cli = true               # Better proxy support

# ✅ GOOD: Unreliable network
[http]
timeout = 60                            # Longer timeout
low-speed-limit = 5                     # More patient

[net]
retry = 5                               # More retries

# ✅ GOOD: Offline development
[net]
offline = true                          # Fail fast if network needed
# Or: cargo build --offline

# ❌ BAD: Too aggressive timeouts
[http]
timeout = 5                             # Too short for large crates!
low-speed-limit = 100                   # Too high - times out on slow networks
```

**Rationale**: Network issues are common. Configure timeouts and retries for your environment. Corporate networks often need proxy and custom CA configuration. Use `offline = true` or `--offline` when network should be impossible.

**See also**: http configuration, net configuration

---

## CG-CF-12: Document Project Configuration

**Strength**: SHOULD

**Summary**: Document project-specific configuration in README or CONTRIBUTING guide.

```markdown
# ✅ GOOD: Document in README.md

## Development Setup

This project uses custom Cargo configuration in `.cargo/config.toml`:

### Build Configuration
- **Linker**: Uses `lld` for faster linking
  - Install: `sudo apt install lld` (Linux) or `brew install llvm` (macOS)
- **Target**: Defaults to `x86_64-unknown-linux-gnu`

### Aliases
- `cargo ca`: Check with all features
- `cargo ta`: Test with all features  
- `cargo ci`: Full CI-style checks (clippy + tests)

### Cross-Compilation
To build for ARM:
```bash
cargo build --target aarch64-unknown-linux-gnu
```

Requires: `sudo apt install gcc-aarch64-linux-gnu`

### Environment Variables
Optional environment overrides:
- `CARGO_BUILD_JOBS`: Parallel compilation jobs (default: CPU count)
- `CARGO_BUILD_TARGET_DIR`: Custom target directory

---

```

```toml
# ✅ GOOD: Comment in config itself
# .cargo/config.toml

# Team standard linker - provides 2-3x faster linking
# Install: sudo apt install lld (Linux) or brew install llvm (macOS)
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=lld"]

# Common workflow aliases
[alias]
# ca = check all: check with all features
ca = "check --all-features"
# ta = test all: test with all features
ta = "test --all-features"

# ❌ BAD: Undocumented mysterious configuration
# .cargo/config.toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-Wl,-plugin-opt=thinlto"]
# What does this do? Why? When was it added?
```

**Rationale**: Configuration affects everyone on the team. Document requirements (like installing lld), explain aliases, and clarify why settings exist. This helps onboarding and prevents confusion when builds behave unexpectedly.

**See also**: CG-CF-03 (project config), CG-CF-05 (aliases)

---

## Best Practices Summary

### Quick Reference Table

| Pattern | Strength | Key Insight |
|---------|----------|-------------|
| Configuration hierarchy | MUST | Command-line > env vars > project config > user config |
| Environment variables | SHOULD | Perfect for CI/CD, secrets, and machine-specific overrides |
| Project vs user config | MUST | Project=team settings (commit), user=personal (don't commit) |
| Common build settings | SHOULD | Optimize for dev workflow, avoid breaking portability |
| Command aliases | SHOULD | Save typing for common workflows, keep mnemonic |
| Target-specific settings | SHOULD | Use [target.<triple>] for cross-compilation |
| Cross-compilation | SHOULD | Configure linker, runner, flags per target |
| CI optimization | SHOULD | Disable incremental, use env vars, cache wisely |
| Temporary overrides | CONSIDER | Use --config for one-offs, not regular builds |
| Secure credentials | MUST | Never commit tokens, use env vars or credentials.toml |
| Network configuration | CONSIDER | Configure timeouts, retries, proxies as needed |
| Document configuration | SHOULD | Explain custom config in README/CONTRIBUTING |

---

## Related Guidelines

- **Basics**: See `01-cargo-basics.md` for workspace configuration
- **Build System**: See `02-cargo-build-system.md` for profile configuration
- **Publishing**: See `04-cargo-publishing.md` for registry configuration
- **Advanced**: See `06-cargo-advanced.md` for optimization techniques

---

## External References

- [Cargo Book - Configuration](https://doc.rust-lang.org/cargo/reference/config.html)
- [Cargo Book - Environment Variables](https://doc.rust-lang.org/cargo/reference/environment-variables.html)
- [Cargo Book - Build Cache](https://doc.rust-lang.org/cargo/guide/build-cache.html)
- [cross tool](https://github.com/cross-rs/cross)
