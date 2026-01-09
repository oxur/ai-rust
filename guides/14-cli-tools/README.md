# CLI Tools Development Guide

> **Comprehensive guide for building command-line applications in Rust**
>
> Part of the Rust Guidelines series - Guide #14

## Overview

This guide covers everything you need to build professional, polished command-line tools in Rust. It emphasizes modern best practices, particularly the use of **clap's derive API** for argument parsing, and covers the full lifecycle from project setup through distribution.

## Quick Start

```rust
use clap::Parser;

/// Simple CLI application
#[derive(Parser)]
#[command(version, about, long_about = None)]
struct Cli {
    /// Name of the person to greet
    #[arg(short, long)]
    name: String,
    
    /// Number of times to greet
    #[arg(short, long, default_value_t = 1)]
    count: u8,
}

fn main() {
    let cli = Cli::parse();
    
    for _ in 0..cli.count {
        println!("Hello {}!", cli.name);
    }
}
```

```bash
$ cargo add clap --features derive
$ cargo run -- --name Alice --count 3
Hello Alice!
Hello Alice!
Hello Alice!
```

## Guide Structure

This guide is organized into 9 focused sections covering 52 patterns (CLI-01 through CLI-52):

### [01. Project Setup](01-project-setup.md) (CLI-01 to CLI-04)

Essential patterns for structuring Rust CLI applications.

- **CLI-01**: Binary Crate Structure for CLI Tools [MUST]
- **CLI-02**: Cargo.toml Dependencies and Features [MUST]
- **CLI-03**: Separation of Library and Binary [SHOULD]
- **CLI-04**: Entry Point and Main Function Structure [MUST]

**Key takeaway**: Keep `main.rs` thin, put logic in library modules, expose a testable API.

### [02. Argument Parsing](02-argument-parsing.md) (CLI-05 to CLI-15)

Core patterns for parsing command-line arguments using clap's derive API.

- **CLI-05**: Use Clap Derive for All Argument Parsing [MUST]
- **CLI-06**: Positional Arguments [MUST]
- **CLI-07**: Optional Arguments with Defaults [SHOULD]
- **CLI-08**: Boolean Flags [MUST]
- **CLI-09**: Multiple Values and Collections [SHOULD]
- **CLI-10**: Subcommands with Enums [SHOULD]
- **CLI-11**: Value Validation and Custom Parsers [SHOULD]
- **CLI-12**: Environment Variable Integration [CONSIDER]
- **CLI-13**: Argument Groups and Relationships [CONSIDER]
- **CLI-14**: Help Text and Documentation [MUST]
- **CLI-15**: Version Information [MUST]

**Key takeaway**: Never use `std::env::args()`. Always use clap derive for type-safe, validated argument parsing.

### [03. Error Handling](03-error-handling.md) (CLI-16 to CLI-20)

Patterns for handling and reporting errors in command-line applications.

- **CLI-16**: Exit Codes (0 for Success, Non-Zero for Errors) [MUST]
- **CLI-17**: User-Friendly Error Messages [MUST]
- **CLI-18**: Error Context for CLI Operations [SHOULD]
- **CLI-19**: Distinguish User Errors from System Errors [SHOULD]
- **CLI-20**: Error Output to Stderr [MUST]

**Key takeaway**: Exit 0 for success, non-zero for errors. Write errors to stderr with clear, actionable messages.

### [04. Output and User Experience](04-output-and-ux.md) (CLI-21 to CLI-28)

Patterns for creating polished, user-friendly command-line interfaces.

- **CLI-21**: Human vs Machine-Readable Output [SHOULD]
- **CLI-22**: Progress Indicators for Long Operations [CONSIDER]
- **CLI-23**: Colored Output with Terminal Detection [CONSIDER]
- **CLI-24**: Verbosity Levels and Logging [SHOULD]
- **CLI-25**: Confirmation Prompts [CONSIDER]
- **CLI-26**: Structured Output (JSON, CSV) [CONSIDER]
- **CLI-27**: Stdout vs Stderr Usage [MUST]
- **CLI-28**: Buffered vs Unbuffered Output [CONSIDER]

**Key takeaway**: Data to stdout, everything else to stderr. Support both human and machine formats.

### [05. Configuration](05-configuration.md) (CLI-29 to CLI-33)

Patterns for managing configuration in command-line applications.

- **CLI-29**: Configuration File Formats [CONSIDER]
- **CLI-30**: Configuration File Locations [SHOULD]
- **CLI-31**: Configuration Precedence (Defaults → File → Env → Args) [MUST]
- **CLI-32**: Config Validation and Defaults [SHOULD]
- **CLI-33**: XDG Base Directory Specification [CONSIDER]

**Key takeaway**: Precedence is defaults → file → env vars → CLI args. Use platform-specific directories.

### [06. Testing](06-testing.md) (CLI-34 to CLI-38)

Patterns for testing command-line applications.

- **CLI-34**: Integration Testing with assert_cmd [SHOULD]
- **CLI-35**: Testing Argument Parsing [MUST]
- **CLI-36**: Snapshot Testing for Output [CONSIDER]
- **CLI-37**: Testing Error Conditions [SHOULD]
- **CLI-38**: Testing with Temporary Files [CONSIDER]

**Key takeaway**: Test your CLI end-to-end with `assert_cmd`. Test argument parsing explicitly.

### [07. Distribution and Packaging](07-distribution.md) (CLI-39 to CLI-42)

Patterns for distributing and packaging command-line tools.

- **CLI-39**: Binary Size Optimization [CONSIDER]
- **CLI-40**: Release Profile Configuration [SHOULD]
- **CLI-41**: Cross-Compilation Considerations [CONSIDER]
- **CLI-42**: cargo-binstall Compatibility [CONSIDER]

**Key takeaway**: Optimize release builds for size. Support cross-compilation and pre-built binaries.

### [08. Advanced Topics](08-advanced-topics.md) (CLI-43 to CLI-48)

Advanced patterns for sophisticated command-line applications.

- **CLI-43**: Signal Handling (Ctrl-C, SIGTERM) [SHOULD]
- **CLI-44**: Shell Completion Generation [CONSIDER]
- **CLI-45**: Plugin Architectures [CONSIDER]
- **CLI-46**: Async CLI Applications [CONSIDER]
- **CLI-47**: Cross-Platform Considerations [SHOULD]
- **CLI-48**: Piping and Stdin/Stdout Integration [SHOULD]

**Key takeaway**: Handle signals gracefully. Support stdin/stdout for Unix-style composition.

### [09. Common Pitfalls](09-common-pitfalls.md) (CLI-49 to CLI-52)

Anti-patterns and pitfalls to avoid when building command-line tools.

- **CLI-49**: Never Use std::env::args Directly [AVOID]
- **CLI-50**: Don't Ignore Broken Pipe Errors [AVOID]
- **CLI-51**: Avoid Blocking Operations Without Feedback [AVOID]
- **CLI-52**: Don't Mix Output Types on Same Stream [AVOID]

**Key takeaway**: Avoid std::env::args, handle broken pipes, show progress, separate data from messages.

## Complete Pattern Index

### MUST Patterns (Required for Correctness)

| Pattern | Summary | Location |
|---------|---------|----------|
| CLI-01 | Binary crate structure | [01-project-setup.md](01-project-setup.md#cli-01) |
| CLI-02 | Cargo.toml configuration | [01-project-setup.md](01-project-setup.md#cli-02) |
| CLI-04 | Main function structure | [01-project-setup.md](01-project-setup.md#cli-04) |
| CLI-05 | Use clap derive | [02-argument-parsing.md](02-argument-parsing.md#cli-05) |
| CLI-06 | Positional arguments | [02-argument-parsing.md](02-argument-parsing.md#cli-06) |
| CLI-08 | Boolean flags | [02-argument-parsing.md](02-argument-parsing.md#cli-08) |
| CLI-14 | Help text | [02-argument-parsing.md](02-argument-parsing.md#cli-14) |
| CLI-15 | Version information | [02-argument-parsing.md](02-argument-parsing.md#cli-15) |
| CLI-16 | Exit codes | [03-error-handling.md](03-error-handling.md#cli-16) |
| CLI-17 | User-friendly errors | [03-error-handling.md](03-error-handling.md#cli-17) |
| CLI-20 | Errors to stderr | [03-error-handling.md](03-error-handling.md#cli-20) |
| CLI-27 | Stdout vs stderr | [04-output-and-ux.md](04-output-and-ux.md#cli-27) |
| CLI-31 | Config precedence | [05-configuration.md](05-configuration.md#cli-31) |
| CLI-35 | Test argument parsing | [06-testing.md](06-testing.md#cli-35) |

### SHOULD Patterns (Strong Recommendations)

| Pattern | Summary | Location |
|---------|---------|----------|
| CLI-03 | Library/binary separation | [01-project-setup.md](01-project-setup.md#cli-03) |
| CLI-07 | Defaults for arguments | [02-argument-parsing.md](02-argument-parsing.md#cli-07) |
| CLI-09 | Multiple values | [02-argument-parsing.md](02-argument-parsing.md#cli-09) |
| CLI-10 | Subcommands | [02-argument-parsing.md](02-argument-parsing.md#cli-10) |
| CLI-11 | Value validation | [02-argument-parsing.md](02-argument-parsing.md#cli-11) |
| CLI-18 | Error context | [03-error-handling.md](03-error-handling.md#cli-18) |
| CLI-19 | Distinguish error types | [03-error-handling.md](03-error-handling.md#cli-19) |
| CLI-21 | Human/machine output | [04-output-and-ux.md](04-output-and-ux.md#cli-21) |
| CLI-24 | Verbosity levels | [04-output-and-ux.md](04-output-and-ux.md#cli-24) |
| CLI-30 | Config locations | [05-configuration.md](05-configuration.md#cli-30) |
| CLI-32 | Config validation | [05-configuration.md](05-configuration.md#cli-32) |
| CLI-34 | Integration testing | [06-testing.md](06-testing.md#cli-34) |
| CLI-37 | Test errors | [06-testing.md](06-testing.md#cli-37) |
| CLI-40 | Release profiles | [07-distribution.md](07-distribution.md#cli-40) |
| CLI-43 | Signal handling | [08-advanced-topics.md](08-advanced-topics.md#cli-43) |
| CLI-47 | Cross-platform | [08-advanced-topics.md](08-advanced-topics.md#cli-47) |
| CLI-48 | Stdin/stdout support | [08-advanced-topics.md](08-advanced-topics.md#cli-48) |

### AVOID Patterns (Anti-patterns)

| Pattern | Summary | Location |
|---------|---------|----------|
| CLI-49 | Never use std::env::args | [09-common-pitfalls.md](09-common-pitfalls.md#cli-49) |
| CLI-50 | Handle broken pipes | [09-common-pitfalls.md](09-common-pitfalls.md#cli-50) |
| CLI-51 | Show progress | [09-common-pitfalls.md](09-common-pitfalls.md#cli-51) |
| CLI-52 | Don't mix output types | [09-common-pitfalls.md](09-common-pitfalls.md#cli-52) |

## Essential Dependencies

```toml
[dependencies]
# Argument parsing (MUST)
clap = { version = "4.5", features = ["derive", "cargo", "wrap_help"] }

# Error handling (SHOULD)
anyhow = "1.0"          # Ergonomic error handling
thiserror = "1.0"       # Custom error types

# Logging (SHOULD)
log = "0.4"             # Logging facade
env_logger = "0.11"     # Simple logger

# Optional but useful
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"      # JSON support
colored = "2.0"         # Terminal colors
indicatif = "0.17"      # Progress bars
directories = "5.0"     # Platform-specific paths

[dev-dependencies]
# Testing (SHOULD)
assert_cmd = "2.0"      # CLI testing
assert_fs = "1.1"       # Filesystem fixtures
predicates = "3.1"      # Assertions
insta = "1.34"          # Snapshot testing
```

## Typical CLI Project Structure

```
my-tool/
├── Cargo.toml
├── README.md
├── LICENSE
├── src/
│   ├── main.rs              # Thin entry point
│   ├── lib.rs               # Main library
│   ├── cli.rs               # CLI argument definitions
│   ├── commands/            # Command implementations
│   │   ├── mod.rs
│   │   ├── process.rs
│   │   └── validate.rs
│   ├── config.rs            # Configuration handling
│   └── error.rs             # Error types
├── tests/
│   ├── cli_tests.rs         # Integration tests
│   └── fixtures/            # Test data
└── examples/
    └── basic_usage.rs       # Usage examples
```

## Example: Complete CLI Application

```rust
// src/cli.rs
use clap::{Parser, Subcommand};
use std::path::PathBuf;

/// A tool for processing data files
#[derive(Parser)]
#[command(version, about, long_about = None)]
pub struct Cli {
    /// Config file location
    #[arg(short, long)]
    pub config: Option<PathBuf>,
    
    /// Verbose output
    #[arg(short, long, action = clap::ArgAction::Count)]
    pub verbose: u8,
    
    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Subcommand)]
pub enum Commands {
    /// Process a file
    Process {
        /// Input file
        input: PathBuf,
        
        /// Output file
        #[arg(short, long)]
        output: Option<PathBuf>,
    },
    
    /// Validate configuration
    Validate {
        /// Config file to validate
        config: PathBuf,
    },
}

// src/main.rs
use anyhow::Result;
use clap::Parser;

mod cli;
use cli::{Cli, Commands};

fn main() {
    // Setup logging
    env_logger::init();
    
    // Parse CLI
    let cli = Cli::parse();
    
    // Execute
    if let Err(e) = run(cli) {
        eprintln!("Error: {}", e);
        std::process::exit(1);
    }
}

fn run(cli: Cli) -> Result<()> {
    match cli.command {
        Commands::Process { input, output } => {
            my_tool::commands::process(&input, output.as_ref())?;
        }
        Commands::Validate { config } => {
            my_tool::commands::validate(&config)?;
        }
    }
    Ok(())
}
```

## Related Guidelines

This CLI guide builds upon and references these other guidelines:

- **[01-core-idioms.md](../01-core-idioms.md)** - Rust fundamentals
- **[02-api-design.md](../02-api-design.md)** - Struct design for CLI arguments
- **[03-error-handling.md](../03-error-handling.md)** - Error types and propagation
- **[07-concurrency-async.md](../07-concurrency-async.md)** - Async patterns
- **[12-project-structure.md](../12-project-structure.md)** - Project organization
- **[13-documentation.md](../13-documentation.md)** - Documentation patterns

## External Resources

### Official Documentation
- [Clap Documentation](https://docs.rs/clap) - Argument parsing
- [Clap Derive Tutorial](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html)
- [Command Line Applications in Rust Book](https://rust-cli.github.io/book/)

### Essential Crates
- [clap](https://docs.rs/clap) - Command-line argument parser
- [clap_complete](https://docs.rs/clap_complete) - Shell completions
- [anyhow](https://docs.rs/anyhow) - Ergonomic error handling
- [thiserror](https://docs.rs/thiserror) - Custom error types
- [assert_cmd](https://docs.rs/assert_cmd) - CLI testing
- [indicatif](https://docs.rs/indicatif) - Progress bars
- [colored](https://docs.rs/colored) - Terminal colors
- [directories](https://docs.rs/directories) - Platform directories

### Best Practices
- [Command Line Interface Guidelines](https://clig.dev/)
- [12 Factor CLI Apps](https://medium.com/@jdxcode/12-factor-cli-apps-dd3c227a0e46)
- [POSIX Utility Conventions](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap12.html)

## Quick Reference

### Common Patterns Cheatsheet

```rust
// Basic CLI
#[derive(Parser)]
struct Cli {
    input: PathBuf,                    // Required positional
    #[arg(short, long)]
    output: Option<PathBuf>,           // Optional flag
    #[arg(short, long)]
    verbose: bool,                     // Boolean flag
}

// With defaults
#[derive(Parser)]
struct Cli {
    #[arg(short, long, default_value = "8080")]
    port: u16,
}

// Multiple values
#[derive(Parser)]
struct Cli {
    files: Vec<PathBuf>,               // Multiple positional
    #[arg(short, long)]
    tags: Vec<String>,                 // Multiple flag values
}

// Subcommands
#[derive(Parser)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    Process { file: PathBuf },
    Validate { config: PathBuf },
}

// Value validation
#[derive(Parser)]
struct Cli {
    #[arg(value_parser = clap::value_parser!(u16).range(1..))]
    port: u16,
}

// Environment variables
#[derive(Parser)]
struct Cli {
    #[arg(long, env = "MY_TOOL_TOKEN")]
    token: String,
}
```

## Contributing

This guide is part of the Rust Guidelines series. For updates or corrections, please see the main guidelines repository.

---

**Last Updated**: 2026-01-09  
**Guide Version**: 1.0  
**Compatible with**: Rust 1.74+, Clap 4.5+
