# Cargo Plugin Development

Custom cargo subcommands extend Cargo's functionality without modifying Cargo itself. This guide covers patterns for creating, distributing, and integrating cargo plugins.

---

## CG-P-01: Follow the `cargo-*` Naming Convention

**Strength**: MUST

**Summary**: Name plugin binaries `cargo-<command>` to enable invocation as `cargo <command>`.

Cargo translates `cargo <command>` invocations into searches for executables named `cargo-<command>` in the user's PATH. This convention is fundamental to cargo's plugin system.

```rust
// ❌ BAD: Non-standard naming breaks cargo integration
// Binary name: my-cargo-tool
// User must run: ./my-cargo-tool (not integrated with cargo)

// ✅ GOOD: Standard naming enables cargo integration
// In Cargo.toml:
[package]
name = "cargo-example"  # Creates binary: cargo-example

[[bin]]
name = "cargo-example"
path = "src/main.rs"

// User can now run: cargo example
```

**Rationale**: Cargo automatically discovers and invokes executables following the `cargo-*` naming pattern. Tools with non-standard names cannot be invoked as cargo subcommands, forcing users to run them separately and breaking the unified cargo interface. The plugin must also be in `$PATH` or `$CARGO_HOME/bin` (which Cargo prioritizes) to be discovered.

**See also**: CG-P-02 (argument handling), CG-P-08 (distribution)

---

## CG-P-02: Handle Subcommand Name in Arguments

**Strength**: MUST

**Summary**: Expect the subcommand name as the second argument when invoked via cargo.

When Cargo invokes `cargo-foo` via `cargo foo`, it passes the subcommand name (`foo`) as the second argument. Your plugin must handle this to work correctly with cargo's invocation pattern.

```rust
// ❌ BAD: Ignoring the subcommand argument
fn main() {
    let args: Vec<String> = env::args().collect();
    // args[1] might be "foo" or actual flags
    let config_path = &args[1];  // Wrong!
}

// ✅ GOOD: Properly handling subcommand name
use clap::Parser;

#[derive(Parser)]
#[command(bin_name = "cargo")]
enum Cargo {
    #[command(name = "example")]
    Example(Args),
}

#[derive(Parser)]
struct Args {
    #[arg(long)]
    config: Option<PathBuf>,
}

fn main() {
    let Cargo::Example(args) = Cargo::parse();
    // Now args contains actual arguments, not "example"
}

// ✅ ALSO GOOD: Manual argument parsing
fn main() {
    let mut args: Vec<String> = env::args().collect();
    
    // When invoked as "cargo foo", args[1] is "foo"
    // When invoked as "cargo-foo", args[1] is the first real arg
    if args.len() > 1 && args[1] == "foo" {
        args.remove(1);  // Remove subcommand name
    }
    
    // Now process remaining arguments
}
```

**Rationale**: Cargo passes the subcommand name as an argument to maintain consistency with how cargo's built-in subcommands work. Failing to account for this causes plugins to misinterpret their first real argument as the subcommand name. Using `clap` with `bin_name = "cargo"` handles this automatically.

**See also**: CG-P-03 (help integration)

---

## CG-P-03: Implement `--help` for Cargo Help Integration

**Strength**: MUST

**Summary**: Support `--help` as the third argument to integrate with `cargo help <command>`.

Cargo assumes plugins print help when their third argument is `--help`, enabling `cargo help example` to display your plugin's help text.

```rust
// ❌ BAD: No help support
fn main() {
    let args: Vec<String> = env::args().collect();
    // No --help handling; cargo help example fails
}

// ✅ GOOD: Proper help integration with clap
use clap::Parser;

#[derive(Parser)]
#[command(
    name = "cargo-example",
    bin_name = "cargo",
    about = "Example cargo plugin",
    long_about = "A detailed description of what this plugin does."
)]
enum Cargo {
    #[command(name = "example")]
    Example(Args),
}

// clap automatically handles --help
// Both "cargo example --help" and "cargo help example" work

// ✅ ALSO GOOD: Manual help handling
fn main() {
    let args: Vec<String> = env::args().collect();
    
    if args.len() >= 3 && args[2] == "--help" {
        print_help();
        return;
    }
    
    // Normal processing
}

fn print_help() {
    println!("cargo-example 
Example cargo plugin

USAGE:
    cargo example [OPTIONS] [ARGS]

OPTIONS:
    --config <PATH>    Configuration file path
    -h, --help         Print help information");
}
```

**Rationale**: The `cargo help` command provides a unified interface for accessing documentation. Users expect `cargo help example` to work just like `cargo help build`. Without proper `--help` support, your plugin appears broken or unprofessional.

**See also**: CG-P-02 (argument handling), CG-P-05 (error messages)

---

## CG-P-04: Use `cargo metadata` for Project Information

**Strength**: SHOULD

**Summary**: Query project structure using `cargo metadata` rather than parsing Cargo.toml directly.

The `cargo metadata` command provides stable, versioned JSON output about project structure, dependencies, and configuration. The `cargo_metadata` crate provides a Rust interface.

```rust
// ❌ BAD: Parsing Cargo.toml manually
use std::fs;
use toml::Value;

fn get_package_name() -> Result<String> {
    let content = fs::read_to_string("Cargo.toml")?;
    let value: Value = toml::from_str(&content)?;
    // Fragile: breaks with workspace inheritance, missing fields, etc.
    Ok(value["package"]["name"]
        .as_str()
        .unwrap()
        .to_string())
}

// ✅ GOOD: Using cargo metadata
use cargo_metadata::{MetadataCommand, CargoOpt};

fn get_package_info() -> Result<()> {
    let metadata = MetadataCommand::new()
        .manifest_path("./Cargo.toml")
        .features(CargoOpt::AllFeatures)
        .exec()?;
    
    for package in metadata.workspace_packages() {
        println!("Package: {} v{}", package.name, package.version);
        println!("Dependencies: {}", package.dependencies.len());
        
        // Access all project information
        for target in &package.targets {
            println!("  Target: {} ({})", target.name, target.kind[0]);
        }
    }
    
    Ok(())
}

// ✅ GOOD: Getting dependency graph
fn analyze_dependencies() -> Result<()> {
    let metadata = MetadataCommand::new().exec()?;
    
    // Resolve dependency graph
    let resolve = metadata.resolve.as_ref().unwrap();
    for node in &resolve.nodes {
        println!("Package: {}", node.id);
        for dep in &node.deps {
            println!("  -> {}", dep.pkg);
        }
    }
    
    Ok(())
}
```

**Rationale**: Cargo.toml syntax has many edge cases: workspace inheritance, optional fields, renamed dependencies, platform-specific dependencies. The `cargo metadata` format is stable, versioned, and handles all these complexities. Manually parsing Cargo.toml is fragile and breaks with new Cargo features. Using the `cargo_metadata` crate provides type safety and automatic deserialization.

**See also**: CG-P-06 (CARGO environment variable), CG-CF-01 (configuration hierarchy)

---

## CG-P-05: Provide Clear Error Messages and Exit Codes

**Strength**: MUST

**Summary**: Use descriptive error messages and appropriate exit codes (0 for success, non-zero for failure).

Cargo plugins should integrate seamlessly with cargo's UX. Clear errors help users understand and fix problems quickly, especially in CI environments.

```rust
// ❌ BAD: Cryptic errors and wrong exit codes
fn main() {
    let result = do_work();
    if result.is_err() {
        println!("Error");  // Unhelpful
        std::process::exit(0);  // Wrong exit code!
    }
}

// ✅ GOOD: Clear errors with context
use anyhow::{Context, Result};

fn main() {
    if let Err(e) = run() {
        eprintln!("Error: {:?}", e);  // Use stderr
        std::process::exit(1);
    }
}

fn run() -> Result<()> {
    let config_path = "config.toml";
    let config = std::fs::read_to_string(config_path)
        .context(format!("Failed to read config file at {}", config_path))?;
    
    let parsed: Config = toml::from_str(&config)
        .context("Failed to parse config file (check TOML syntax)")?;
    
    process_config(parsed)
        .context("Failed to process configuration")?;
    
    Ok(())
}

// ✅ GOOD: User-friendly error formatting
use anyhow::{anyhow, bail, Result};

fn validate_target(target: &str) -> Result<()> {
    if !target.contains("::") {
        bail!(
            "Invalid target format: '{}'\n\
             Expected format: 'crate::module::function'\n\
             Example: cargo example --target myapp::utils::helper",
            target
        );
    }
    Ok(())
}

// ✅ GOOD: Colored output for better UX (optional)
use colored::Colorize;

fn print_success(msg: &str) {
    println!("{} {}", "✓".green(), msg);
}

fn print_error(msg: &str) {
    eprintln!("{} {}", "✗".red(), msg.red());
}
```

**Rationale**: Users invoke cargo plugins in scripts, CI pipelines, and makefiles that rely on exit codes to determine success. Exit code 0 must mean success; non-zero means failure. Writing errors to stderr (not stdout) ensures they appear in logs and don't pollute output parsing. Context-rich errors save debugging time by immediately showing what went wrong and where.

**See also**: CG-P-03 (help messages), CG-A-08 (CI integration)

---

## CG-P-06: Use the `CARGO` Environment Variable

**Strength**: SHOULD

**Summary**: Call back to cargo using the `CARGO` environment variable rather than assuming `cargo` is in PATH.

When cargo invokes your plugin, it sets the `CARGO` environment variable to the path of the cargo binary being used. Use this for invoking cargo commands.

```rust
// ❌ BAD: Assuming cargo is in PATH
use std::process::Command;

fn build_project() -> Result<()> {
    Command::new("cargo")  // Might not be the same cargo!
        .args(&["build", "--release"])
        .status()?;
    Ok(())
}

// ✅ GOOD: Using CARGO environment variable
use std::env;
use std::process::Command;

fn build_project() -> Result<()> {
    let cargo = env::var("CARGO").unwrap_or_else(|_| "cargo".to_string());
    
    Command::new(cargo)
        .args(&["build", "--release"])
        .status()?;
    
    Ok(())
}

// ✅ GOOD: Running cargo metadata
fn get_metadata() -> Result<cargo_metadata::Metadata> {
    let cargo = env::var("CARGO").unwrap_or_else(|_| "cargo".to_string());
    
    cargo_metadata::MetadataCommand::new()
        .cargo_path(cargo)
        .exec()
        .context("Failed to run cargo metadata")
}

// ✅ GOOD: Respecting user's cargo configuration
fn run_with_user_cargo() -> Result<()> {
    let cargo = env::var("CARGO").unwrap_or_else(|_| "cargo".to_string());
    
    // This respects .cargo/config.toml settings
    let output = Command::new(&cargo)
        .args(&["check", "--all-features"])
        .output()?;
    
    if !output.status.success() {
        bail!("cargo check failed");
    }
    
    Ok(())
}
```

**Rationale**: Users may have multiple Rust installations (rustup, system, custom) or use cargo wrappers. The `CARGO` environment variable ensures your plugin uses the same cargo binary that invoked it, maintaining consistency with the user's environment. This is especially important in CI environments or when using cargo through rustup.

**See also**: CG-P-04 (cargo metadata), CG-CF-02 (environment variables)

---

## CG-P-07: Design for Both Direct and Cargo Invocation

**Strength**: CONSIDER

**Summary**: Support both `cargo example` and `cargo-example` invocation patterns.

While cargo integration is primary, supporting direct invocation improves debuggability and flexibility.

```rust
// ✅ GOOD: Detecting invocation method
use clap::Parser;
use std::env;

#[derive(Parser)]
#[command(
    name = "cargo-example",
    bin_name = "cargo"  // For cargo invocation
)]
enum Cli {
    #[command(name = "example")]
    Example(Args),
    
    #[command(external_subcommand)]
    External(Vec<String>),
}

#[derive(Parser)]
struct Args {
    #[arg(long)]
    verbose: bool,
}

fn main() {
    // Detect if invoked directly or via cargo
    let invoked_as_cargo_subcommand = env::args().nth(1)
        .map(|arg| arg == "example")
        .unwrap_or(false);
    
    let cli = if invoked_as_cargo_subcommand {
        Cli::parse()
    } else {
        // Direct invocation: wrap in subcommand
        let args = env::args().collect::<Vec<_>>();
        let mut new_args = vec![args[0].clone(), "example".to_string()];
        new_args.extend_from_slice(&args[1..]);
        Cli::parse_from(new_args)
    };
    
    let Cli::Example(args) = cli else {
        return;
    };
    
    // Process arguments
    if args.verbose {
        println!("Running in verbose mode");
    }
}

// ✅ GOOD: Simple dual invocation support
fn main() {
    let args: Vec<String> = env::args().collect();
    
    // Skip "example" if present (from cargo invocation)
    let args = if args.get(1).map(|s| s.as_str()) == Some("example") {
        &args[2..]
    } else {
        &args[1..]
    };
    
    // Now process args uniformly
    process_args(args);
}
```

**Rationale**: Direct invocation helps during development (running from IDE, debugging with breakpoints) and in edge cases where cargo integration isn't working. However, the primary interface should be cargo integration. Most users will never invoke the plugin directly, so this is optional complexity.

**See also**: CG-P-02 (argument handling), CG-P-03 (help integration)

---

## CG-P-08: Distribute via `cargo install` as Primary Method

**Strength**: SHOULD

**Summary**: Make your plugin installable via `cargo install` by publishing to crates.io.

The `cargo install` mechanism is the standard way Rust users install cargo plugins, providing automatic compilation and PATH management.

```toml
# ✅ GOOD: Cargo.toml configured for cargo install
[package]
name = "cargo-example"
version = "0.1.0"
description = "An example cargo plugin"
license = "MIT OR Apache-2.0"
repository = "https://github.com/user/cargo-example"
keywords = ["cargo", "plugin", "development-tools"]
categories = ["development-tools::cargo-plugins"]

[[bin]]
name = "cargo-example"
path = "src/main.rs"

[dependencies]
clap = { version = "4", features = ["derive"] }
cargo_metadata = "0.18"
anyhow = "1"

# ✅ GOOD: README instructions for installation
```

```markdown
# Installation

Install via cargo:

```bash
cargo install cargo-example
```

This installs the plugin to `~/.cargo/bin/` which should be in your PATH.

## Usage

```bash
cargo example --help
```

## Alternative Installation Methods

### From source:
```bash
cargo install --git https://github.com/user/cargo-example
```

### Using cargo-binstall (faster, downloads pre-built binaries):
```bash
cargo binstall cargo-example
```
```

**Rationale**: Users expect cargo plugins to be installable via `cargo install`, which is the standard Rust tool distribution method. It handles compilation, binary placement in `~/.cargo/bin`, and automatic PATH setup. Publishing to crates.io provides version management, dependency resolution, and discoverability. Alternative methods like `cargo-binstall` are useful optimizations but should supplement, not replace, `cargo install`.

**See also**: CG-PUB-01 (pre-publish checklist), CG-PUB-11 (categories)

---

## CG-P-09: Respect Standard Cargo Flags and Environment

**Strength**: SHOULD

**Summary**: Honor common cargo flags like `--manifest-path`, `--color`, and environment variables.

Consistent behavior with cargo's built-in commands improves user experience and enables integration in complex workflows.

```rust
// ❌ BAD: Ignoring cargo conventions
use cargo_metadata::MetadataCommand;

fn main() {
    // Always uses ./Cargo.toml
    let metadata = MetadataCommand::new().exec().unwrap();
}

// ✅ GOOD: Respecting --manifest-path
use clap::Parser;
use cargo_metadata::MetadataCommand;
use std::path::PathBuf;

#[derive(Parser)]
struct Args {
    /// Path to Cargo.toml
    #[arg(long, value_name = "PATH")]
    manifest_path: Option<PathBuf>,
    
    /// Coloring: auto, always, never
    #[arg(long, value_name = "WHEN")]
    color: Option<String>,
    
    /// Use verbose output
    #[arg(short, long)]
    verbose: bool,
}

fn main() -> Result<()> {
    let args = Args::parse();
    
    // Use specified manifest or discover it
    let mut cmd = MetadataCommand::new();
    if let Some(path) = args.manifest_path {
        cmd.manifest_path(path);
    }
    
    let metadata = cmd.exec()?;
    
    // Respect color settings
    configure_colors(&args.color);
    
    Ok(())
}

// ✅ GOOD: Supporting cargo-standard environment variables
use std::env;

fn should_use_color() -> bool {
    // Check CARGO_TERM_COLOR environment variable
    match env::var("CARGO_TERM_COLOR").as_deref() {
        Ok("always") => true,
        Ok("never") => false,
        Ok("auto") | _ => {
            // Check if output is a terminal
            atty::is(atty::Stream::Stdout)
        }
    }
}

// ✅ GOOD: Verbose output control
fn log(args: &Args, msg: &str) {
    if args.verbose {
        eprintln!("[cargo-example] {}", msg);
    }
}
```

**Rationale**: Users expect cargo plugins to behave like cargo itself. Supporting `--manifest-path` allows running the plugin from any directory. Respecting `--color` and `CARGO_TERM_COLOR` ensures proper integration with CI systems and user preferences. Consistent flag naming reduces cognitive load and enables plugins to be used interchangeably in scripts.

**See also**: CG-CF-02 (environment variables), CG-P-05 (error messages)

---

## CG-P-10: Avoid Linking Against Cargo as a Library

**Strength**: AVOID

**Summary**: Don't link to the `cargo` crate as a library; use cargo as a CLI tool instead.

While tempting, linking against cargo's internal APIs creates maintenance burden and version conflicts.

```rust
// ❌ BAD: Linking against cargo as library
// In Cargo.toml:
[dependencies]
cargo = "0.75"  // DON'T DO THIS

// In code:
use cargo::core::Workspace;
// This API is unstable and changes frequently!

// ✅ GOOD: Using cargo as a CLI tool
use std::process::Command;
use std::env;

fn build_project() -> Result<()> {
    let cargo = env::var("CARGO").unwrap_or_else(|_| "cargo".to_string());
    
    let status = Command::new(cargo)
        .args(&["build", "--release"])
        .status()?;
    
    if !status.success() {
        bail!("Cargo build failed");
    }
    
    Ok(())
}

// ✅ GOOD: Using cargo metadata for information
use cargo_metadata::MetadataCommand;

fn get_project_info() -> Result<()> {
    let metadata = MetadataCommand::new().exec()?;
    // cargo_metadata crate provides stable interface
    
    for package in metadata.workspace_packages() {
        println!("{} v{}", package.name, package.version);
    }
    
    Ok(())
}
```

**Rationale**: Cargo's internal APIs are explicitly unstable and change without notice. Your plugin might work with one cargo version but break with another, frustrating users. Version conflicts arise when your plugin and the user's cargo binary use different cargo library versions. The CLI interface is stable, versioned, and guaranteed to work. Use `cargo_metadata` crate for structured data access—it provides a stable interface to cargo's metadata command.

**See also**: CG-P-04 (cargo metadata), CG-P-06 (CARGO environment variable)

---

## CG-P-11: Provide Workspace-Aware Functionality

**Strength**: CONSIDER

**Summary**: Detect and handle workspaces appropriately, offering per-package or workspace-wide operations.

Many projects use workspaces, and plugins should handle them gracefully.

```rust
// ❌ BAD: Only working with single packages
fn analyze_project() -> Result<()> {
    let metadata = MetadataCommand::new().exec()?;
    
    // Assumes single package, breaks in workspaces
    let package = &metadata.packages[0];
    process_package(package)?;
    
    Ok(())
}

// ✅ GOOD: Workspace-aware operation
use cargo_metadata::{MetadataCommand, Package};
use clap::Parser;

#[derive(Parser)]
struct Args {
    /// Process all workspace members
    #[arg(long)]
    workspace: bool,
    
    /// Process specific package
    #[arg(short, long)]
    package: Option<String>,
}

fn analyze_project(args: &Args) -> Result<()> {
    let metadata = MetadataCommand::new().exec()?;
    
    let packages: Vec<&Package> = if args.workspace {
        // Process all workspace members
        metadata.workspace_packages().collect()
    } else if let Some(pkg_name) = &args.package {
        // Process specific package
        metadata.packages.iter()
            .filter(|p| p.name == *pkg_name)
            .collect()
    } else {
        // Process current package (default)
        vec![metadata.root_package()
            .context("Not in a package directory")?]
    };
    
    for package in packages {
        println!("Analyzing: {}", package.name);
        process_package(package)?;
    }
    
    Ok(())
}

// ✅ GOOD: Detecting workspace structure
fn is_workspace_root(metadata: &cargo_metadata::Metadata) -> bool {
    !metadata.workspace_members.is_empty() && 
    metadata.workspace_root == metadata.target_directory.parent().unwrap()
}

fn print_workspace_info(metadata: &cargo_metadata::Metadata) {
    if is_workspace_root(metadata) {
        println!("Workspace root: {}", metadata.workspace_root.display());
        println!("Members ({}):", metadata.workspace_members.len());
        for member in &metadata.workspace_members {
            println!("  - {}", member);
        }
    } else {
        println!("Single package: {}", 
            metadata.root_package().unwrap().name);
    }
}
```

**Rationale**: Workspaces are common in larger Rust projects. Plugins that only work with single packages frustrate users who work in workspaces. Providing `--workspace` and `--package` flags mirrors cargo's built-in commands, making your plugin feel native. However, if your plugin's functionality doesn't make sense in a workspace context, document this clearly rather than adding complex workspace handling.

**See also**: CG-B-07 (workspace inheritance), CG-P-04 (cargo metadata)

---

## CG-P-12: Document Plugin in README and `--help`

**Strength**: MUST

**Summary**: Provide comprehensive documentation in both README.md and command-line help.

Good documentation is essential for adoption and proper usage of your plugin.

```markdown
# ✅ GOOD: Comprehensive README.md

# cargo-example

A cargo plugin that [brief description of what it does].

## Installation

```bash
cargo install cargo-example
```

## Usage

```bash
# Basic usage
cargo example

# With options
cargo example --config custom.toml --verbose

# In a workspace
cargo example --workspace
```

## Examples

### Example 1: Basic analysis
```bash
cd my-project
cargo example
```

This will analyze your project and output...

### Example 2: Custom configuration
```bash
cargo example --config .example.toml
```

## Configuration

Create a `.example.toml` file in your project root:

```toml
[settings]
threshold = 0.8
ignore_patterns = ["tests/*"]
```

## Command-Line Options

- `--config <PATH>` - Path to configuration file
- `--workspace` - Process all workspace members
- `--package <NAME>` - Process specific package
- `--verbose` - Enable verbose output

## Examples

See the `examples/` directory for complete examples.

## Troubleshooting

### Error: "Not in a Cargo project"
Make sure you run the command from a directory containing Cargo.toml.

### Error: "Failed to parse config"
Check your configuration file syntax...

## Contributing

Contributions welcome! See CONTRIBUTING.md.

## License

MIT OR Apache-2.0
```

```rust
// ✅ GOOD: Comprehensive command-line help
use clap::Parser;

#[derive(Parser)]
#[command(
    name = "cargo-example",
    version,
    about = "Analyze Rust projects for common patterns",
    long_about = "cargo-example is a static analysis tool that scans \
                  your Rust codebase for common patterns, anti-patterns, \
                  and potential issues.
                  
                  It supports both single packages and workspaces, with \
                  configurable rules and output formats.",
    after_help = "EXAMPLES:
    # Analyze current package
    cargo example
    
    # Analyze all workspace members
    cargo example --workspace
    
    # Use custom config
    cargo example --config custom.toml
    
For more information, see: https://github.com/user/cargo-example"
)]
enum Cli {
    #[command(name = "example")]
    Example(Args),
}

#[derive(Parser)]
struct Args {
    /// Path to configuration file
    #[arg(long, value_name = "PATH")]
    config: Option<PathBuf>,
    
    /// Process all workspace members
    #[arg(long)]
    workspace: bool,
    
    /// Process specific package
    #[arg(short, long, value_name = "NAME")]
    package: Option<String>,
    
    /// Enable verbose output
    #[arg(short, long)]
    verbose: bool,
    
    /// Output format: text, json, or markdown
    #[arg(long, value_name = "FORMAT", default_value = "text")]
    format: String,
}
```

**Rationale**: Users discover plugins through search and GitHub. A clear README with installation instructions, usage examples, and troubleshooting increases adoption. Command-line help must be comprehensive because users often check `--help` before reading documentation. Good documentation reduces support burden and improves the user experience. Include examples for common use cases—concrete examples are more useful than abstract descriptions.

**See also**: CG-P-03 (help integration), CG-P-08 (distribution)

---

## Best Practices Summary

| Pattern | Key Takeaway | Strength |
|---------|-------------|----------|
| CG-P-01 | Use `cargo-*` naming for plugin binaries | MUST |
| CG-P-02 | Handle subcommand name in arguments | MUST |
| CG-P-03 | Implement `--help` for cargo integration | MUST |
| CG-P-04 | Use `cargo metadata` for project info | SHOULD |
| CG-P-05 | Provide clear errors and correct exit codes | MUST |
| CG-P-06 | Use `CARGO` environment variable | SHOULD |
| CG-P-07 | Support both cargo and direct invocation | CONSIDER |
| CG-P-08 | Distribute via `cargo install` | SHOULD |
| CG-P-09 | Respect standard cargo flags | SHOULD |
| CG-P-10 | Avoid linking cargo as library | AVOID |
| CG-P-11 | Provide workspace-aware functionality | CONSIDER |
| CG-P-12 | Document comprehensively | MUST |

## Related Guidelines

- **01-cargo-basics.md**: Understanding cargo's package system helps design better plugins
- **05-cargo-configuration.md**: Plugins should respect cargo configuration
- **04-cargo-publishing.md**: Publishing your plugin to crates.io

## External References

- [Cargo External Tools](https://doc.rust-lang.org/cargo/reference/external-tools.html)
- [cargo_metadata crate](https://docs.rs/cargo_metadata/)
- [clap: Command Line Argument Parser](https://docs.rs/clap/)
- [Third-Party Cargo Commands](https://github.com/rust-lang/cargo/wiki/Third-party-cargo-subcommands)
