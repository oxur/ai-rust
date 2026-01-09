# CLI Testing

Patterns for testing command-line applications.

---

## CLI-34: Integration Testing with assert_cmd

**Strength**: SHOULD

**Summary**: Use `assert_cmd` to test your CLI application by running it as a subprocess.

```rust
// tests/cli_tests.rs
use assert_cmd::Command;
use predicates::prelude::*;

// ✅ GOOD: Basic CLI test
#[test]
fn test_help_flag() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("--help")
        .assert()
        .success()
        .stdout(predicate::str::contains("Usage:"))
        .stdout(predicate::str::contains("Options:"));
}

// ✅ GOOD: Test error cases
#[test]
fn test_missing_required_argument() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.assert()
        .failure()
        .stderr(predicate::str::contains("required arguments"))
        .code(2);  // Usage error
}

// ✅ GOOD: Test with arguments
#[test]
fn test_process_file() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("process")
        .arg("test.txt")
        .arg("--output")
        .arg("result.txt")
        .assert()
        .success();
}

// ✅ GOOD: Test stdin input
#[test]
fn test_stdin_input() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.write_stdin("test input\n")
        .assert()
        .success()
        .stdout(predicate::str::contains("test input"));
}

// ✅ GOOD: Test environment variables
#[test]
fn test_env_var_config() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.env("MY_TOOL_TOKEN", "secret")
        .assert()
        .success();
}

// ❌ BAD: Not testing the actual binary
#[test]
fn test_main_function() {
    // This tests the library, not the CLI interface
    let result = my_tool::run(/* args */);
    assert!(result.is_ok());
}
```

**Rationale**: Integration tests verify that your CLI works end-to-end, including argument parsing, error messages, and exit codes. They catch issues that unit tests miss.

**See also**:
- CLI-35: Testing argument parsing
- [External: assert_cmd](https://docs.rs/assert_cmd)

---

## CLI-35: Testing Argument Parsing

**Strength**: MUST

**Summary**: Test argument parsing explicitly, including edge cases and error conditions.

```rust
use clap::Parser;

#[derive(Parser)]
struct Cli {
    #[arg(short, long)]
    port: u16,
    
    #[arg(short, long)]
    verbose: bool,
}

// ✅ GOOD: Unit test for parsing
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_valid_args() {
        let cli = Cli::try_parse_from(["my-tool", "--port", "8080"]).unwrap();
        assert_eq!(cli.port, 8080);
        assert!(!cli.verbose);
    }
    
    #[test]
    fn test_short_flags() {
        let cli = Cli::try_parse_from(["my-tool", "-p", "3000", "-v"]).unwrap();
        assert_eq!(cli.port, 3000);
        assert!(cli.verbose);
    }
    
    #[test]
    fn test_invalid_port() {
        let result = Cli::try_parse_from(["my-tool", "--port", "99999"]);
        assert!(result.is_err());
    }
    
    #[test]
    fn test_missing_required() {
        let result = Cli::try_parse_from(["my-tool"]);
        assert!(result.is_err());
    }
}

// ✅ GOOD: Test with assert_cmd
#[test]
fn test_arg_combinations() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("--port").arg("8080")
        .arg("--verbose")
        .assert()
        .success();
}

#[test]
fn test_invalid_arg_value() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("--port").arg("invalid")
        .assert()
        .failure()
        .stderr(predicate::str::contains("invalid value"));
}

// ✅ GOOD: Test argument conflicts
#[test]
fn test_conflicting_args() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("--json").arg("--yaml")
        .assert()
        .failure()
        .stderr(predicate::str::contains("cannot be used with"));
}
```

**What to test**:
- Valid argument combinations
- Short and long forms of flags
- Default values
- Invalid values and types
- Missing required arguments
- Conflicting arguments
- Argument groups

**Rationale**: Argument parsing is the entry point to your application. Bugs here affect every user immediately.

**See also**:
- CLI-05 through CLI-15: Argument parsing patterns
- CLI-34: Integration testing with assert_cmd

---

## CLI-36: Snapshot Testing for Output

**Strength**: CONSIDER

**Summary**: Use snapshot testing to verify complex output remains consistent across changes.

```rust
// tests/snapshots.rs
use assert_cmd::Command;
use assert_fs::TempDir;

// ✅ GOOD: Snapshot test for help output
#[test]
fn test_help_output() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    let output = cmd.arg("--help")
        .output()
        .unwrap();
    
    let stdout = String::from_utf8_lossy(&output.stdout);
    insta::assert_snapshot!(stdout);
}

// ✅ GOOD: Snapshot test for formatted output
#[test]
fn test_table_output() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    let output = cmd.arg("list")
        .arg("--format").arg("table")
        .output()
        .unwrap();
    
    let stdout = String::from_utf8_lossy(&output.stdout);
    insta::assert_snapshot!(stdout);
}

// ✅ GOOD: Snapshot test for JSON output
#[test]
fn test_json_output() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    let output = cmd.arg("list")
        .arg("--format").arg("json")
        .output()
        .unwrap();
    
    let stdout = String::from_utf8_lossy(&output.stdout);
    // Parse to ensure valid JSON
    let _: serde_json::Value = serde_json::from_str(&stdout).unwrap();
    insta::assert_snapshot!(stdout);
}

// ✅ GOOD: Redact dynamic values
#[test]
fn test_output_with_timestamp() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    let output = cmd.arg("status")
        .output()
        .unwrap();
    
    let stdout = String::from_utf8_lossy(&output.stdout);
    
    // Redact timestamps before snapshotting
    let redacted = regex::Regex::new(r"\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}")
        .unwrap()
        .replace_all(&stdout, "[TIMESTAMP]");
    
    insta::assert_snapshot!(redacted);
}

// ❌ BAD: Testing exact output with hardcoded strings
#[test]
fn test_output_exact() {
    let output = /* run command */;
    assert_eq!(
        output,
        "File: test.txt\nSize: 1234 bytes\nModified: 2024-01-01\n"
    );
    // Brittle! Breaks with any formatting change
}
```

**When to use snapshots**:
- Help text and documentation
- Formatted tables
- Complex structured output
- Error messages

**When not to use snapshots**:
- Output with dynamic values (timestamps, IDs)
- Binary output
- Very large output

**Rationale**: Snapshot tests catch unintended output changes and make it easy to review output modifications. They're especially useful for help text and formatted output.

**See also**:
- [External: insta](https://docs.rs/insta) - Snapshot testing
- CLI-34: Integration testing with assert_cmd

---

## CLI-37: Testing Error Conditions

**Strength**: SHOULD

**Summary**: Explicitly test error cases, error messages, and exit codes.

```rust
use assert_cmd::Command;
use predicates::prelude::*;
use assert_fs::TempDir;

// ✅ GOOD: Test file not found
#[test]
fn test_file_not_found() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("process").arg("nonexistent.txt")
        .assert()
        .failure()
        .code(66)  // EX_NOINPUT
        .stderr(predicate::str::contains("File not found"))
        .stderr(predicate::str::contains("nonexistent.txt"));
}

// ✅ GOOD: Test invalid input
#[test]
fn test_invalid_port() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("--port").arg("99999")
        .assert()
        .failure()
        .code(2)  // Usage error
        .stderr(predicate::str::contains("invalid value"))
        .stderr(predicate::str::contains("99999"));
}

// ✅ GOOD: Test permission denied
#[test]
fn test_permission_denied() {
    let temp = TempDir::new().unwrap();
    let file = temp.child("readonly.txt");
    file.write_str("content").unwrap();
    
    // Make file read-only
    use std::fs::Permissions;
    use std::os::unix::fs::PermissionsExt;
    std::fs::set_permissions(
        file.path(),
        Permissions::from_mode(0o444)
    ).unwrap();
    
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("write").arg(file.path())
        .assert()
        .failure()
        .stderr(predicate::str::contains("Permission denied"));
}

// ✅ GOOD: Test invalid config
#[test]
fn test_invalid_config() {
    let temp = TempDir::new().unwrap();
    let config = temp.child("config.toml");
    config.write_str("invalid toml content {").unwrap();
    
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("--config").arg(config.path())
        .assert()
        .failure()
        .stderr(predicate::str::contains("Failed to parse config"));
}

// ✅ GOOD: Test error message quality
#[test]
fn test_helpful_error_message() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("--format").arg("invalid")
        .assert()
        .failure()
        .stderr(predicate::str::contains("invalid format"))
        .stderr(predicate::str::contains("Supported formats:"))
        .stderr(predicate::str::contains("json"))
        .stderr(predicate::str::contains("yaml"));
}

// ❌ BAD: Only testing success cases
#[test]
fn test_process_file() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    cmd.arg("test.txt").assert().success();
    // What about missing files? Invalid input? Permission errors?
}
```

**Error conditions to test**:
- File not found
- Permission denied
- Invalid input format
- Missing required arguments
- Invalid argument values
- Configuration errors
- Network failures (if applicable)

**Rationale**: Error paths are often under-tested but critical for user experience. Testing ensures errors provide helpful messages.

**See also**:
- CLI-16: Exit codes
- CLI-17: User-friendly error messages
- CLI-19: Distinguish user errors from system errors

---

## CLI-38: Testing with Temporary Files

**Strength**: CONSIDER

**Summary**: Use `assert_fs` to create temporary test fixtures that are cleaned up automatically.

```rust
use assert_cmd::Command;
use assert_fs::prelude::*;
use assert_fs::TempDir;
use predicates::prelude::*;

// ✅ GOOD: Create temporary input files
#[test]
fn test_process_input_file() {
    let temp = TempDir::new().unwrap();
    let input_file = temp.child("input.txt");
    input_file.write_str("test content").unwrap();
    
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("process").arg(input_file.path())
        .assert()
        .success();
    
    // Temp dir automatically cleaned up when dropped
}

// ✅ GOOD: Verify output files
#[test]
fn test_output_file_creation() {
    let temp = TempDir::new().unwrap();
    let input_file = temp.child("input.txt");
    input_file.write_str("test content").unwrap();
    
    let output_file = temp.child("output.txt");
    
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("convert")
        .arg(input_file.path())
        .arg("--output").arg(output_file.path())
        .assert()
        .success();
    
    output_file.assert(predicate::path::exists());
    output_file.assert(predicate::str::contains("converted"));
}

// ✅ GOOD: Test with directory structure
#[test]
fn test_process_directory() {
    let temp = TempDir::new().unwrap();
    
    // Create directory structure
    temp.child("subdir").create_dir_all().unwrap();
    temp.child("subdir/file1.txt").write_str("content 1").unwrap();
    temp.child("subdir/file2.txt").write_str("content 2").unwrap();
    
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("process-dir").arg(temp.path().join("subdir"))
        .assert()
        .success()
        .stdout(predicate::str::contains("Processed 2 files"));
}

// ✅ GOOD: Test with JSON fixtures
#[test]
fn test_json_input() {
    let temp = TempDir::new().unwrap();
    let input = temp.child("data.json");
    
    let data = serde_json::json!({
        "name": "test",
        "value": 42
    });
    input.write_str(&data.to_string()).unwrap();
    
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    cmd.arg("process").arg(input.path())
        .assert()
        .success();
}

// ✅ GOOD: Test file overwrites
#[test]
fn test_overwrite_existing_file() {
    let temp = TempDir::new().unwrap();
    let file = temp.child("output.txt");
    file.write_str("old content").unwrap();
    
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    // Should fail without --force
    cmd.arg("write").arg(file.path())
        .assert()
        .failure();
    
    // Should succeed with --force
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    cmd.arg("write").arg(file.path()).arg("--force")
        .assert()
        .success();
    
    file.assert(predicate::str::contains("new content"));
}

// ❌ BAD: Using real files in tests
#[test]
fn test_with_real_files() {
    let mut cmd = Command::cargo_bin("my-tool").unwrap();
    
    // Create file in project directory
    std::fs::write("test_input.txt", "content").unwrap();
    
    cmd.arg("test_input.txt").assert().success();
    
    // Forgot to clean up! File persists after test
}
```

**Rationale**: Temporary files ensure tests are isolated and don't leave artifacts. `assert_fs` provides a clean API and automatic cleanup.

**See also**:
- CLI-34: Integration testing with assert_cmd
- [External: assert_fs](https://docs.rs/assert_fs)

---

## Best Practices Summary

| Pattern | Strength | Key Takeaway |
|---------|----------|--------------|
| CLI-34 | SHOULD | Use assert_cmd for end-to-end CLI testing |
| CLI-35 | MUST | Explicitly test argument parsing and edge cases |
| CLI-36 | CONSIDER | Snapshot tests for complex output verification |
| CLI-37 | SHOULD | Test error conditions and error messages |
| CLI-38 | CONSIDER | Use assert_fs for temporary test fixtures |

## Related Guidelines

- CLI-16: Exit codes (in 03-error-handling.md)
- CLI-17: User-friendly error messages (in 03-error-handling.md)
- [External: Rust testing best practices](https://doc.rust-lang.org/book/ch11-00-testing.html)

## External References

- [assert_cmd](https://docs.rs/assert_cmd) - CLI testing
- [assert_fs](https://docs.rs/assert_fs) - Filesystem fixtures
- [predicates](https://docs.rs/predicates) - Assertion predicates
- [insta](https://docs.rs/insta) - Snapshot testing
