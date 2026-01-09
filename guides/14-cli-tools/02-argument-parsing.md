# Argument Parsing with Clap Derive

Core patterns for parsing command-line arguments using clap's derive API.

---

## CLI-05: Use Clap Derive for All Argument Parsing

**Strength**: MUST

**Summary**: Always use clap's derive API (`#[derive(Parser)]`) for argument parsing. Never use `std::env::args()` directly.

```rust
// ❌ BAD: Manual argument parsing with std::env::args
fn main() {
    let args: Vec<String> = std::env::args().collect();
    
    if args.len() < 2 {
        eprintln!("Usage: {} <file>", args[0]);
        std::process::exit(1);
    }
    
    let filename = &args[1];
    // No type safety, no validation, poor error messages
}

// ❌ BAD: Trying to parse flags manually
fn main() {
    let args: Vec<String> = std::env::args().collect();
    let mut verbose = false;
    let mut filename = None;
    
    for arg in &args[1..] {
        if arg == "--verbose" || arg == "-v" {
            verbose = true;
        } else {
            filename = Some(arg.clone());
        }
    }
    // Fragile, doesn't handle --verbose=true, -vvv, etc.
}

// ✅ GOOD: Use clap derive
use clap::Parser;
use std::path::PathBuf;

#[derive(Parser)]
#[command(name = "my-tool")]
#[command(version, about, long_about = None)]
struct Cli {
    /// Input file to process
    file: PathBuf,
    
    /// Verbose output
    #[arg(short, long)]
    verbose: bool,
}

fn main() {
    let cli = Cli::parse();
    
    if cli.verbose {
        println!("Processing: {}", cli.file.display());
    }
    // Type-safe, validated, automatic help, great error messages!
}
```

**Rationale**: Clap provides type safety, validation, automatic help generation, shell completions, and excellent error messages. Manual argument parsing is error-prone, lacks these features, and doesn't scale. Using `std::env::args()` directly is an anti-pattern in modern Rust CLI development.

**See also**:
- CLI-49: Never use std::env::args directly [AVOID]
- [External: Clap Derive Tutorial](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html)

---

## CLI-06: Positional Arguments

**Strength**: MUST

**Summary**: Use plain struct fields for required positional arguments, `Option<T>` for optional ones.

```rust
use clap::Parser;
use std::path::PathBuf;

// ✅ GOOD: Required positional argument
#[derive(Parser)]
struct Cli {
    /// Input file to process
    input: PathBuf,
}

// ✅ GOOD: Optional positional argument
#[derive(Parser)]
struct Cli {
    /// Input file (defaults to stdin if not provided)
    input: Option<PathBuf>,
}

// ✅ GOOD: Multiple positional arguments
#[derive(Parser)]
struct Cli {
    /// Source file
    source: PathBuf,
    
    /// Destination file
    destination: PathBuf,
}

// ✅ GOOD: Multiple values with Vec
#[derive(Parser)]
struct Cli {
    /// Files to process
    files: Vec<PathBuf>,
}

// ❌ BAD: Optional positional with default_value_t is confusing
#[derive(Parser)]
struct Cli {
    #[arg(default_value_t = String::from("input.txt"))]
    input: String,  // User doesn't know if they need to provide it
}

// ✅ GOOD: Use Option for truly optional, or make it required
#[derive(Parser)]
struct Cli {
    /// Input file (optional, defaults to stdin)
    input: Option<PathBuf>,
}
```

**Usage examples**:
```bash
# Required positional
$ my-tool input.txt

# Optional positional  
$ my-tool              # Works without argument
$ my-tool input.txt    # Works with argument

# Multiple positional
$ my-tool src.txt dst.txt

# Vec of values
$ my-tool file1.txt file2.txt file3.txt
```

**Rationale**: Positional arguments are intuitive for file-based operations. Required arguments should use plain types, optional arguments should use `Option<T>`. This makes the API self-documenting.

**See also**:
- CLI-07: Optional arguments with defaults
- CLI-09: Multiple values and collections

---

## CLI-07: Optional Arguments with Defaults

**Strength**: SHOULD

**Summary**: Use `#[arg(default_value = "...")]` for optional arguments with sensible defaults.

```rust
use clap::Parser;

// ✅ GOOD: Optional flag with default
#[derive(Parser)]
struct Cli {
    /// Port to bind to
    #[arg(short, long, default_value = "8080")]
    port: u16,
    
    /// Output format
    #[arg(short, long, default_value = "json")]
    format: String,
}

// ✅ GOOD: Default with type inference
#[derive(Parser)]
struct Cli {
    /// Number of retries
    #[arg(short, long, default_value_t = 3)]
    retries: u32,
}

// ✅ GOOD: Default from environment
#[derive(Parser)]
struct Cli {
    /// Config file path
    #[arg(short, long, default_value = "config.toml")]
    #[arg(env = "MY_TOOL_CONFIG")]
    config: PathBuf,
}

// ✅ GOOD: No default for sensitive defaults
#[derive(Parser)]
struct Cli {
    /// API endpoint
    #[arg(short, long)]
    #[arg(env = "API_ENDPOINT")]
    endpoint: Option<String>,  // Force user to set it
}

// ❌ BAD: Default that doesn't make sense
#[derive(Parser)]
struct Cli {
    #[arg(long, default_value = "unknown")]
    username: String,  // This will cause confusion
}
```

**Rationale**: Defaults make tools easier to use for common cases while allowing customization. Document defaults in help text and choose sensible values that work for most users.

**See also**:
- CLI-12: Environment variable integration
- CLI-31: Configuration precedence

---

## CLI-08: Boolean Flags

**Strength**: MUST

**Summary**: Use `bool` type for flags, prefer `#[arg(short, long)]` for discoverability.

```rust
use clap::Parser;

// ✅ GOOD: Simple boolean flag
#[derive(Parser)]
struct Cli {
    /// Enable verbose output
    #[arg(short, long)]
    verbose: bool,
}

// ✅ GOOD: Multiple flags
#[derive(Parser)]
struct Cli {
    /// Verbose output (-v)
    #[arg(short, long)]
    verbose: bool,
    
    /// Force operation without confirmation
    #[arg(short, long)]
    force: bool,
    
    /// Dry run - don't make changes
    #[arg(short, long)]
    dry_run: bool,
}

// ✅ GOOD: Counting flag (e.g., -vvv for more verbosity)
use clap::Parser;

#[derive(Parser)]
struct Cli {
    /// Increase verbosity (-v, -vv, -vvv)
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,
}

fn main() {
    let cli = Cli::parse();
    
    let log_level = match cli.verbose {
        0 => "error",
        1 => "warn",
        2 => "info",
        _ => "debug",
    };
}

// ✅ GOOD: Negation flags
#[derive(Parser)]
struct Cli {
    /// Enable color output
    #[arg(long, default_value_t = true)]
    #[arg(action = clap::ArgAction::SetTrue)]
    color: bool,
    
    /// Disable color output
    #[arg(long)]
    #[arg(action = clap::ArgAction::SetFalse)]
    no_color: bool,
}

// ❌ BAD: Using Option<bool> for flags
#[derive(Parser)]
struct Cli {
    #[arg(short, long)]
    verbose: Option<bool>,  // Confusing! Just use bool
}
```

**Usage examples**:
```bash
# Simple flag
$ my-tool --verbose
$ my-tool -v

# Multiple flags
$ my-tool -vf  # verbose and force
$ my-tool --verbose --force

# Counting flags
$ my-tool -vvv  # Very verbose

# Negation
$ my-tool --no-color
```

**Rationale**: Boolean flags are the standard way to enable/disable features. Both short (`-v`) and long (`--verbose`) forms improve usability. Count actions support common patterns like increasing verbosity.

**See also**:
- CLI-24: Verbosity levels and logging
- CLI-14: Help text and documentation

---

## CLI-09: Multiple Values and Collections

**Strength**: SHOULD

**Summary**: Use `Vec<T>` for collecting multiple values, configure delimiter and occurrence behavior.

```rust
use clap::Parser;
use std::path::PathBuf;

// ✅ GOOD: Vec for multiple files
#[derive(Parser)]
struct Cli {
    /// Files to process
    files: Vec<PathBuf>,
}
// Usage: my-tool file1.txt file2.txt file3.txt

// ✅ GOOD: Named option with multiple values
#[derive(Parser)]
struct Cli {
    /// Tags to filter by
    #[arg(short, long)]
    tags: Vec<String>,
}
// Usage: my-tool --tags foo --tags bar --tags baz

// ✅ GOOD: Value delimiter for comma-separated values
#[derive(Parser)]
struct Cli {
    /// Tags (comma-separated)
    #[arg(short, long, value_delimiter = ',')]
    tags: Vec<String>,
}
// Usage: my-tool --tags foo,bar,baz

// ✅ GOOD: Require at least one value
#[derive(Parser)]
struct Cli {
    /// At least one file required
    #[arg(required = true)]
    files: Vec<PathBuf>,
}

// ✅ GOOD: Exact number of values
#[derive(Parser)]
struct Cli {
    /// RGB color values (exactly 3 numbers)
    #[arg(short, long, num_args = 3)]
    color: Vec<u8>,
}
// Usage: my-tool --color 255 128 0

// ❌ BAD: Vec without clear usage pattern
#[derive(Parser)]
struct Cli {
    values: Vec<String>,  // How many? How to specify?
}
```

**Rationale**: Collections enable processing multiple inputs elegantly. Use positional `Vec<T>` for multiple files (like `grep pattern file1 file2`), and named options with `Vec<T>` for repeated flags (like `--tag foo --tag bar`).

**See also**:
- CLI-06: Positional arguments
- CLI-48: Piping and stdin/stdout integration

---

## CLI-10: Subcommands with Enums

**Strength**: SHOULD

**Summary**: Use `#[derive(Subcommand)]` on enums to implement Git-style subcommands.

```rust
use clap::{Parser, Subcommand};
use std::path::PathBuf;

// ✅ GOOD: Subcommands with enum
#[derive(Parser)]
#[command(name = "my-tool")]
#[command(about = "A tool with multiple commands")]
struct Cli {
    /// Global verbose flag
    #[arg(short, long, global = true)]
    verbose: bool,
    
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Process a file
    Process {
        /// Input file
        file: PathBuf,
        
        /// Output file
        #[arg(short, long)]
        output: Option<PathBuf>,
    },
    
    /// Validate configuration
    Validate {
        /// Config file to validate
        #[arg(default_value = "config.toml")]
        config: PathBuf,
    },
    
    /// Show version info
    Version,
}

fn main() {
    let cli = Cli::parse();
    
    match cli.command {
        Commands::Process { file, output } => {
            if cli.verbose {
                println!("Processing {:?} -> {:?}", file, output);
            }
            // Process logic
        }
        Commands::Validate { config } => {
            // Validation logic
        }
        Commands::Version => {
            println!("Version: {}", env!("CARGO_PKG_VERSION"));
        }
    }
}

// ✅ GOOD: Nested subcommands
#[derive(Subcommand)]
enum Commands {
    /// Database operations
    Db {
        #[command(subcommand)]
        command: DbCommands,
    },
}

#[derive(Subcommand)]
enum DbCommands {
    /// Migrate the database
    Migrate,
    /// Seed the database
    Seed,
}

// ❌ BAD: No subcommands for multi-function tools
#[derive(Parser)]
struct Cli {
    #[arg(long)]
    process: bool,
    
    #[arg(long)]
    validate: bool,
    // Hard to manage, confusing UX
}
```

**Usage examples**:
```bash
$ my-tool process input.txt -o output.txt
$ my-tool validate config.toml
$ my-tool version
$ my-tool --verbose process input.txt
$ my-tool db migrate
```

**Rationale**: Subcommands organize multiple operations cleanly, following the pattern established by Git, Cargo, and other modern CLI tools. They scale better than multiple flags and provide clearer help text.

**See also**:
- CLI-14: Help text and documentation
- [External: Cargo command structure](https://doc.rust-lang.org/cargo/)

---

## CLI-11: Value Validation and Custom Parsers

**Strength**: SHOULD

**Summary**: Use `value_parser` for type validation and custom parsing logic.

```rust
use clap::Parser;
use std::path::PathBuf;

// ✅ GOOD: Built-in range validation
#[derive(Parser)]
struct Cli {
    /// Port number (1-65535)
    #[arg(short, long)]
    #[arg(value_parser = clap::value_parser!(u16).range(1..))]
    port: u16,
}

// ✅ GOOD: Enum for restricted values
use clap::ValueEnum;

#[derive(Debug, Clone, Copy, ValueEnum)]
enum Format {
    Json,
    Yaml,
    Toml,
}

#[derive(Parser)]
struct Cli {
    /// Output format
    #[arg(short, long, value_enum)]
    format: Format,
}

// ✅ GOOD: Custom validation function
use anyhow::{Context, Result};

fn parse_percentage(s: &str) -> Result<f64> {
    let value: f64 = s.parse()
        .context("Must be a number")?;
    
    if !(0.0..=100.0).contains(&value) {
        anyhow::bail!("Must be between 0 and 100");
    }
    
    Ok(value)
}

#[derive(Parser)]
struct Cli {
    /// Percentage value (0-100)
    #[arg(short, long, value_parser = parse_percentage)]
    percent: f64,
}

// ✅ GOOD: File existence validation
use clap::Parser;

fn existing_file(s: &str) -> Result<PathBuf, String> {
    let path = PathBuf::from(s);
    if path.exists() && path.is_file() {
        Ok(path)
    } else {
        Err(format!("File does not exist: {}", s))
    }
}

#[derive(Parser)]
struct Cli {
    /// Input file (must exist)
    #[arg(value_parser = existing_file)]
    input: PathBuf,
}

// ❌ BAD: No validation
#[derive(Parser)]
struct Cli {
    port: u16,  // Could be 0 or invalid
}

// ❌ BAD: Validation after parsing
#[derive(Parser)]
struct Cli {
    port: u16,
}

fn main() {
    let cli = Cli::parse();
    
    if cli.port == 0 {
        eprintln!("Port cannot be 0");
        std::process::exit(1);
    }
    // Should validate during parsing!
}
```

**Rationale**: Validating during argument parsing provides immediate, clear feedback to users. Custom parsers enable domain-specific validation that goes beyond basic type checking.

**See also**:
- CLI-17: User-friendly error messages
- [03-error-handling.md](../03-error-handling.md) - Error handling patterns

---

## CLI-12: Environment Variable Integration

**Strength**: CONSIDER

**Summary**: Use `#[arg(env = "VAR_NAME")]` to allow configuration via environment variables.

```rust
use clap::Parser;
use std::path::PathBuf;

// ✅ GOOD: Environment variable fallback
#[derive(Parser)]
struct Cli {
    /// API token
    #[arg(long, env = "MY_TOOL_API_TOKEN")]
    token: String,
    
    /// Config file location
    #[arg(long, env = "MY_TOOL_CONFIG")]
    #[arg(default_value = "config.toml")]
    config: PathBuf,
}

// ✅ GOOD: Hide sensitive values in help
#[derive(Parser)]
struct Cli {
    /// API token (can also use MY_TOOL_TOKEN env var)
    #[arg(long, env = "MY_TOOL_TOKEN")]
    #[arg(hide_env_values = true)]
    token: String,
}

// ✅ GOOD: Optional env var
#[derive(Parser)]
struct Cli {
    /// Database URL
    #[arg(long, env = "DATABASE_URL")]
    database_url: Option<String>,
}

// ❌ BAD: Inconsistent env var naming
#[derive(Parser)]
struct Cli {
    #[arg(long, env = "token")]  // Should be MY_TOOL_TOKEN
    token: String,
    
    #[arg(long, env = "API_KEY")]  // Inconsistent prefix
    api_key: String,
}

// ✅ GOOD: Consistent naming convention
#[derive(Parser)]
#[command(name = "my-tool")]
struct Cli {
    #[arg(long, env = "MY_TOOL_TOKEN")]
    token: String,
    
    #[arg(long, env = "MY_TOOL_API_KEY")]
    api_key: String,
    
    #[arg(long, env = "MY_TOOL_ENDPOINT")]
    endpoint: String,
}
```

**Environment variable naming conventions**:
- Use `SCREAMING_SNAKE_CASE`
- Prefix with tool name: `MY_TOOL_*`
- Be consistent across all variables

**Usage examples**:
```bash
# Command line takes precedence
$ MY_TOOL_TOKEN=secret my-tool --token override

# Environment variable used if flag not provided
$ MY_TOOL_TOKEN=secret my-tool

# No env var or flag = error
$ my-tool
Error: required option --token not provided
```

**Rationale**: Environment variables are standard for configuration in CLI tools and CI/CD systems. They're especially useful for secrets and deployment-specific settings.

**See also**:
- CLI-31: Configuration precedence (defaults → file → env → args)
- CLI-29: Configuration file formats

---

## CLI-13: Argument Groups and Relationships

**Strength**: CONSIDER

**Summary**: Use `requires`, `conflicts_with`, and `groups` to express argument dependencies.

```rust
use clap::{Parser, Args};

// ✅ GOOD: Mutually exclusive options
#[derive(Parser)]
struct Cli {
    /// Output in JSON format
    #[arg(long, conflicts_with = "yaml")]
    json: bool,
    
    /// Output in YAML format
    #[arg(long, conflicts_with = "json")]
    yaml: bool,
}

// ✅ GOOD: Required together
#[derive(Parser)]
struct Cli {
    /// Username
    #[arg(long, requires = "password")]
    username: Option<String>,
    
    /// Password
    #[arg(long, requires = "username")]
    password: Option<String>,
}

// ✅ GOOD: Argument groups
#[derive(Parser)]
struct Cli {
    /// Input from file
    #[arg(short, long, group = "input")]
    file: Option<PathBuf>,
    
    /// Input from URL
    #[arg(short, long, group = "input")]
    url: Option<String>,
    
    /// Process stdin
    #[arg(long, group = "input")]
    stdin: bool,
}
// Exactly one from "input" group required

// ✅ GOOD: Complex relationships
#[derive(Parser)]
struct Cli {
    /// Enable caching
    #[arg(long)]
    cache: bool,
    
    /// Cache directory (requires --cache)
    #[arg(long, requires = "cache")]
    cache_dir: Option<PathBuf>,
    
    /// Clear cache (requires --cache, conflicts with cache_dir)
    #[arg(long, requires = "cache", conflicts_with = "cache_dir")]
    clear_cache: bool,
}
```

**Rationale**: Expressing argument relationships in the schema provides clear error messages and prevents invalid combinations at parse time rather than after.

**See also**:
- CLI-17: User-friendly error messages
- [External: Clap Argument Relations](https://docs.rs/clap/latest/clap/_derive/index.html#arg-attributes)

---

## CLI-14: Help Text and Documentation

**Strength**: MUST

**Summary**: Provide clear help text using doc comments and attributes. Help is your user's primary documentation.

```rust
use clap::Parser;

// ✅ GOOD: Comprehensive help text
/// A tool for processing data files
///
/// This tool reads data files, processes them according to rules,
/// and outputs the results. See the manual for full documentation.
#[derive(Parser)]
#[command(name = "my-tool")]
#[command(version)]
#[command(about, long_about = None)]
#[command(author)]
struct Cli {
    /// Input file to process
    ///
    /// The file must be in JSON or YAML format. Use '-' for stdin.
    input: PathBuf,
    
    /// Output file path
    ///
    /// If not specified, outputs to stdout
    #[arg(short, long)]
    output: Option<PathBuf>,
    
    /// Verbose output
    ///
    /// Repeat for more verbosity: -v (warn), -vv (info), -vvv (debug)
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,
    
    /// Processing mode
    #[arg(short, long, default_value = "fast")]
    #[arg(value_parser = ["fast", "thorough", "paranoid"])]
    mode: String,
}

// ✅ GOOD: Using cargo metadata
#[derive(Parser)]
#[command(author, version, about)]  // Pulls from Cargo.toml
struct Cli {
    // ...
}

// ✅ GOOD: Custom examples in help
#[derive(Parser)]
#[command(after_help = "EXAMPLES:\n  my-tool input.txt -o output.txt\n  cat input.txt | my-tool - > output.txt")]
struct Cli {
    // ...
}

// ❌ BAD: No help text
#[derive(Parser)]
struct Cli {
    input: PathBuf,  // User has no idea what this is for
}

// ❌ BAD: Vague help text
#[derive(Parser)]
struct Cli {
    /// The file
    input: PathBuf,  // What kind of file? What format?
}
```

**Help text guidelines**:
- Start with action verbs: "Process", "Generate", "Convert"
- Specify formats, constraints, and valid values
- Mention defaults explicitly
- Add examples in `after_help` for complex tools
- Keep line length under 80 characters

**Rationale**: Users will run `--help` frequently. Clear, comprehensive help text reduces confusion and support burden.

**See also**:
- [13-documentation.md](../13-documentation.md) - Documentation patterns
- CLI-15: Version information

---

## CLI-15: Version Information

**Strength**: MUST

**Summary**: Always include version information using `#[command(version)]` or explicit version strings.

```rust
use clap::Parser;

// ✅ GOOD: Auto version from Cargo.toml
#[derive(Parser)]
#[command(version)]  // Uses CARGO_PKG_VERSION
struct Cli {
    // ...
}

// ✅ GOOD: Custom version string
#[derive(Parser)]
#[command(version = concat!(
    env!("CARGO_PKG_VERSION"),
    " (",
    env!("CARGO_PKG_NAME"),
    " ",
    env!("GIT_HASH"),  // If you set this in build.rs
    ")"
))]
struct Cli {
    // ...
}

// ✅ GOOD: Long version with details
#[derive(Parser)]
#[command(version)]
#[command(long_version = concat!(
    "Version: ", env!("CARGO_PKG_VERSION"), "\n",
    "Authors: ", env!("CARGO_PKG_AUTHORS"), "\n",
    "Git commit: ", env!("GIT_HASH"), "\n",
    "Build date: ", env!("BUILD_DATE")
))]
struct Cli {
    // ...
}

// ❌ BAD: No version information
#[derive(Parser)]
struct Cli {
    // ...  
}
// Users run --version and get nothing!
```

**Usage**:
```bash
$ my-tool --version
my-tool 1.2.3

$ my-tool --long-version
Version: 1.2.3
Authors: John Doe <john@example.com>
Git commit: abc123
Build date: 2024-01-15
```

**Rationale**: Version information is essential for bug reports, debugging, and tracking deployments. Users expect `--version` to work.

**See also**:
- CLI-14: Help text and documentation
- [External: build.rs for build-time info](https://doc.rust-lang.org/cargo/reference/build-scripts.html)

---

## Best Practices Summary

| Pattern | Strength | Key Takeaway |
|---------|----------|--------------|
| CLI-05 | MUST | Always use clap derive, never std::env::args |
| CLI-06 | MUST | Plain types for required args, Option for optional |
| CLI-07 | SHOULD | Use default_value for sensible defaults |
| CLI-08 | MUST | Use bool for flags, support both short and long forms |
| CLI-09 | SHOULD | Vec<T> for multiple values, consider delimiters |
| CLI-10 | SHOULD | Subcommands for multi-function tools |
| CLI-11 | SHOULD | Validate during parsing with value_parser |
| CLI-12 | CONSIDER | Environment variables for deployment config |
| CLI-13 | CONSIDER | Express arg relationships with requires/conflicts_with |
| CLI-14 | MUST | Comprehensive help text is essential |
| CLI-15 | MUST | Always include version information |

## Related Guidelines

- [02-api-design.md](../02-api-design.md) - Struct design principles
- [03-error-handling.md](../03-error-handling.md) - Error handling
- CLI-01 through CLI-04: Project setup (previous file)
- CLI-16 through CLI-20: Error handling (next file)

## External References

- [Clap Derive Tutorial](https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html)
- [Clap Derive Reference](https://docs.rs/clap/latest/clap/_derive/index.html)
- [ValueEnum Documentation](https://docs.rs/clap/latest/clap/trait.ValueEnum.html)
