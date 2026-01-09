# CLI Output and User Experience

Patterns for creating polished, user-friendly command-line interfaces.

---

## CLI-21: Human vs Machine-Readable Output

**Strength**: SHOULD

**Summary**: Provide options for both human-friendly and machine-readable output formats.

```rust
use clap::Parser;
use serde::Serialize;

#[derive(Parser)]
struct Cli {
    /// Output format
    #[arg(long, default_value = "human")]
    #[arg(value_parser = ["human", "json", "csv"])]
    format: String,
}

#[derive(Serialize)]
struct Result {
    filename: String,
    size: u64,
    modified: String,
}

// ✅ GOOD: Switch output format based on flag
fn display_results(results: &[Result], format: &str) {
    match format {
        "json" => {
            println!("{}", serde_json::to_string_pretty(results).unwrap());
        }
        "csv" => {
            println!("filename,size,modified");
            for r in results {
                println!("{},{},{}", r.filename, r.size, r.modified);
            }
        }
        "human" | _ => {
            for r in results {
                println!("{:20} {:>10} bytes  {}", r.filename, r.size, r.modified);
            }
        }
    }
}

// ✅ GOOD: Auto-detect when piped
use std::io::IsTerminal;

fn main() {
    let cli = Cli::parse();
    let results = get_results();
    
    let format = if std::io::stdout().is_terminal() {
        // Interactive terminal, use human format
        &cli.format
    } else {
        // Piped or redirected, use machine-readable
        "json"
    };
    
    display_results(&results, format);
}

// ✅ GOOD: Quiet mode for scripting
#[derive(Parser)]
struct Cli {
    /// Quiet mode - only output essential data
    #[arg(short, long)]
    quiet: bool,
}

fn main() {
    let cli = Cli::parse();
    
    if !cli.quiet {
        eprintln!("Processing files...");  // To stderr
    }
    
    let result = process();
    println!("{}", result);  // Output to stdout
    
    if !cli.quiet {
        eprintln!("Done!");
    }
}

// ❌ BAD: Only human-readable format
fn display_results(results: &[Result]) {
    for r in results {
        println!("File: {}", r.filename);
        println!("  Size: {} bytes", r.size);
        println!("  Modified: {}", r.modified);
        println!();
    }
    // Hard to parse programmatically!
}
```

**Machine-readable format guidelines**:
- **JSON**: Best for complex nested data
- **CSV**: Best for tabular data
- **TSV**: Like CSV but easier to parse
- **Line-delimited JSON**: One JSON object per line

**Rationale**: CLI tools are often used in scripts and pipelines. Supporting machine-readable output enables automation while keeping human-friendly defaults.

**See also**:
- CLI-27: Stdout vs stderr usage
- CLI-28: Buffered vs unbuffered output

---

## CLI-22: Progress Indicators for Long Operations

**Strength**: CONSIDER

**Summary**: Show progress for operations that take more than a few seconds using progress bars or spinners.

```rust
use indicatif::{ProgressBar, ProgressStyle};

// ✅ GOOD: Progress bar for known amount of work
fn process_files(files: &[PathBuf]) -> Result<()> {
    let pb = ProgressBar::new(files.len() as u64);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("[{elapsed_precise}] {bar:40.cyan/blue} {pos}/{len} {msg}")?
            .progress_chars("##-")
    );
    
    for file in files {
        pb.set_message(format!("Processing {}", file.display()));
        process_file(file)?;
        pb.inc(1);
    }
    
    pb.finish_with_message("Done!");
    Ok(())
}

// ✅ GOOD: Spinner for unknown duration
use indicatif::{ProgressBar, ProgressStyle};

fn download_file(url: &str) -> Result<()> {
    let spinner = ProgressBar::new_spinner();
    spinner.set_style(
        ProgressStyle::default_spinner()
            .template("{spinner:.green} {msg}")?
    );
    spinner.set_message(format!("Downloading {}...", url));
    spinner.enable_steady_tick(Duration::from_millis(100));
    
    let data = fetch(url)?;
    
    spinner.finish_with_message(format!("Downloaded {} bytes", data.len()));
    Ok(())
}

// ✅ GOOD: Only show progress if terminal is interactive
use std::io::IsTerminal;

fn process_files(files: &[PathBuf]) -> Result<()> {
    let pb = if std::io::stderr().is_terminal() {
        let pb = ProgressBar::new(files.len() as u64);
        pb.set_style(/* ... */);
        Some(pb)
    } else {
        None  // Don't show progress in scripts
    };
    
    for file in files {
        if let Some(pb) = &pb {
            pb.set_message(format!("Processing {}", file.display()));
        }
        process_file(file)?;
        if let Some(pb) = &pb {
            pb.inc(1);
        }
    }
    
    if let Some(pb) = pb {
        pb.finish_with_message("Done!");
    }
    
    Ok(())
}

// ❌ BAD: No feedback for long operations
fn process_files(files: &[PathBuf]) -> Result<()> {
    for file in files {
        process_file(file)?;  // User sees nothing for minutes
    }
    Ok(())
}

// ❌ BAD: Too much output
fn process_files(files: &[PathBuf]) -> Result<()> {
    for file in files {
        println!("Processing {}...", file.display());
        println!("  Reading file...");
        println!("  Parsing data...");
        println!("  Validating...");
        println!("  Writing output...");
        // Floods the terminal!
    }
    Ok(())
}
```

**Rationale**: Progress feedback improves perceived performance and helps users understand when operations will complete. But it should only appear on interactive terminals, not in logs or scripts.

**See also**:
- [External: indicatif](https://docs.rs/indicatif) - Progress bars and spinners
- CLI-24: Verbosity levels and logging

---

## CLI-23: Colored Output with Terminal Detection

**Strength**: CONSIDER

**Summary**: Use colors to highlight important information, but detect terminal capabilities and provide options to disable.

```rust
use colored::*;

// ✅ GOOD: Colors with terminal detection
#[derive(Parser)]
struct Cli {
    /// Disable colored output
    #[arg(long)]
    no_color: bool,
}

fn main() {
    let cli = Cli::parse();
    
    // Respect NO_COLOR env var and --no-color flag
    if cli.no_color || std::env::var("NO_COLOR").is_ok() {
        colored::control::set_override(false);
    }
    
    println!("{}", "Success!".green());
    println!("{}", "Warning: check config".yellow());
    println!("{}", "Error: file not found".red());
}

// ✅ GOOD: Automatic color detection
use std::io::IsTerminal;
use colored::*;

fn main() {
    // Only enable colors if stdout is a terminal
    if !std::io::stdout().is_terminal() {
        colored::control::set_override(false);
    }
    
    println!("{}", "This will be colored only in terminals".green());
}

// ✅ GOOD: Semantic color usage
fn display_status(status: &Status) {
    match status {
        Status::Success => println!("{}", "✓ Success".green()),
        Status::Warning => println!("{}", "⚠ Warning".yellow()),
        Status::Error => println!("{}", "✗ Error".red()),
        Status::Info => println!("{}", "ℹ Info".blue()),
    }
}

// ❌ BAD: Always colored, even when piped
fn main() {
    println!("{}", "Error".red());
    // When piped: Error with ANSI codes in output file
}

// ❌ BAD: Excessive colors
fn main() {
    println!("{}", "Processing".cyan());
    println!("{}", "file.txt".magenta());
    println!("{}", "with".green());
    println!("{}", "options".yellow());
    // Too much! Distracting and hard to read
}

// ✅ GOOD: Minimal strategic colors
fn main() {
    println!("Processing file.txt...");
    println!("{}: Operation completed", "Success".green());
    // Colors only for status, easy to read
}
```

**Color usage guidelines**:
- **Green**: Success, completion
- **Yellow/Amber**: Warnings, important info
- **Red**: Errors, failures
- **Blue**: Informational messages
- **Dim/Gray**: Less important info
- **Bold**: Emphasis within status messages

**Rationale**: Colors improve scannability and make important information stand out. But they must be disabled in non-terminal contexts (pipes, logs, CI) to avoid ANSI escape codes in output.

**See also**:
- [External: colored crate](https://docs.rs/colored)
- [External: NO_COLOR standard](https://no-color.org/)

---

## CLI-24: Verbosity Levels and Logging

**Strength**: SHOULD

**Summary**: Implement verbosity levels using the `log` crate and control output with `-v` flags.

```rust
use clap::Parser;
use env_logger::Env;
use log::{debug, error, info, warn};

#[derive(Parser)]
struct Cli {
    /// Increase verbosity (-v, -vv, -vvv)
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,
}

fn main() {
    let cli = Cli::parse();
    
    // Set log level based on verbosity
    let log_level = match cli.verbose {
        0 => "error",
        1 => "warn",
        2 => "info",
        3 => "debug",
        _ => "trace",
    };
    
    env_logger::Builder::from_env(Env::default().default_filter_or(log_level))
        .init();
    
    info!("Starting application");
    debug!("Debug information");
    
    if let Err(e) = run() {
        error!("Error: {}", e);
        std::process::exit(1);
    }
}

// ✅ GOOD: Appropriate log levels
fn process_file(path: &Path) -> Result<()> {
    info!("Processing file: {}", path.display());
    debug!("File size: {} bytes", path.metadata()?.len());
    
    if is_empty(path) {
        warn!("File is empty: {}", path.display());
    }
    
    let data = read_file(path)?;
    trace!("File contents: {:?}", data);
    
    Ok(())
}

// ✅ GOOD: Environment variable support
fn main() {
    // RUST_LOG=debug my-tool
    // or
    // my-tool -vv
    let cli = Cli::parse();
    
    env_logger::Builder::from_env(
        Env::default()
            .filter_or("RUST_LOG", match cli.verbose {
                0 => "error",
                1 => "warn",
                2 => "info",
                _ => "debug",
            })
    )
    .init();
}

// ❌ BAD: println! for different verbosity levels
fn process_file(path: &Path, verbose: bool) -> Result<()> {
    if verbose {
        println!("Processing {}", path.display());
    }
    // Hard to manage multiple verbosity levels
}

// ❌ BAD: Log spam at default level
fn process_item(item: &str) {
    info!("Starting processing of {}", item);
    info!("Reading data...");
    info!("Parsing data...");
    info!("Validating data...");
    info!("Writing results...");
    // Too much! Use debug for these details
}
```

**Log level guidelines**:
- **error**: Errors that prevent operation
- **warn**: Problems that don't prevent operation
- **info**: High-level progress (default for `-v`)
- **debug**: Detailed progress (for `-vv`)
- **trace**: Very detailed internals (for `-vvv`)

**Rationale**: The `log` crate provides a standard logging facade that users can control. Verbosity flags are a common UX pattern that users expect.

**See also**:
- CLI-22: Progress indicators for long operations
- [External: log crate](https://docs.rs/log)
- [External: env_logger](https://docs.rs/env_logger)

---

## CLI-25: Confirmation Prompts

**Strength**: CONSIDER

**Summary**: Ask for confirmation before destructive operations, with `--force` or `--yes` to skip.

```rust
use std::io::{self, Write};

#[derive(Parser)]
struct Cli {
    /// Skip confirmation prompts
    #[arg(short = 'y', long)]
    yes: bool,
    
    /// Force operation without confirmation
    #[arg(short, long)]
    force: bool,
}

// ✅ GOOD: Confirm destructive operations
fn delete_all(cli: &Cli) -> Result<()> {
    if !cli.yes && !cli.force {
        print!("This will delete all data. Continue? (y/N) ");
        io::stdout().flush()?;
        
        let mut input = String::new();
        io::stdin().read_line(&mut input)?;
        
        if !input.trim().eq_ignore_ascii_case("y") {
            println!("Cancelled");
            return Ok(());
        }
    }
    
    // Perform deletion
    Ok(())
}

// ✅ GOOD: Show what will be deleted
fn delete_files(files: &[PathBuf], cli: &Cli) -> Result<()> {
    println!("Will delete {} files:", files.len());
    for file in files {
        println!("  {}", file.display());
    }
    
    if !cli.yes {
        print!("\nContinue? (y/N) ");
        io::stdout().flush()?;
        
        let mut input = String::new();
        io::stdin().read_line(&mut input)?;
        
        if !input.trim().eq_ignore_ascii_case("y") {
            return Ok(());
        }
    }
    
    for file in files {
        std::fs::remove_file(file)?;
    }
    
    Ok(())
}

// ✅ GOOD: Only prompt if terminal is interactive
fn confirm(message: &str) -> Result<bool> {
    if !std::io::stdin().is_terminal() {
        // Non-interactive, don't prompt
        return Ok(false);
    }
    
    print!("{} (y/N) ", message);
    io::stdout().flush()?;
    
    let mut input = String::new();
    io::stdin().read_line(&mut input)?;
    
    Ok(input.trim().eq_ignore_ascii_case("y"))
}

// ❌ BAD: No confirmation for destructive operation
fn delete_all() -> Result<()> {
    std::fs::remove_dir_all("./data")?;  // Dangerous!
    Ok(())
}

// ❌ BAD: Confirmation in non-interactive mode
fn delete_all() -> Result<()> {
    print!("Continue? (y/N) ");
    // When piped: hangs waiting for input that never comes
    // ...
}
```

**When to prompt**:
- Deleting files or data
- Overwriting existing files
- Destructive database operations
- Irreversible changes

**When not to prompt**:
- Read-only operations
- When `--yes` or `--force` is passed
- When stdin is not a terminal
- In batch/script mode

**Rationale**: Prompts prevent accidental data loss while flags like `--yes` enable automation. Always check for terminal interactivity.

**See also**:
- CLI-48: Piping and stdin/stdout integration

---

## CLI-26: Structured Output (JSON, CSV)

**Strength**: CONSIDER

**Summary**: Provide structured output formats for programmatic consumption.

```rust
use clap::Parser;
use serde::{Serialize, Deserialize};

#[derive(Parser)]
struct Cli {
    /// Output format
    #[arg(short, long, default_value = "table")]
    #[arg(value_parser = ["table", "json", "csv", "yaml"])]
    output: String,
}

#[derive(Serialize)]
struct Item {
    name: String,
    count: u32,
    active: bool,
}

// ✅ GOOD: Multiple output formats
fn display_items(items: &[Item], format: &str) -> Result<()> {
    match format {
        "json" => {
            serde_json::to_writer_pretty(std::io::stdout(), items)?;
            println!();
        }
        "yaml" => {
            serde_yaml::to_writer(std::io::stdout(), items)?;
        }
        "csv" => {
            let mut wtr = csv::Writer::from_writer(std::io::stdout());
            for item in items {
                wtr.serialize(item)?;
            }
            wtr.flush()?;
        }
        "table" | _ => {
            println!("{:<20} {:>8} {:<10}", "Name", "Count", "Active");
            println!("{:-<40}", "");
            for item in items {
                println!("{:<20} {:>8} {:<10}", 
                    item.name, item.count, item.active);
            }
        }
    }
    Ok(())
}

// ✅ GOOD: Line-delimited JSON for streaming
fn stream_results(items: impl Iterator<Item = Item>) -> Result<()> {
    for item in items {
        serde_json::to_writer(std::io::stdout(), &item)?;
        println!();  // Newline after each JSON object
    }
    Ok(())
}

// ❌ BAD: Inconsistent output format
fn display_items(items: &[Item]) {
    println!("Items:");
    for item in items {
        println!("- {}: {} ({})", 
            item.name, 
            item.count, 
            if item.active { "active" } else { "inactive" }
        );
    }
    // Hard to parse programmatically
}
```

**Format selection**:
- **JSON**: Universal, handles nested data, widely supported
- **CSV**: Tabular data, Excel-compatible
- **YAML**: Human-readable, supports comments
- **Line-delimited JSON**: Streaming, one object per line

**Rationale**: Structured output enables CLI tools to be part of data pipelines and automation workflows.

**See also**:
- CLI-21: Human vs machine-readable output
- CLI-27: Stdout vs stderr usage

---

## CLI-27: Stdout vs Stderr Usage

**Strength**: MUST

**Summary**: Use stdout for data output, stderr for everything else (progress, warnings, errors).

```rust
// ✅ GOOD: Clear stream separation
fn process_file(path: &Path, verbose: bool) -> Result<String> {
    // Progress to stderr
    if verbose {
        eprintln!("Processing {}...", path.display());
    }
    
    // Warnings to stderr
    if path.extension() != Some(OsStr::new("txt")) {
        eprintln!("Warning: {} is not a .txt file", path.display());
    }
    
    let result = do_processing(path)?;
    
    // Data to stdout
    println!("{}", result);
    
    Ok(result)
}

// ✅ GOOD: Progress indicators to stderr
use indicatif::ProgressBar;

fn main() -> Result<()> {
    let files = get_files();
    
    // Progress bar writes to stderr by default
    let pb = ProgressBar::new(files.len() as u64);
    
    for file in files {
        pb.inc(1);
        let result = process_file(file)?;
        println!("{}", result);  // Data to stdout
    }
    
    pb.finish();
    Ok(())
}

// ✅ GOOD: Structured data to stdout, messages to stderr
fn main() -> Result<()> {
    eprintln!("Starting analysis...");
    
    let results = analyze()?;
    
    // JSON data to stdout
    serde_json::to_writer_pretty(std::io::stdout(), &results)?;
    println!();
    
    eprintln!("Analysis complete!");
    Ok(())
}

// ❌ BAD: Mixed output
fn main() -> Result<()> {
    println!("Processing files...");  // Should be eprintln!
    
    for file in files {
        let result = process(file)?;
        println!("{}", result);  // Data mixed with status
        println!("Done!");  // Should be eprintln!
    }
    
    Ok(())
}
```

**Stream usage**:
```bash
# ✅ Capture just the data
$ my-tool input.txt > output.json
# Progress/warnings still visible, data in file

# ✅ Capture just errors
$ my-tool input.txt 2> errors.log
# Output to screen, errors in file

# ✅ Pipe to another command
$ my-tool input.txt | jq '.items[]'
# Only data is piped, progress still visible
```

**Rationale**: Unix convention separates data from messages. This enables piping, redirection, and automation without interference.

**See also**:
- CLI-20: Error output to stderr
- CLI-48: Piping and stdin/stdout integration

---

## CLI-28: Buffered vs Unbuffered Output

**Strength**: CONSIDER

**Summary**: Use buffered output for performance, unbuffered for real-time feedback.

```rust
use std::io::{self, Write};

// ✅ GOOD: Buffered output for large data
fn write_large_dataset(data: &[Item]) -> Result<()> {
    let stdout = io::stdout();
    let mut handle = io::BufWriter::new(stdout.lock());
    
    for item in data {
        writeln!(handle, "{}", serde_json::to_string(item)?)?;
    }
    
    handle.flush()?;
    Ok(())
}

// ✅ GOOD: Unbuffered for progress updates
fn process_with_progress(items: &[Item]) -> Result<()> {
    for (i, item) in items.iter().enumerate() {
        eprint!("\rProcessing {}/{}...", i + 1, items.len());
        io::stderr().flush()?;  // Immediate display
        process(item)?;
    }
    eprintln!("\nDone!");
    Ok(())
}

// ✅ GOOD: Lock stdout once for multiple writes
fn print_table(data: &[Item]) {
    let stdout = io::stdout();
    let mut handle = stdout.lock();
    
    writeln!(handle, "{:<20} {:>10}", "Name", "Count").unwrap();
    for item in data {
        writeln!(handle, "{:<20} {:>10}", item.name, item.count).unwrap();
    }
    // Lock automatically released
}

// ❌ BAD: println! in tight loop
fn write_large_dataset(data: &[Item]) {
    for item in data {
        println!("{}", serde_json::to_string(item).unwrap());
        // Each println! locks/unlocks stdout - slow!
    }
}
```

**When to use buffering**:
- Writing large amounts of data
- Performance-critical output
- Batch operations

**When to flush immediately**:
- Progress indicators
- Interactive prompts
- Real-time status updates

**Rationale**: Buffering improves performance by reducing syscalls. But progress indicators need immediate flushing to be visible in real-time.

**See also**:
- CLI-22: Progress indicators for long operations

---

## Best Practices Summary

| Pattern | Strength | Key Takeaway |
|---------|----------|--------------|
| CLI-21 | SHOULD | Support both human and machine-readable output |
| CLI-22 | CONSIDER | Show progress for long operations |
| CLI-23 | CONSIDER | Use colors, but detect terminals and provide --no-color |
| CLI-24 | SHOULD | Implement verbosity with log crate and -v flags |
| CLI-25 | CONSIDER | Confirm destructive operations, respect --yes flag |
| CLI-26 | CONSIDER | Provide structured formats (JSON, CSV) for automation |
| CLI-27 | MUST | Data to stdout, messages/errors to stderr |
| CLI-28 | CONSIDER | Buffer output for performance, flush for real-time |

## Related Guidelines

- CLI-20: Error output to stderr (in 03-error-handling.md)
- CLI-48: Piping and stdin/stdout integration (in 08-advanced-topics.md)

## External References

- [colored crate](https://docs.rs/colored) - Terminal colors
- [indicatif](https://docs.rs/indicatif) - Progress bars
- [log crate](https://docs.rs/log) - Logging facade
- [env_logger](https://docs.rs/env_logger) - Simple logger
- [NO_COLOR standard](https://no-color.org/) - Color opt-out
