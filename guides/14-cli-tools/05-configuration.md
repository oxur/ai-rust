# CLI Configuration

Patterns for managing configuration in command-line applications.

---

## CLI-29: Configuration File Formats

**Strength**: CONSIDER

**Summary**: Choose a configuration format based on your use case: TOML for general config, JSON for interop, YAML for complex structures.

```rust
use serde::{Deserialize, Serialize};
use std::path::Path;

#[derive(Debug, Deserialize, Serialize)]
struct Config {
    server: ServerConfig,
    database: DatabaseConfig,
    logging: LoggingConfig,
}

#[derive(Debug, Deserialize, Serialize)]
struct ServerConfig {
    host: String,
    port: u16,
    workers: usize,
}

// ✅ GOOD: TOML (recommended for human editing)
fn load_toml_config(path: &Path) -> Result<Config> {
    let contents = std::fs::read_to_string(path)?;
    let config: Config = toml::from_str(&contents)?;
    Ok(config)
}

// config.toml
/*
[server]
host = "0.0.0.0"
port = 8080
workers = 4

[database]
url = "postgresql://localhost/mydb"
pool_size = 10

[logging]
level = "info"
format = "json"
*/

// ✅ GOOD: JSON (for programmatic generation)
fn load_json_config(path: &Path) -> Result<Config> {
    let contents = std::fs::read_to_string(path)?;
    let config: Config = serde_json::from_str(&contents)?;
    Ok(config)
}

// ✅ GOOD: YAML (for complex nested configs)
fn load_yaml_config(path: &Path) -> Result<Config> {
    let contents = std::fs::read_to_string(path)?;
    let config: Config = serde_yaml::from_str(&contents)?;
    Ok(config)
}

// ✅ GOOD: Support multiple formats
fn load_config(path: &Path) -> Result<Config> {
    let contents = std::fs::read_to_string(path)?;
    
    match path.extension().and_then(|s| s.to_str()) {
        Some("toml") => Ok(toml::from_str(&contents)?),
        Some("json") => Ok(serde_json::from_str(&contents)?),
        Some("yaml") | Some("yml") => Ok(serde_yaml::from_str(&contents)?),
        _ => Err(anyhow::anyhow!("Unsupported config format")),
    }
}

// ❌ BAD: Custom config format
fn parse_custom_config(contents: &str) -> Config {
    // Don't reinvent the wheel!
    // Use established formats with good tools
}
```

**Format comparison**:
- **TOML**: Human-friendly, good for app config, limited nesting
- **JSON**: Universal, good for interop, no comments
- **YAML**: Complex structures, supports comments, subtle gotchas
- **RON**: Rust-specific, good type system integration

**Rationale**: Use established formats that have good tooling and are familiar to users. TOML is the de facto standard for Rust tools (like Cargo.toml).

**See also**:
- CLI-30: Configuration file locations
- [External: TOML spec](https://toml.io/)

---

## CLI-30: Configuration File Locations

**Strength**: SHOULD

**Summary**: Follow platform conventions for config file locations using the directories crate.

```rust
use directories::ProjectDirs;
use std::path::PathBuf;

#[derive(Parser)]
struct Cli {
    /// Config file path (overrides default locations)
    #[arg(short, long)]
    config: Option<PathBuf>,
}

// ✅ GOOD: Platform-specific config directories
fn get_config_path(cli: &Cli, filename: &str) -> Option<PathBuf> {
    // 1. Explicit path from CLI
    if let Some(path) = &cli.config {
        return Some(path.clone());
    }
    
    // 2. Environment variable
    if let Ok(path) = std::env::var("MY_TOOL_CONFIG") {
        return Some(PathBuf::from(path));
    }
    
    // 3. Current directory
    let local_config = PathBuf::from(filename);
    if local_config.exists() {
        return Some(local_config);
    }
    
    // 4. Platform-specific user config directory
    if let Some(proj_dirs) = ProjectDirs::from("com", "MyOrg", "MyTool") {
        let config_dir = proj_dirs.config_dir();
        let config_path = config_dir.join(filename);
        if config_path.exists() {
            return Some(config_path);
        }
    }
    
    None
}

// ✅ GOOD: Create default config if missing
fn ensure_config(filename: &str) -> Result<PathBuf> {
    let proj_dirs = ProjectDirs::from("com", "MyOrg", "MyTool")
        .ok_or_else(|| anyhow::anyhow!("Could not determine config directory"))?;
    
    let config_dir = proj_dirs.config_dir();
    std::fs::create_dir_all(config_dir)?;
    
    let config_path = config_dir.join(filename);
    
    if !config_path.exists() {
        let default_config = Config::default();
        let toml = toml::to_string_pretty(&default_config)?;
        std::fs::write(&config_path, toml)?;
    }
    
    Ok(config_path)
}

// ✅ GOOD: Check multiple locations in order
fn load_config() -> Result<Option<Config>> {
    let locations = vec![
        // Current directory (highest priority)
        PathBuf::from("my-tool.toml"),
        
        // User config directory
        ProjectDirs::from("com", "MyOrg", "MyTool")
            .map(|d| d.config_dir().join("my-tool.toml")),
        
        // System-wide config (lowest priority)
        Some(PathBuf::from("/etc/my-tool/config.toml")),
    ];
    
    for location in locations.into_iter().flatten() {
        if location.exists() {
            let config = load_config_from_file(&location)?;
            return Ok(Some(config));
        }
    }
    
    Ok(None)
}

// ❌ BAD: Hardcoded paths
fn get_config_path() -> PathBuf {
    PathBuf::from("/home/user/.config/my-tool/config.toml")
    // Won't work on Windows, other users, or different setups
}

// ❌ BAD: Only one location
fn load_config() -> Result<Config> {
    load_config_from_file("./config.toml")
    // Must be in current directory - inflexible
}
```

**Standard locations** (via `directories` crate):
- **Linux**: `~/.config/my-tool/config.toml`
- **macOS**: `~/Library/Application Support/com.MyOrg.MyTool/config.toml`
- **Windows**: `C:\Users\Name\AppData\Roaming\MyOrg\MyTool\config\config.toml`

**Search order** (highest to lowest priority):
1. CLI argument: `--config path/to/config.toml`
2. Environment variable: `MY_TOOL_CONFIG`
3. Current directory: `./my-tool.toml`
4. User config dir: `~/.config/my-tool/config.toml`
5. System config: `/etc/my-tool/config.toml`

**Rationale**: Following platform conventions makes your tool feel native. Users know where to find config files. The `directories` crate handles platform differences.

**See also**:
- CLI-31: Configuration precedence
- [External: directories crate](https://docs.rs/directories)
- [External: XDG Base Directory](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)

---

## CLI-31: Configuration Precedence (Defaults → File → Env → Args)

**Strength**: MUST

**Summary**: Apply configuration in order: built-in defaults, config file, environment variables, command-line arguments.

```rust
use clap::Parser;
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize)]
struct Config {
    host: String,
    port: u16,
    verbose: bool,
}

impl Default for Config {
    fn default() -> Self {
        Self {
            host: "localhost".to_string(),
            port: 8080,
            verbose: false,
        }
    }
}

#[derive(Parser)]
struct Cli {
    /// Config file path
    #[arg(short, long)]
    config: Option<PathBuf>,
    
    /// Server host
    #[arg(long, env = "MY_TOOL_HOST")]
    host: Option<String>,
    
    /// Server port
    #[arg(long, env = "MY_TOOL_PORT")]
    port: Option<u16>,
    
    /// Verbose output
    #[arg(short, long, env = "MY_TOOL_VERBOSE")]
    verbose: bool,
}

// ✅ GOOD: Proper precedence (defaults → file → env → args)
fn build_config(cli: Cli) -> Result<Config> {
    // 1. Start with defaults
    let mut config = Config::default();
    
    // 2. Load from config file if specified
    if let Some(config_path) = cli.config {
        let file_config: Config = load_config_file(&config_path)?;
        config.host = file_config.host;
        config.port = file_config.port;
        config.verbose = file_config.verbose;
    }
    
    // 3. Override with CLI args (which include env vars)
    if let Some(host) = cli.host {
        config.host = host;
    }
    if let Some(port) = cli.port {
        config.port = port;
    }
    if cli.verbose {
        config.verbose = true;
    }
    
    Ok(config)
}

// ✅ GOOD: Using merge pattern
impl Config {
    pub fn merge(self, other: Config) -> Config {
        Config {
            host: if other.host != Config::default().host {
                other.host
            } else {
                self.host
            },
            port: if other.port != Config::default().port {
                other.port
            } else {
                self.port
            },
            verbose: self.verbose || other.verbose,
        }
    }
}

fn build_config(cli: Cli) -> Result<Config> {
    let defaults = Config::default();
    
    let file_config = cli.config
        .as_ref()
        .map(|p| load_config_file(p))
        .transpose()?
        .unwrap_or_default();
    
    let cli_config = Config {
        host: cli.host.unwrap_or_default(),
        port: cli.port.unwrap_or_default(),
        verbose: cli.verbose,
    };
    
    Ok(defaults.merge(file_config).merge(cli_config))
}

// ✅ GOOD: Document precedence
/// Configuration is loaded in the following order (later overrides earlier):
/// 1. Built-in defaults
/// 2. Config file (if specified with --config or found in standard locations)
/// 3. Environment variables (MY_TOOL_*)
/// 4. Command-line arguments
fn load_config(cli: Cli) -> Result<Config> {
    // Implementation
}

// ❌ BAD: Wrong precedence
fn build_config(cli: Cli) -> Result<Config> {
    let mut config = load_config_file("config.toml")?;
    
    // Wrong! CLI args should override file, not the reverse
    if config.host.is_empty() {
        config.host = cli.host.unwrap_or_default();
    }
    
    Ok(config)
}
```

**Precedence order** (last wins):
1. **Defaults**: Sensible built-in defaults
2. **Config file**: Persistent user preferences
3. **Environment variables**: Deployment-specific settings
4. **Command-line arguments**: Immediate overrides

**Rationale**: This precedence order matches user expectations: defaults are always available, config files are semi-permanent, env vars are deployment-specific, and CLI args are immediate overrides.

**See also**:
- CLI-12: Environment variable integration
- CLI-30: Configuration file locations

---

## CLI-32: Config Validation and Defaults

**Strength**: SHOULD

**Summary**: Validate configuration after loading and provide sensible defaults for all settings.

```rust
use anyhow::{Context, Result};
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize, Serialize)]
struct Config {
    #[serde(default = "default_host")]
    host: String,
    
    #[serde(default = "default_port")]
    port: u16,
    
    #[serde(default)]
    timeout_secs: u64,
    
    #[serde(default = "default_workers")]
    workers: usize,
}

fn default_host() -> String {
    "localhost".to_string()
}

fn default_port() -> u16 {
    8080
}

fn default_workers() -> usize {
    num_cpus::get()
}

impl Config {
    // ✅ GOOD: Validation method
    pub fn validate(&self) -> Result<()> {
        if self.host.is_empty() {
            anyhow::bail!("host cannot be empty");
        }
        
        if self.port == 0 {
            anyhow::bail!("port must be between 1 and 65535");
        }
        
        if self.workers == 0 {
            anyhow::bail!("workers must be at least 1");
        }
        
        if self.timeout_secs > 3600 {
            anyhow::bail!("timeout cannot exceed 1 hour");
        }
        
        Ok(())
    }
    
    // ✅ GOOD: Apply defaults for missing values
    pub fn apply_defaults(mut self) -> Self {
        if self.host.is_empty() {
            self.host = default_host();
        }
        
        if self.port == 0 {
            self.port = default_port();
        }
        
        if self.workers == 0 {
            self.workers = default_workers();
        }
        
        self
    }
}

// ✅ GOOD: Load and validate
fn load_config(path: &Path) -> Result<Config> {
    let contents = std::fs::read_to_string(path)
        .with_context(|| format!("Failed to read config file: {}", path.display()))?;
    
    let config: Config = toml::from_str(&contents)
        .with_context(|| format!("Failed to parse config file: {}", path.display()))?;
    
    config.validate()
        .with_context(|| format!("Invalid config in: {}", path.display()))?;
    
    Ok(config)
}

// ✅ GOOD: Helpful validation errors
impl Config {
    pub fn validate(&self) -> Result<()> {
        if self.port < 1024 {
            anyhow::bail!(
                "Port {} requires root privileges. Use a port >= 1024",
                self.port
            );
        }
        
        if self.workers > 128 {
            anyhow::bail!(
                "Workers set to {}, which is very high. \
                 This will consume significant resources. \
                 Recommended: 1-{} (number of CPU cores)",
                self.workers,
                num_cpus::get()
            );
        }
        
        Ok(())
    }
}

// ❌ BAD: No validation
fn load_config(path: &Path) -> Result<Config> {
    let config: Config = toml::from_str(&std::fs::read_to_string(path)?)?;
    Ok(config)  // Could have port=0, empty host, etc.
}

// ❌ BAD: No defaults
#[derive(Deserialize)]
struct Config {
    host: String,
    port: u16,
    // User must specify every field - annoying!
}
```

**Validation guidelines**:
- Validate ranges: ports (1-65535), percentages (0-100)
- Check for empty strings, zero values where invalid
- Validate paths exist if required
- Validate formats (URLs, emails)
- Provide helpful error messages with suggestions

**Rationale**: Validation catches configuration errors early with clear messages. Defaults make config files smaller and easier to write.

**See also**:
- CLI-17: User-friendly error messages
- CLI-31: Configuration precedence

---

## CLI-33: XDG Base Directory Specification

**Strength**: CONSIDER

**Summary**: On Linux, follow XDG Base Directory spec for config, data, and cache locations.

```rust
use directories::ProjectDirs;
use std::path::PathBuf;

// ✅ GOOD: Using XDG directories
fn get_project_dirs() -> Option<ProjectDirs> {
    ProjectDirs::from("com", "MyOrg", "MyTool")
}

fn get_config_dir() -> Result<PathBuf> {
    let proj_dirs = get_project_dirs()
        .ok_or_else(|| anyhow::anyhow!("Could not determine config directory"))?;
    
    let config_dir = proj_dirs.config_dir();
    std::fs::create_dir_all(config_dir)?;
    
    Ok(config_dir.to_path_buf())
}

fn get_data_dir() -> Result<PathBuf> {
    let proj_dirs = get_project_dirs()
        .ok_or_else(|| anyhow::anyhow!("Could not determine data directory"))?;
    
    let data_dir = proj_dirs.data_dir();
    std::fs::create_dir_all(data_dir)?;
    
    Ok(data_dir.to_path_buf())
}

fn get_cache_dir() -> Result<PathBuf> {
    let proj_dirs = get_project_dirs()
        .ok_or_else(|| anyhow::anyhow!("Could not determine cache directory"))?;
    
    let cache_dir = proj_dirs.cache_dir();
    std::fs::create_dir_all(cache_dir)?;
    
    Ok(cache_dir.to_path_buf())
}

// ✅ GOOD: Respect XDG environment variables
fn get_config_home() -> PathBuf {
    std::env::var("XDG_CONFIG_HOME")
        .map(PathBuf::from)
        .unwrap_or_else(|_| {
            let home = std::env::var("HOME").expect("HOME not set");
            PathBuf::from(home).join(".config")
        })
}

// ✅ GOOD: Clean up cache on command
fn clear_cache() -> Result<()> {
    let cache_dir = get_cache_dir()?;
    
    if cache_dir.exists() {
        std::fs::remove_dir_all(&cache_dir)?;
        println!("Cache cleared: {}", cache_dir.display());
    }
    
    Ok(())
}

// ✅ GOOD: Organize by purpose
// Config: ~/.config/my-tool/
//   config.toml      - User settings
// Data: ~/.local/share/my-tool/
//   database.db      - Persistent app data
//   plugins/         - User-installed plugins
// Cache: ~/.cache/my-tool/
//   downloads/       - Temporary downloads
//   index.json       - Cached index

// ❌ BAD: Everything in one directory
fn get_app_dir() -> PathBuf {
    let home = std::env::var("HOME").unwrap();
    PathBuf::from(home).join(".my-tool")
    // Config, data, cache all mixed together
}
```

**XDG directories** (Linux):
- `XDG_CONFIG_HOME` (default: `~/.config`) - Configuration files
- `XDG_DATA_HOME` (default: `~/.local/share`) - Application data
- `XDG_CACHE_HOME` (default: `~/.cache`) - Cached data

**Platform equivalents**:
- **macOS**: Uses `~/Library/` hierarchy
- **Windows**: Uses `AppData/Roaming`, `AppData/Local`

**What goes where**:
- **Config**: Settings, preferences (should be backed up)
- **Data**: Databases, plugins, important files (should be backed up)
- **Cache**: Temporary data, can be deleted safely

**Rationale**: Following XDG spec makes your tool integrate well with Linux systems and backup tools. The `directories` crate handles platform differences.

**See also**:
- CLI-30: Configuration file locations
- [External: XDG Base Directory](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)
- [External: directories crate](https://docs.rs/directories)

---

## Best Practices Summary

| Pattern | Strength | Key Takeaway |
|---------|----------|--------------|
| CLI-29 | CONSIDER | TOML for config, JSON for interop, YAML for complexity |
| CLI-30 | SHOULD | Use platform-specific locations via directories crate |
| CLI-31 | MUST | Precedence: defaults → file → env → args |
| CLI-32 | SHOULD | Validate config and provide sensible defaults |
| CLI-33 | CONSIDER | Follow XDG Base Directory spec on Linux |

## Related Guidelines

- CLI-12: Environment variable integration (in 02-argument-parsing.md)
- CLI-17: User-friendly error messages (in 03-error-handling.md)

## External References

- [TOML specification](https://toml.io/)
- [directories crate](https://docs.rs/directories) - Platform directories
- [config crate](https://docs.rs/config) - Configuration library
- [XDG Base Directory](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)
