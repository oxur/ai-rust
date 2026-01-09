# CLI Project Setup

Essential patterns for structuring Rust CLI applications.

---

## CLI-01: Binary Crate Structure for CLI Tools

**Strength**: MUST

**Summary**: CLI applications should be binary crates with a focused `main.rs` and logic in library modules.

```rust
// ❌ BAD: Everything in main.rs
// src/main.rs (500+ lines)
use clap::Parser;
use std::fs;

#[derive(Parser)]
struct Cli {
    file: PathBuf,
}

fn main() {
    let cli = Cli::parse();
    // 400 lines of business logic here...
    // Hard to test, hard to maintain
}

// ✅ GOOD: Focused main.rs, logic in library
// src/main.rs
use clap::Parser;
use my_cli::{Cli, run};

fn main() {
    let cli = Cli::parse();
    if let Err(e) = run(cli) {
        eprintln!("Error: {}", e);
        std::process::exit(1);
    }
}

// src/lib.rs
use clap::Parser;
use anyhow::Result;
use std::path::PathBuf;

#[derive(Parser)]
pub struct Cli {
    /// Input file to process
    pub file: PathBuf,
}

pub fn run(cli: Cli) -> Result<()> {
    // Business logic here, fully testable
    process_file(&cli.file)?;
    Ok(())
}

fn process_file(path: &Path) -> Result<()> {
    // Implementation
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_process_file() {
        // Easy to test!
    }
}
```

**Rationale**: Separating CLI parsing from business logic enables unit testing, code reuse, and better organization. Your `main.rs` should be thin—just parse arguments and call library functions.

**See also**: 
- [12-project-structure.md](../12-project-structure.md) - General project organization
- CLI-03: Separation of library and binary

---

## CLI-02: Cargo.toml Dependencies and Features

**Strength**: MUST

**Summary**: Use appropriate clap features and common CLI dependencies in `Cargo.toml`.

```toml
# ❌ BAD: Missing essential features or unnecessary dependencies
[dependencies]
clap = "4.5"  # Missing derive feature!

# ✅ GOOD: Essential clap configuration
[dependencies]
clap = { version = "4.5", features = ["derive", "cargo", "wrap_help"] }
anyhow = "1.0"  # Error handling
env_logger = "0.11"  # Logging
log = "0.4"  # Logging facade

# ✅ GOOD: Additional useful crates
[dependencies]
clap = { version = "4.5", features = ["derive", "cargo"] }
anyhow = "1.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"  # For JSON output
indicatif = "0.17"  # Progress bars
colored = "2.0"  # Terminal colors

[dev-dependencies]
assert_cmd = "2.0"  # Integration testing
assert_fs = "1.1"  # Filesystem fixtures
predicates = "3.1"  # Assertions

# ✅ GOOD: Optimized release profile
[profile.release]
lto = true
codegen-units = 1
strip = true  # Remove debug symbols
opt-level = "z"  # Optimize for size
```

**Clap features explained**:
- `derive` - Enable `#[derive(Parser)]` (essential)
- `cargo` - `cargo!()` macro for metadata from Cargo.toml
- `wrap_help` - Wrap long help text at terminal width
- `color` - Colored help output (default, but can disable)
- `suggestions` - Suggest corrections for typos (enabled by default)

**Rationale**: The `derive` feature is essential for modern clap usage. The `cargo` feature lets you pull version and description from Cargo.toml automatically. Release profile optimization is critical for CLI tool distribution.

**See also**:
- CLI-39: Binary size optimization
- [External: clap feature flags](https://docs.rs/clap/latest/clap/_features/index.html)

---

## CLI-03: Separation of Library and Binary

**Strength**: SHOULD

**Summary**: Expose CLI functionality as a library to enable testing, reuse, and alternative interfaces.

```rust
// ❌ BAD: No library interface
// src/main.rs only, no src/lib.rs
fn main() {
    // All logic here, cannot reuse or test easily
}

// ✅ GOOD: Clear library interface
// src/lib.rs
pub mod cli;
pub mod commands;
pub mod error;

pub use cli::Cli;
pub use error::Error;

/// Main entry point for the library
pub fn run(cli: Cli) -> Result<(), Error> {
    match cli.command {
        Commands::Process { file } => commands::process(file),
        Commands::Validate { file } => commands::validate(file),
    }
}

// src/main.rs
use clap::Parser;
use my_tool::{Cli, run};

fn main() {
    let cli = Cli::parse();
    
    if let Err(e) = run(cli) {
        eprintln!("Error: {}", e);
        std::process::exit(1);
    }
}

// Now you can test from tests/:
// tests/integration.rs
use my_tool::{Cli, run};

#[test]
fn test_process_command() {
    let cli = Cli {
        command: Commands::Process {
            file: "test.txt".into()
        }
    };
    
    assert!(run(cli).is_ok());
}
```

**Rationale**: Exposing a library interface enables comprehensive testing without spawning processes, allows other Rust code to embed your tool's functionality, and lets you build alternative interfaces (GUI, TUI, API server) on the same core logic.

**See also**:
- CLI-01: Binary crate structure
- CLI-34: Integration testing with assert_cmd
- [12-project-structure.md](../12-project-structure.md#ps-03)

---

## CLI-04: Entry Point and Main Function Structure

**Strength**: MUST

**Summary**: Structure `main()` to parse arguments, set up logging, execute commands, and handle errors properly.

```rust
// ❌ BAD: Poor error handling, no setup
use clap::Parser;

fn main() {
    let cli = Cli::parse();
    do_work(cli).unwrap();  // Panics on error!
}

// ✅ GOOD: Proper structure with error handling
use clap::Parser;
use anyhow::Result;

fn main() {
    // Set up logging/tracing
    env_logger::init();
    
    // Parse CLI arguments
    let cli = Cli::parse();
    
    // Execute with proper error handling
    if let Err(err) = run(cli) {
        // Print error to stderr
        eprintln!("Error: {}", err);
        
        // Print cause chain if available
        let mut source = err.source();
        while let Some(err) = source {
            eprintln!("  Caused by: {}", err);
            source = err.source();
        }
        
        // Exit with error code
        std::process::exit(1);
    }
}

fn run(cli: Cli) -> Result<()> {
    // Business logic here
    Ok(())
}

// ✅ BETTER: Use anyhow's built-in error reporting
use clap::Parser;

fn main() -> anyhow::Result<()> {
    env_logger::init();
    let cli = Cli::parse();
    run(cli)
    // anyhow automatically provides nice error formatting
}

// ✅ BEST: Control exit code explicitly
use clap::Parser;

fn main() {
    env_logger::init();
    let cli = Cli::parse();
    
    match run(cli) {
        Ok(()) => std::process::exit(0),
        Err(err) => {
            // Custom error formatting
            eprintln!("Error: {:?}", err);
            std::process::exit(1);
        }
    }
}
```

**Error handling approaches**:

1. **Return `Result` from main** (Rust 1.26+):
   ```rust
   fn main() -> Result<(), Box<dyn Error>> {
       // Rust will print the error and exit(1)
   }
   ```

2. **Use anyhow for ergonomic errors**:
   ```rust
   fn main() -> anyhow::Result<()> {
       // Better error messages with context
   }
   ```

3. **Explicit exit codes** (most control):
   ```rust
   fn main() {
       std::process::exit(match run() {
           Ok(()) => 0,
           Err(e) => {
               eprintln!("{}", e);
               1
           }
       });
   }
   ```

**Rationale**: A well-structured `main()` ensures proper initialization (logging), clear error reporting (to stderr), and correct exit codes. Users and scripts rely on exit codes to determine success or failure.

**See also**:
- CLI-16: Exit codes (0 for success, non-zero for errors)
- CLI-20: Error output to stderr
- [03-error-handling.md](../03-error-handling.md) - Error handling patterns

---

## Best Practices Summary

| Pattern | Strength | Key Takeaway |
|---------|----------|--------------|
| CLI-01 | MUST | Keep main.rs thin, put logic in library modules |
| CLI-02 | MUST | Use clap with derive and cargo features |
| CLI-03 | SHOULD | Expose library interface for testing and reuse |
| CLI-04 | MUST | Structure main() properly: setup, parse, execute, error handling |

## Related Guidelines

- [12-project-structure.md](../12-project-structure.md) - Project organization
- [03-error-handling.md](../03-error-handling.md) - Error handling
- CLI-05 through CLI-15: Argument parsing patterns (next file)

## External References

- [Clap Derive Tutorial](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html)
- [Cargo.toml Manifest Format](https://doc.rust-lang.org/cargo/reference/manifest.html)
- [Command Line Applications in Rust Book](https://rust-cli.github.io/book/)
