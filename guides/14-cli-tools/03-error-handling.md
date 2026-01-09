# CLI Error Handling

Patterns for handling and reporting errors in command-line applications.

---

## CLI-16: Exit Codes (0 for Success, Non-Zero for Errors)

**Strength**: MUST

**Summary**: Always exit with code 0 for success and non-zero codes for errors. Follow standard Unix conventions.

```rust
use std::process::exit;

// ✅ GOOD: Explicit exit codes
fn main() {
    match run() {
        Ok(()) => exit(0),
        Err(e) => {
            eprintln!("Error: {}", e);
            exit(1);
        }
    }
}

// ✅ GOOD: Different exit codes for different errors
use std::process::exit;

fn main() {
    match run() {
        Ok(()) => exit(0),
        Err(e) => {
            eprintln!("Error: {}", e);
            let exit_code = match e.kind() {
                ErrorKind::InvalidInput => 2,
                ErrorKind::PermissionDenied => 77,  // EX_NOPERM
                ErrorKind::NotFound => 66,  // EX_NOINPUT
                ErrorKind::ConfigError => 78,  // EX_CONFIG
                _ => 1,
            };
            exit(exit_code);
        }
    }
}

// ✅ GOOD: Using exitcode crate for standard codes
use exitcode;

fn main() {
    match run() {
        Ok(()) => exit(exitcode::OK),
        Err(e) if e.is_config_error() => {
            eprintln!("Configuration error: {}", e);
            exit(exitcode::CONFIG);
        }
        Err(e) if e.is_usage_error() => {
            eprintln!("Usage error: {}", e);
            exit(exitcode::USAGE);
        }
        Err(e) => {
            eprintln!("Error: {}", e);
            exit(exitcode::SOFTWARE);
        }
    }
}

// ❌ BAD: No exit code handling
fn main() {
    run().unwrap();  // Exits with 101 on panic!
}

// ❌ BAD: Success but exits with error code
fn main() {
    do_work();
    exit(1);  // Wrong! Should be 0
}

// ❌ BAD: Swallowing errors
fn main() {
    let _ = run();  // Error ignored, exits with 0
}
```

**Standard exit codes** (from BSD `sysexits.h`):
- `0` - Success
- `1` - General error
- `2` - Misuse of shell command (usually for argument errors)
- `64` - Command line usage error (EX_USAGE)
- `65` - Data format error (EX_DATAERR)
- `66` - Cannot open input (EX_NOINPUT)
- `70` - Internal software error (EX_SOFTWARE)
- `73` - Can't create output file (EX_CANTCREAT)
- `74` - I/O error (EX_IOERR)
- `77` - Permission denied (EX_NOPERM)
- `78` - Configuration error (EX_CONFIG)

**Rationale**: Scripts and CI/CD systems rely on exit codes to determine success or failure. Using standard codes enables better automation and debugging.

**See also**:
- CLI-19: Distinguish user errors from system errors
- [External: exitcode crate](https://docs.rs/exitcode)
- [External: BSD sysexits](https://man.freebsd.org/cgi/man.cgi?query=sysexits)

---

## CLI-17: User-Friendly Error Messages

**Strength**: MUST

**Summary**: Error messages should be clear, actionable, and written for end users, not developers.

```rust
use anyhow::{Context, Result};

// ❌ BAD: Technical error messages
fn process_file(path: &Path) -> Result<()> {
    let file = File::open(path)?;  // Error: "No such file or directory"
    Ok(())
}

// ✅ GOOD: User-friendly with context
fn process_file(path: &Path) -> Result<()> {
    let file = File::open(path)
        .with_context(|| format!("Failed to open file: {}", path.display()))?;
    Ok(())
}
// Error: Failed to open file: /path/to/missing.txt
//   Caused by: No such file or directory

// ✅ GOOD: Actionable error messages
fn validate_config(config: &Config) -> Result<()> {
    if config.port == 0 {
        anyhow::bail!(
            "Invalid port number: 0\n\
             \n\
             Port must be between 1 and 65535.\n\
             Check your config file or use --port <NUMBER>"
        );
    }
    Ok(())
}

// ✅ GOOD: Suggest corrections
use anyhow::bail;

fn load_format(name: &str) -> Result<Format> {
    match name.to_lowercase().as_str() {
        "json" => Ok(Format::Json),
        "yaml" | "yml" => Ok(Format::Yaml),
        "toml" => Ok(Format::Toml),
        other => {
            bail!(
                "Unknown format: {}\n\
                 \n\
                 Supported formats: json, yaml, toml\n\
                 Did you mean 'json'?",
                other
            );
        }
    }
}

// ✅ GOOD: Include what you were trying to do
fn save_results(path: &Path, data: &str) -> Result<()> {
    std::fs::write(path, data)
        .with_context(|| {
            format!(
                "Failed to save results to {}\n\
                 \n\
                 Make sure you have write permissions and sufficient disk space.",
                path.display()
            )
        })?;
    Ok(())
}

// ❌ BAD: Unclear error
fn main() {
    if let Err(e) = run() {
        eprintln!("{}", e);  // Just "invalid input"
        exit(1);
    }
}

// ✅ GOOD: Full error chain
fn main() {
    if let Err(e) = run() {
        eprintln!("Error: {}", e);
        
        let mut source = e.source();
        while let Some(cause) = source {
            eprintln!("  Caused by: {}", cause);
            source = cause.source();
        }
        
        exit(1);
    }
}
```

**Error message guidelines**:
1. Start with what went wrong: "Failed to open file"
2. Include the specific item: "config.toml"
3. Explain why: "No such file or directory"
4. Suggest action: "Check the path and try again"

**Rationale**: Good error messages help users fix problems themselves. Technical error messages from the stdlib are often not helpful without context.

**See also**:
- CLI-18: Error context for CLI operations
- [03-error-handling.md](../03-error-handling.md#eh-04) - Error context patterns

---

## CLI-18: Error Context for CLI Operations

**Strength**: SHOULD

**Summary**: Add context to errors using `anyhow` or custom error types with context.

```rust
use anyhow::{Context, Result};
use std::fs;
use std::path::Path;

// ✅ GOOD: Context at each layer
fn process_directory(dir: &Path) -> Result<()> {
    let entries = fs::read_dir(dir)
        .with_context(|| format!("Failed to read directory: {}", dir.display()))?;
    
    for entry in entries {
        let entry = entry
            .with_context(|| format!("Failed to read directory entry in {}", dir.display()))?;
        
        let path = entry.path();
        process_file(&path)
            .with_context(|| format!("Failed to process file: {}", path.display()))?;
    }
    
    Ok(())
}

fn process_file(path: &Path) -> Result<()> {
    let contents = fs::read_to_string(path)
        .with_context(|| format!("Failed to read file: {}", path.display()))?;
    
    let data: Data = serde_json::from_str(&contents)
        .with_context(|| format!("Failed to parse JSON from: {}", path.display()))?;
    
    validate_data(&data)
        .with_context(|| format!("Validation failed for: {}", path.display()))?;
    
    Ok(())
}

// ✅ GOOD: Custom error type with context
use thiserror::Error;

#[derive(Error, Debug)]
pub enum CliError {
    #[error("Failed to read configuration from {path}")]
    ConfigRead {
        path: String,
        #[source]
        source: std::io::Error,
    },
    
    #[error("Invalid configuration in {path}: {message}")]
    ConfigInvalid {
        path: String,
        message: String,
    },
    
    #[error("Failed to connect to {endpoint}")]
    ConnectionFailed {
        endpoint: String,
        #[source]
        source: reqwest::Error,
    },
}

// ❌ BAD: No context
fn process_file(path: &Path) -> Result<()> {
    let contents = fs::read_to_string(path)?;  // Error: "No such file"
    let data: Data = serde_json::from_str(&contents)?;  // Error: "missing field"
    Ok(())
}

// ✅ GOOD: Clear error chain in output
// $ my-tool process data.json
// Error: Failed to process file: data.json
//   Caused by: Failed to parse JSON from: data.json
//   Caused by: missing field `name` at line 5 column 3
```

**Rationale**: Context helps users and developers understand exactly where and why an error occurred. Anyhow's error chain provides a breadcrumb trail from high-level operation to low-level cause.

**See also**:
- [03-error-handling.md](../03-error-handling.md) - Error handling patterns
- [External: anyhow documentation](https://docs.rs/anyhow)

---

## CLI-19: Distinguish User Errors from System Errors

**Strength**: SHOULD

**Summary**: Clearly distinguish between user errors (bad input) and system errors (bugs, I/O failures).

```rust
use thiserror::Error;

// ✅ GOOD: Separate user and system error types
#[derive(Error, Debug)]
pub enum CliError {
    // User errors - exit code 2 or specific usage codes
    #[error("Invalid port number: {0}. Must be between 1 and 65535")]
    InvalidPort(u16),
    
    #[error("Unsupported format: {0}. Use 'json', 'yaml', or 'toml'")]
    UnsupportedFormat(String),
    
    #[error("File not found: {0}\n\nCheck the path and try again")]
    FileNotFound(String),
    
    // System errors - exit code 1 or 70
    #[error("I/O error")]
    Io(#[from] std::io::Error),
    
    #[error("JSON parsing error")]
    Json(#[from] serde_json::Error),
    
    #[error("Internal error: {0}\n\nThis is a bug. Please report it.")]
    Internal(String),
}

impl CliError {
    pub fn exit_code(&self) -> i32 {
        match self {
            // User errors
            Self::InvalidPort(_) => 2,
            Self::UnsupportedFormat(_) => 2,
            Self::FileNotFound(_) => 66,  // EX_NOINPUT
            
            // System errors
            Self::Io(_) => 74,  // EX_IOERR
            Self::Json(_) => 65,  // EX_DATAERR
            Self::Internal(_) => 70,  // EX_SOFTWARE
        }
    }
    
    pub fn is_user_error(&self) -> bool {
        matches!(
            self,
            Self::InvalidPort(_) | Self::UnsupportedFormat(_) | Self::FileNotFound(_)
        )
    }
}

// ✅ GOOD: Different handling for user vs system errors
fn main() {
    if let Err(e) = run() {
        if e.is_user_error() {
            eprintln!("{}", e);
            eprintln!("\nRun with --help for usage information.");
        } else {
            eprintln!("Error: {}", e);
            eprintln!("\nIf this problem persists, please file a bug report.");
        }
        
        exit(e.exit_code());
    }
}

// ❌ BAD: All errors look the same
fn main() {
    if let Err(e) = run() {
        eprintln!("Error: {}", e);  // User doesn't know if it's their fault
        exit(1);
    }
}
```

**User error characteristics**:
- Caused by invalid input or usage
- User can fix by changing input
- Examples: wrong format, missing file, invalid value

**System error characteristics**:
- Caused by system state or bugs
- User cannot directly fix
- Examples: out of memory, disk full, network timeout, bugs

**Rationale**: Distinguishing error types helps users understand if they need to change their input or if there's a deeper problem. It also helps with exit code selection.

**See also**:
- CLI-16: Exit codes
- CLI-17: User-friendly error messages

---

## CLI-20: Error Output to Stderr

**Strength**: MUST

**Summary**: Always write errors to stderr, not stdout. This enables proper piping and output redirection.

```rust
// ❌ BAD: Errors to stdout
fn main() {
    if let Err(e) = run() {
        println!("Error: {}", e);  // WRONG! Goes to stdout
        exit(1);
    }
}

// ✅ GOOD: Errors to stderr
fn main() {
    if let Err(e) = run() {
        eprintln!("Error: {}", e);  // Correct! Goes to stderr
        exit(1);
    }
}

// ✅ GOOD: Structured error output
fn main() {
    if let Err(e) = run() {
        // Main error
        eprintln!("Error: {}", e);
        
        // Cause chain
        let mut source = e.source();
        while let Some(cause) = source {
            eprintln!("  Caused by: {}", cause);
            source = cause.source();
        }
        
        exit(1);
    }
}

// ✅ GOOD: Warnings also go to stderr
fn process_file(path: &Path) -> Result<()> {
    if path.extension() != Some("txt") {
        eprintln!("Warning: {} is not a .txt file", path.display());
    }
    // ... process anyway
    Ok(())
}

// ❌ BAD: Mixed output streams
fn process_file(path: &Path) -> Result<()> {
    println!("Processing {}...", path.display());  // Status to stdout
    
    if let Err(e) = validate(path) {
        println!("Error: {}", e);  // Error to stdout - WRONG!
        return Err(e);
    }
    
    Ok(())
}

// ✅ GOOD: Consistent stream usage
fn process_file(path: &Path, verbose: bool) -> Result<()> {
    // Status/progress to stderr
    if verbose {
        eprintln!("Processing {}...", path.display());
    }
    
    // Results to stdout
    let result = process(path)?;
    println!("{}", result);
    
    Ok(())
}
```

**Why stderr for errors**:
```bash
# ✅ Redirect output without capturing errors
$ my-tool input.txt > output.txt
# Errors still visible on screen

# ✅ Capture only errors
$ my-tool input.txt 2> errors.txt
# Output to screen, errors to file

# ✅ Pipe output to another command
$ my-tool input.txt | grep pattern
# Errors still visible, only output is piped

# ❌ If errors go to stdout:
$ my-tool input.txt | grep pattern
# Errors are piped to grep! Probably not what you want
```

**Stream usage guidelines**:
- **stdout**: Primary output data (results, formatted output)
- **stderr**: Errors, warnings, progress, status messages
- **stdin**: Input data (when reading from pipe)

**Rationale**: Separating output streams is a Unix convention that enables powerful composition and redirection. Mixing error messages with output data breaks pipeline workflows.

**See also**:
- CLI-27: Stdout vs stderr usage
- CLI-48: Piping and stdin/stdout integration

---

## Best Practices Summary

| Pattern | Strength | Key Takeaway |
|---------|----------|--------------|
| CLI-16 | MUST | Exit 0 for success, non-zero for errors |
| CLI-17 | MUST | Error messages must be clear and actionable |
| CLI-18 | SHOULD | Add context with anyhow or custom types |
| CLI-19 | SHOULD | Distinguish user errors from system errors |
| CLI-20 | MUST | Always write errors to stderr, not stdout |

## Related Guidelines

- [03-error-handling.md](../03-error-handling.md) - General error handling patterns
- CLI-27: Stdout vs stderr usage (in 04-output-and-ux.md)
- CLI-34: Integration testing with assert_cmd (in 06-testing.md)

## External References

- [exitcode crate](https://docs.rs/exitcode) - Standard exit codes
- [anyhow documentation](https://docs.rs/anyhow) - Ergonomic error handling
- [thiserror documentation](https://docs.rs/thiserror) - Custom error types
- [BSD sysexits](https://man.freebsd.org/cgi/man.cgi?query=sysexits) - Standard exit codes
