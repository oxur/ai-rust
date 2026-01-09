# Cargo Build System

Advanced Cargo patterns for features, profiles, build scripts, and optimization. These patterns enable fine-grained control over compilation behavior and project configurability.

---

## CG-BS-01: Design Features for Additivity

**Strength**: MUST

**Summary**: Features should be additive-only - enabling a feature should never disable functionality.

```toml
# ✅ GOOD: Additive features
[features]
default = ["std"]
std = []                    # Enables std library support
serde = ["dep:serde"]       # Adds serde support
async = ["dep:tokio"]       # Adds async runtime
full = ["serde", "async"]   # Combines multiple features

# Each feature ADDS capability, never removes it

# ❌ BAD: Non-additive features
[features]
std = []
no-std = []  # Anti-feature! Enabling this disables std
# Violates additivity principle - features should only add, never subtract

# ❌ BAD: Mutually exclusive features
[features]
backend-a = []
backend-b = []  # Can't have both? Not additive!
# If exclusive choice needed, use cfg or runtime selection instead
```

**Rationale**: Cargo's feature resolver unifies features across the dependency graph. If package A enables `feature-x` and package B enables `feature-y` on the same dependency, both features will be active. Non-additive features break this model and cause confusing build failures.

**See also**: CG-BS-02 (optional dependencies), RFC 1956 (additive features)

---

## CG-BS-02: Use Optional Dependencies for Feature-Gated Code

**Strength**: SHOULD

**Summary**: Make dependencies optional and expose them through features when they're not always needed.

```toml
[dependencies]
# ✅ GOOD: Optional dependencies
serde = { version = "1.0", optional = true }
tokio = { version = "1.35", optional = true, features = ["full"] }

[features]
default = []
# Explicit features that enable optional deps
serialization = ["dep:serde"]
async-runtime = ["dep:tokio"]
```

```rust
// In code: conditional compilation based on features
#[cfg(feature = "serde")]
use serde::{Serialize, Deserialize};

#[cfg_attr(feature = "serde", derive(Serialize, Deserialize))]
pub struct Config {
    pub name: String,
    pub value: i32,
}

// ✅ GOOD: Feature-gated implementation
#[cfg(feature = "async-runtime")]
pub async fn fetch_data() -> Result<Data, Error> {
    tokio::time::sleep(Duration::from_secs(1)).await;
    Ok(Data::default())
}

// ❌ BAD: Always including expensive dependency
// [dependencies]
// tokio = "1.35"  # Always compiled even if never used
```

**Rationale**: Optional dependencies reduce compile time and binary size for users who don't need those features. They also allow libraries to support multiple ecosystems (e.g., sync and async) without forcing all users to pay the cost.

**See also**: CG-BS-03 (feature naming), Features documentation

---

## CG-BS-03: Name Features Clearly and Consistently

**Strength**: SHOULD

**Summary**: Use clear, descriptive feature names that follow community conventions.

```toml
[features]
# ✅ GOOD: Clear, descriptive names
default = ["std"]
std = []                          # Standard library support
serde = ["dep:serde"]             # Matches dependency name
tokio = ["dep:tokio"]             # Matches dependency name
full = ["std", "serde", "tokio"]  # "full" = all features

# ✅ GOOD: Domain-specific naming
tls = ["dep:rustls"]
compression = ["dep:flate2"]
websocket = ["dep:tungstenite"]

# ❌ BAD: Unclear abbreviations
ser = ["dep:serde"]               # What does "ser" mean?
tk = ["dep:tokio"]                # Unclear abbreviation

# ❌ BAD: Inconsistent with dependency
serialization = ["dep:serde"]     # Just call it "serde"

# ❌ BAD: Implementation details in name
use_rustls_not_openssl = ["dep:rustls"]  # Too verbose, implementation detail
```

**Rationale**: Feature names become part of your public API. Users specify them in their Cargo.toml and documentation. Clear naming reduces friction. Convention is to name features after the optional dependency they enable (e.g., `serde` feature enables `dep:serde`).

**See also**: CG-BS-02 (optional dependencies)

---

## CG-BS-04: Provide a Useful `default` Feature Set

**Strength**: SHOULD

**Summary**: The `default` feature should enable commonly-used functionality without unnecessary bloat.

```toml
# ✅ GOOD: Sensible defaults for most users
[features]
default = ["std"]    # Most users want std
std = []
alloc = []          # No-std with allocator

# ✅ GOOD: Default for a web framework
[features]
default = ["cookies", "sessions"]
cookies = []
sessions = ["dep:cookie"]
websockets = ["dep:tungstenite"]  # Not default - specialized use case

# ✅ GOOD: Default for a CLI tool
[features]
default = ["colors", "suggestions"]
colors = ["dep:colored"]
suggestions = ["dep:similar"]
advanced = ["colors", "suggestions", "history"]  # Power users opt-in

# ❌ BAD: Empty default when most need features
[features]
default = []  # Forces every user to manually enable std
std = []

# ❌ BAD: Too many features by default
[features]
default = ["std", "serde", "tokio", "tracing", "metrics", "profiling"]
# Forces all users to compile features they might not need
```

**Rationale**: The `default` feature is what users get if they don't specify `default-features = false`. It should represent the most common use case. Users needing minimal builds can opt-out with `default-features = false`.

**See also**: CG-BS-05 (no-std support)

---

## CG-BS-05: Support `no_std` with `std` Feature

**Strength**: CONSIDER

**Summary**: For libraries that can work without std, gate std-dependent code behind a `std` feature.

```toml
[features]
default = ["std"]
std = []
alloc = []  # For collections without full std
```

```rust
// ✅ GOOD: Conditional std usage
#![no_std]
#![cfg_attr(feature = "std", feature(std))]

#[cfg(feature = "std")]
extern crate std;

#[cfg(feature = "alloc")]
extern crate alloc;

// Use conditional imports
#[cfg(feature = "std")]
use std::vec::Vec;

#[cfg(all(feature = "alloc", not(feature = "std")))]
use alloc::vec::Vec;

// ✅ GOOD: Platform-agnostic error type
#[cfg(feature = "std")]
pub type Error = Box<dyn std::error::Error>;

#[cfg(not(feature = "std"))]
pub type Error = &'static str;

// ❌ BAD: Hard-coding std dependency
use std::collections::HashMap;  // Breaks no_std builds
```

**Rationale**: Embedded systems, WASM, and OS kernels often need `no_std`. Supporting it expands your library's applicability. The standard pattern is: `std` feature in default, users opt-out with `default-features = false`.

**See also**: The Embedded Rust Book, CG-BS-04 (default features)

---

## CG-BS-06: Optimize Development Profile for Fast Iteration

**Strength**: SHOULD

**Summary**: Customize the dev profile to balance compile speed with debuggability.

```toml
# ✅ GOOD: Fast iterative development
[profile.dev]
opt-level = 0              # No optimization - fastest compile
debug = "line-tables-only" # Minimal debug info for backtraces
strip = "none"             # Keep symbols
incremental = true         # Enable incremental compilation

# ✅ GOOD: Optimize dependencies but not workspace code
[profile.dev.package."*"]
opt-level = 2              # Optimize dependencies
debug = false              # Skip debug info for dependencies

# ✅ GOOD: Custom debugging profile when needed
[profile.debugging]
inherits = "dev"
debug = true               # Full debug info
opt-level = 0

# Usage: cargo build --profile debugging

# ❌ BAD: Over-optimizing dev profile
[profile.dev]
opt-level = 3              # Slow compiles during development!
lto = true                 # Very slow linking!
codegen-units = 1          # Serializes compilation!
```

**Rationale**: Development builds happen frequently. Minimize their cost by skipping optimization and minimizing debug info. Optimize dependencies (they change rarely) but not workspace code (it changes constantly). Create separate profiles for when you need full debugging or profiling.

**See also**: CG-BS-07 (release profile), Optimizing Build Performance

---

## CG-BS-07: Configure Release Profile for Production

**Strength**: SHOULD

**Summary**: Customize the release profile for optimal runtime performance and binary size.

```toml
# ✅ GOOD: Optimize for performance
[profile.release]
opt-level = 3              # Maximum optimization
lto = "thin"               # Link-time optimization (good balance)
codegen-units = 1          # Better optimization, slower compile
strip = "symbols"          # Remove debug symbols
panic = "abort"            # Smaller binary, no unwinding

# ✅ GOOD: Optimize for binary size
[profile.release-small]
inherits = "release"
opt-level = "z"            # Optimize for size
lto = true                 # Full LTO for size reduction
strip = "symbols"          # Remove symbols
panic = "abort"            # No unwind tables

# Usage: cargo build --profile release-small

# ✅ GOOD: Debug release builds (profiling)
[profile.release-debug]
inherits = "release"
debug = true               # Keep debug info for profiling
strip = "none"             # Keep symbols

# ❌ BAD: Disabling optimizations in release
[profile.release]
opt-level = 1              # Defeats purpose of release builds
lto = false                # Missing optimization opportunity
```

**Rationale**: Release builds are for production - optimize aggressively. Use `lto = "thin"` for good optimization with reasonable compile times. Create custom profiles for specific needs (size optimization, profiling). Never compromise production performance for faster release builds.

**See also**: CG-BS-06 (dev profile), Profile settings documentation

---

## CG-BS-08: Use Build Scripts Only When Necessary

**Strength**: SHOULD

**Summary**: Prefer alternatives to build scripts; use them only for legitimate build-time needs.

```rust
// build.rs

// ✅ GOOD: Legitimate build script uses
fn main() {
    // 1. Compiling C/C++ code
    cc::Build::new()
        .file("src/native.c")
        .compile("native");
    
    // 2. Generating code from specification
    protobuf_codegen::Codegen::new()
        .includes(&["protos"])
        .input("protos/api.proto")
        .cargo_out_dir("generated")
        .run()
        .expect("protobuf codegen failed");
    
    // 3. Detecting system libraries
    println!("cargo::rustc-link-lib=dylib=ssl");
    println!("cargo::rustc-link-search=native=/usr/local/lib");
    
    // 4. Setting cfg flags based on environment
    if cfg!(target_os = "windows") {
        println!("cargo::rustc-cfg=windows_specific");
    }
    
    // Tell Cargo when to re-run
    println!("cargo::rerun-if-changed=src/native.c");
    println!("cargo::rerun-if-changed=protos/api.proto");
}
```

```rust
// ❌ BAD: Unnecessary build script usage
fn main() {
    // Bad: Const generation (use const fn or declarative macros)
    let version = env!("CARGO_PKG_VERSION");
    std::fs::write("src/version.rs", 
        format!("const VERSION: &str = \"{}\";", version)).unwrap();
    // Just use: const VERSION: &str = env!("CARGO_PKG_VERSION");
    
    // Bad: Code generation that proc macros can do
    // Use derive macros instead
    
    // Bad: Downloading resources at build time
    // Breaks offline builds and caching
    
    // Bad: Running tests or lints
    // Use cargo test and cargo clippy
}
```

**Rationale**: Build scripts add complexity, slow incremental builds, and can break portability. Use them only for: compiling native code, platform detection, generating code from external specs, or linking system libraries. For other needs: proc macros (code generation), const fn (constant computation), or declarative macros (code templates).

**See also**: CG-BS-09 (build script best practices), Build Scripts documentation

---

## CG-BS-09: Follow Build Script Best Practices

**Strength**: MUST

**Summary**: Build scripts must be deterministic, fast, and correctly declare dependencies.

```rust
// build.rs

// ✅ GOOD: Proper build script structure
fn main() {
    // 1. Declare when to re-run
    println!("cargo::rerun-if-changed=build.rs");
    println!("cargo::rerun-if-changed=src/wrapper.h");
    println!("cargo::rerun-if-env-changed=CC");
    
    // 2. Use OUT_DIR for generated files
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());
    let generated = out_dir.join("generated.rs");
    
    // 3. Fast and deterministic operations
    generate_bindings(&generated);
    
    // 4. Explicit output declarations
    println!("cargo::rustc-link-lib=static=mylib");
}

// ❌ BAD: Common build script mistakes
fn main() {
    // Bad: No rerun-if directives (reruns every build!)
    // Missing: cargo::rerun-if-changed directives
    
    // Bad: Network access (breaks offline builds, caching)
    // let _ = reqwest::blocking::get("https://example.com/data");
    
    // Bad: Non-deterministic output
    // let timestamp = SystemTime::now();  // Changes every build!
    
    // Bad: Writing outside OUT_DIR
    // std::fs::write("src/generated.rs", content).unwrap();
    // Breaks when src/ is read-only or in CI
    
    // Bad: Slow operations without caching
    // Run expensive codegen every time even when inputs unchanged
    
    // Bad: Unwrap without context
    // env::var("SOME_VAR").unwrap();  // Unclear error!
    // Better: .expect("SOME_VAR must be set")
}
```

**Rationale**: Build scripts run during every build affecting compile time. Correct `rerun-if-*` directives enable caching. Deterministic output ensures reproducible builds. Using OUT_DIR prevents conflicts with source control and read-only filesystems.

**See also**: CG-BS-08 (when to use build scripts), Build Script Examples

---

## CG-BS-10: Link Native Libraries Correctly

**Strength**: MUST

**Summary**: Use proper `rustc-link-*` directives and the `links` key for native library integration.

```toml
# Cargo.toml
[package]
name = "mylib-sys"
links = "mylib"  # Ensures only one version links to native lib

[build-dependencies]
cc = "1.0"
pkg-config = "0.3"
```

```rust
// build.rs

// ✅ GOOD: Proper native library linking
fn main() {
    // Try pkg-config first (Linux/macOS)
    if let Ok(lib) = pkg_config::probe_library("mylib") {
        // pkg-config handled link directives
        return;
    }
    
    // Fall back to manual compilation
    cc::Build::new()
        .file("vendor/mylib.c")
        .compile("mylib");
    
    // Declare library linking
    println!("cargo::rustc-link-lib=static=mylib");
    
    // Declare search path if needed
    if let Ok(lib_dir) = env::var("MYLIB_DIR") {
        println!("cargo::rustc-link-search=native={}", lib_dir);
    }
    
    // Export metadata for dependent build scripts
    println!("cargo::include={}/include", env::var("OUT_DIR").unwrap());
    
    // Declare system dependencies
    println!("cargo::rustc-link-lib=dylib=ssl");
    println!("cargo::rustc-link-lib=dylib=crypto");
}

// ❌ BAD: Missing links key
// [package]
// name = "openssl-sys"
// # Missing: links = "openssl"
// # Multiple crates can link to openssl, causing symbol conflicts!

// ❌ BAD: Wrong link type
// println!("cargo::rustc-link-lib=mylib");  // Missing =static or =dylib
// println!("cargo::rustc-link-lib=static=ssl");  // ssl is dylib!
```

**Rationale**: The `links` key prevents multiple crates from linking incompatible versions of the same native library, avoiding symbol conflicts. Correct `rustc-link-lib` usage (static vs dylib) ensures proper linking. pkg-config integration provides portability across systems.

**See also**: CG-BS-11 (sys crate conventions), links key documentation

---

## CG-BS-11: Follow `-sys` Crate Conventions

**Strength**: SHOULD

**Summary**: Separate bindings to native code into `-sys` crates with raw, unsafe interfaces.

```
# Good structure:
mylib-sys/           # Unsafe, raw FFI bindings
├── Cargo.toml      # links = "mylib"
├── build.rs        # Compiles or finds native library
└── src/
    └── lib.rs      # pub use of extern "C" functions

mylib/              # Safe, idiomatic Rust wrapper
├── Cargo.toml
└── src/
    └── lib.rs
```

```toml
# mylib-sys/Cargo.toml
[package]
name = "mylib-sys"
version = "0.1.0"
links = "mylib"        # MUST have this

[build-dependencies]
cc = "1.0"
```

```rust
// mylib-sys/src/lib.rs
// ✅ GOOD: Raw, unsafe bindings
#![allow(non_camel_case_types)]

extern "C" {
    pub fn mylib_init() -> i32;
    pub fn mylib_process(data: *const u8, len: usize) -> i32;
    pub fn mylib_cleanup();
}
```

```toml
# mylib/Cargo.toml
[dependencies]
mylib-sys = "0.1"
```

```rust
// mylib/src/lib.rs
// ✅ GOOD: Safe, idiomatic wrapper
use mylib_sys as ffi;

pub struct MyLib {
    // ...
}

impl MyLib {
    pub fn new() -> Result<Self, Error> {
        unsafe {
            if ffi::mylib_init() != 0 {
                return Err(Error::InitFailed);
            }
        }
        Ok(MyLib { /* ... */ })
    }
    
    pub fn process(&self, data: &[u8]) -> Result<(), Error> {
        unsafe {
            if ffi::mylib_process(data.as_ptr(), data.len()) != 0 {
                return Err(Error::ProcessFailed);
            }
        }
        Ok(())
    }
}

impl Drop for MyLib {
    fn drop(&mut self) {
        unsafe { ffi::mylib_cleanup(); }
    }
}
```

**Rationale**: Separating `-sys` from wrapper crates follows Rust conventions and enables sharing FFI bindings across multiple safe wrapper libraries. The `-sys` crate provides 1:1 mapping to C API (unsafe), while wrapper crates provide safety, ergonomics, and Rust idioms.

**See also**: CG-BS-10 (linking native libraries), libz-sys example

---

## CG-BS-12: Use Incremental Compilation in Development

**Strength**: SHOULD

**Summary**: Enable incremental compilation to speed up iterative development builds.

```toml
# ✅ GOOD: Incremental compilation enabled (default for dev)
[profile.dev]
incremental = true    # Default, but being explicit

# ✅ GOOD: Disable for CI or release
[profile.release]
incremental = false   # Cleaner for distribution builds

# Environment variable override:
# CARGO_INCREMENTAL=1 cargo build  # Force enable
# CARGO_INCREMENTAL=0 cargo build  # Force disable
```

```bash
# ✅ GOOD: CI configuration
# .github/workflows/ci.yml
env:
  CARGO_INCREMENTAL: 0  # Disable for CI (no benefit, uses disk)
  CARGO_TERM_COLOR: always

# ❌ BAD: Leaving incremental on in CI
# Wastes time and disk space since CI starts fresh each run
```

**Rationale**: Incremental compilation reuses previous compilation artifacts, dramatically speeding up rebuilds during development. It's enabled by default for dev profile. Disable in CI (no reuse across runs) and release (cleaner builds). The cache in `target/` can grow large - clean periodically with `cargo clean` if needed.

**See also**: CG-BS-06 (dev profile optimization), Incremental Compilation documentation

---

## Best Practices Summary

### Quick Reference Table

| Pattern | Strength | Key Insight |
|---------|----------|-------------|
| Additive features | MUST | Features should only add, never subtract functionality |
| Optional dependencies | SHOULD | Reduce compile time for unused features |
| Clear feature names | SHOULD | Match dependency names, use community conventions |
| Useful defaults | SHOULD | Enable common features, users opt-out if needed |
| no_std support | CONSIDER | Gate std behind feature for embedded/WASM support |
| Dev profile optimization | SHOULD | Minimize compile time, optimize dependencies only |
| Release profile tuning | SHOULD | Maximize runtime performance with LTO and optimization |
| Build scripts sparingly | SHOULD | Use only for native code, codegen, or platform detection |
| Build script best practices | MUST | Deterministic, fast, correct rerun-if directives |
| Native library linking | MUST | Use links key and proper rustc-link-lib directives |
| -sys crate convention | SHOULD | Separate unsafe FFI from safe wrappers |
| Incremental compilation | SHOULD | Enable for dev, disable for CI and release |

---

## Related Guidelines

- **Basics**: See `01-cargo-basics.md` for package creation and dependencies
- **Publishing**: See `04-cargo-publishing.md` for feature compatibility and SemVer
- **Configuration**: See `05-cargo-configuration.md` for .cargo/config.toml profiles
- **Advanced**: See `06-cargo-advanced.md` for build performance optimization

---

## External References

- [Cargo Book - Features](https://doc.rust-lang.org/cargo/reference/features.html)
- [Cargo Book - Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html)
- [Cargo Book - Build Scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html)
- [RFC 1956 - Additive Features](https://github.com/rust-lang/rfcs/blob/master/text/1956-allocator.md)
- [The Embedded Rust Book - no_std](https://docs.rust-embedded.org/book/intro/no-std.html)
