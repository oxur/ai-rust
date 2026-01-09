# CLI Distribution and Packaging

Patterns for distributing and packaging command-line tools.

---

## CLI-39: Binary Size Optimization

**Strength**: CONSIDER

**Summary**: Optimize binary size for faster downloads and distribution using release profile settings.

```toml
# ✅ GOOD: Optimized release profile
[profile.release]
opt-level = "z"     # Optimize for size
lto = true          # Link-time optimization
codegen-units = 1   # Better optimization, slower compile
strip = true        # Strip debug symbols
panic = "abort"     # Smaller binary, no unwinding

# ✅ GOOD: Separate profiles for different goals
[profile.release]
# Default: balanced
opt-level = 3
lto = "thin"
codegen-units = 16

[profile.release-small]
inherits = "release"
opt-level = "z"
lto = true
codegen-units = 1
strip = true
panic = "abort"

[profile.release-fast]
inherits = "release"
opt-level = 3
lto = "fat"
codegen-units = 1

# ✅ GOOD: Feature flags to reduce dependencies
[features]
default = ["colors", "progress"]
colors = ["dep:colored"]
progress = ["dep:indicatif"]
all = ["colors", "progress", "completions"]

# ❌ BAD: No optimization
[profile.release]
# Using defaults, binary will be unnecessarily large
```

**Build commands**:
```bash
# Standard release build
cargo build --release

# Size-optimized build
cargo build --profile release-small

# Build without optional features
cargo build --release --no-default-features
```

**Size comparison** (typical results):
- Debug build: 15-30 MB
- Release build (default): 3-5 MB
- Release build (opt-level="z", lto, strip): 1-2 MB
- No default features: Save 500KB-1MB

**Additional size optimizations**:
```toml
# Remove debugging info completely
[profile.release]
debug = false

# Use system allocator instead of jemalloc
[dependencies]
# Don't add jemalloc unless needed
```

**Rationale**: Smaller binaries download faster, use less disk space, and start faster. For CLI tools distributed to many users, size optimization is worthwhile.

**See also**:
- CLI-40: Release profile configuration
- [External: Minimizing Rust Binary Size](https://github.com/johnthagen/min-sized-rust)

---

## CLI-40: Release Profile Configuration

**Strength**: SHOULD

**Summary**: Configure release profiles for optimal performance and binary characteristics.

```toml
# ✅ GOOD: Production-ready release profile
[profile.release]
opt-level = 3           # Maximum optimization
lto = true              # Link-time optimization
codegen-units = 1       # Better optimization
strip = true            # Remove debug symbols
panic = "abort"         # Simpler panic handling
overflow-checks = true  # Keep overflow checks even in release

# ✅ GOOD: Profile for development testing
[profile.release-dev]
inherits = "release"
debug = true            # Keep debug symbols
strip = false           # Don't strip
opt-level = 2           # Faster compilation
lto = false

# ✅ GOOD: Profile for benchmarking
[profile.bench]
inherits = "release"
debug = true            # Keep symbols for profiling
lto = "thin"
```

**Profile settings explained**:
- `opt-level`: 0 (none), 1, 2, 3 (max), "s" (size), "z" (size aggressive)
- `lto`: false, true, "thin", "fat" - Link-time optimization
- `codegen-units`: 1-256 - Parallelism vs optimization trade-off
- `strip`: true/false - Remove debug symbols
- `panic`: "unwind" or "abort" - Panic behavior

**Rationale**: Proper release configuration balances size, performance, and compilation time. Different profiles serve different needs.

**See also**:
- CLI-39: Binary size optimization
- [External: Cargo Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html)

---

## CLI-41: Cross-Compilation Considerations

**Strength**: CONSIDER

**Summary**: Design for cross-compilation and document platform-specific requirements.

```rust
// ✅ GOOD: Platform-specific code with cfg
#[cfg(unix)]
fn get_default_shell() -> String {
    std::env::var("SHELL").unwrap_or_else(|_| "/bin/sh".to_string())
}

#[cfg(windows)]
fn get_default_shell() -> String {
    std::env::var("COMSPEC").unwrap_or_else(|_| "cmd.exe".to_string())
}

// ✅ GOOD: Handle path separators correctly
use std::path::{Path, PathBuf, MAIN_SEPARATOR};

fn build_path(base: &Path, relative: &str) -> PathBuf {
    // Use join() instead of string concatenation
    base.join(relative)
}

// ✅ GOOD: Document platform requirements
/// # Platform Notes
///
/// - **Linux**: Requires glibc 2.17 or later
/// - **macOS**: Requires macOS 10.12 or later
/// - **Windows**: Requires Windows 7 or later
///
/// ## Dependencies
///
/// - **Linux**: `libssl-dev`, `pkg-config`
/// - **macOS**: Included in Xcode Command Line Tools
/// - **Windows**: No additional dependencies

// ✅ GOOD: Use cross for cross-compilation
// In .github/workflows/release.yml
/*
- name: Build for Linux
  run: cargo build --release --target x86_64-unknown-linux-gnu

- name: Build for macOS
  run: cargo build --release --target x86_64-apple-darwin

- name: Build for Windows
  run: cargo build --release --target x86_64-pc-windows-msvc
*/

// ❌ BAD: Hardcoded Unix paths
fn get_config_path() -> PathBuf {
    PathBuf::from("/home/user/.config/my-tool/config.toml")
    // Won't work on Windows or other users!
}

// ❌ BAD: Unix-specific assumptions
fn run_command() {
    std::process::Command::new("sh")
        .arg("-c")
        .arg("echo hello")
        .spawn()
        .unwrap();
    // No 'sh' on Windows!
}
```

**Cross-compilation targets**:
```bash
# Install targets
rustup target add x86_64-unknown-linux-gnu
rustup target add x86_64-unknown-linux-musl  # Static binary
rustup target add x86_64-apple-darwin
rustup target add x86_64-pc-windows-msvc
rustup target add aarch64-apple-darwin       # Apple Silicon

# Build for specific target
cargo build --release --target x86_64-unknown-linux-musl
```

**Using cross for easier cross-compilation**:
```bash
# Install cross
cargo install cross

# Build for different platforms
cross build --release --target x86_64-unknown-linux-gnu
cross build --release --target x86_64-pc-windows-gnu
```

**Rationale**: Users run many different platforms. Supporting cross-compilation and documenting platform requirements improves accessibility.

**See also**:
- CLI-47: Cross-platform considerations
- [External: cross tool](https://github.com/cross-rs/cross)
- [External: Platform Support](https://doc.rust-lang.org/nightly/rustc/platform-support.html)

---

## CLI-42: cargo-binstall Compatibility

**Strength**: CONSIDER

**Summary**: Make your tool compatible with `cargo-binstall` for faster installation from pre-built binaries.

```toml
# ✅ GOOD: Package metadata for cargo-binstall
[package]
name = "my-tool"
version = "1.0.0"
description = "A useful CLI tool"
repository = "https://github.com/username/my-tool"
# binstall looks for releases at: 
# {repository}/releases/download/v{version}/{name}-{version}-{target}.tar.gz

# ✅ GOOD: Add metadata in Cargo.toml
[package.metadata.binstall]
# Override default binary name if needed
pkg-name = "my-tool"
bin-dir = "{ bin }{ binary-ext }"
# Disable fallback to source build
no-fallback = false

[package.metadata.binstall.overrides.x86_64-pc-windows-msvc]
pkg-fmt = "zip"
```

**GitHub Releases structure for binstall**:
```
# Expected release artifact names:
my-tool-v1.0.0-x86_64-unknown-linux-gnu.tar.gz
my-tool-v1.0.0-x86_64-unknown-linux-musl.tar.gz
my-tool-v1.0.0-x86_64-apple-darwin.tar.gz
my-tool-v1.0.0-x86_64-pc-windows-msvc.zip

# Archive contents:
my-tool          (or my-tool.exe on Windows)
LICENSE
README.md
```

**CI/CD for releases** (GitHub Actions):
```yaml
# .github/workflows/release.yml
name: Release

on:
  release:
    types: [created]

jobs:
  build:
    strategy:
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            os: ubuntu-latest
          - target: x86_64-apple-darwin
            os: macos-latest
          - target: x86_64-pc-windows-msvc
            os: windows-latest
    
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Build
        run: cargo build --release --target ${{ matrix.target }}
      
      - name: Package
        run: |
          cd target/${{ matrix.target }}/release
          tar czf ../../../my-tool-${{ github.ref_name }}-${{ matrix.target }}.tar.gz my-tool
      
      - name: Upload Release Asset
        uses: actions/upload-release-asset@v1
        with:
          upload_url: ${{ github.event.release.upload_url }}
          asset_path: ./my-tool-${{ github.ref_name }}-${{ matrix.target }}.tar.gz
          asset_name: my-tool-${{ github.ref_name }}-${{ matrix.target }}.tar.gz
```

**Installation methods**:
```bash
# With cargo-binstall (fast, uses pre-built binaries)
cargo binstall my-tool

# Fallback to source build
cargo install my-tool

# From GitHub releases
wget https://github.com/user/my-tool/releases/download/v1.0.0/my-tool-v1.0.0-x86_64-unknown-linux-gnu.tar.gz
tar xzf my-tool-v1.0.0-x86_64-unknown-linux-gnu.tar.gz
sudo mv my-tool /usr/local/bin/
```

**Rationale**: Pre-built binaries install much faster than compiling from source. Supporting `cargo-binstall` improves user experience.

**See also**:
- CLI-39: Binary size optimization
- [External: cargo-binstall](https://github.com/cargo-bins/cargo-binstall)
- [External: cargo-dist](https://github.com/axodotdev/cargo-dist) - Distribution tool

---

## Best Practices Summary

| Pattern | Strength | Key Takeaway |
|---------|----------|--------------|
| CLI-39 | CONSIDER | Optimize binary size with profile settings |
| CLI-40 | SHOULD | Configure release profiles appropriately |
| CLI-41 | CONSIDER | Design for cross-compilation, document requirements |
| CLI-42 | CONSIDER | Make tool compatible with cargo-binstall |

## Related Guidelines

- CLI-47: Cross-platform considerations (in 08-advanced-topics.md)

## External References

- [Minimizing Rust Binary Size](https://github.com/johnthagen/min-sized-rust)
- [Cargo Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html)
- [cross tool](https://github.com/cross-rs/cross) - Easy cross-compilation
- [cargo-binstall](https://github.com/cargo-bins/cargo-binstall) - Binary installation
- [cargo-dist](https://github.com/axodotdev/cargo-dist) - Distribution tool
