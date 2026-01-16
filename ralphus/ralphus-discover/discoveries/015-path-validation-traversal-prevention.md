# Discovery: How does path validation prevent traversal attacks?

> Category: Follow-ups (Security)
> Discovered: 2026-01-16
> Confidence: High

## Answer

Loom implements a **multi-layered defense-in-depth strategy** for path validation and traversal attack prevention. The core pattern is **"Canonicalize-then-Prefix-Check"**: every path is resolved to its canonical form (resolving symlinks and `..` components), then verified to start with the canonicalized workspace root. This pattern is consistently applied across CLI tools, server-side query validation, and remote execution (Weaver) environments.

The system prevents several attack vectors:
1. **Directory traversal** (`../../../etc/passwd`) - blocked by canonicalization + prefix check
2. **Symlink escapes** - blocked by canonicalization resolving symlinks before checking
3. **Null byte injection** (`file\0.txt`) - explicitly rejected
4. **Absolute path injection** (`/etc/passwd`) - rejected at input validation
5. **Sensitive system paths** - blocked by explicit blocklist (`/etc`, `/root`, `/sys`)

For remote execution (Weaver), an additional layer uses **EBPF-based syscall monitoring** to detect sandbox escape attempts at the kernel level.

## Evidence

### Core Validation Pattern (CLI Tools)

Each filesystem tool implements a private `validate_path` method with identical logic:

**`crates/loom-cli-tools/src/read_file.rs` (lines 33-53)**
```rust
fn validate_path(path: &PathBuf, workspace_root: &Path) -> Result<PathBuf, ToolError> {
    // Step 1: Convert to absolute path
    let absolute_path = if path.is_absolute() {
        path.clone()
    } else {
        workspace_root.join(path)
    };

    // Step 2: Canonicalize (resolves symlinks and ..)
    let canonical = absolute_path
        .canonicalize()
        .map_err(|_| ToolError::FileNotFound(absolute_path.clone()))?;

    // Step 3: Canonicalize workspace root
    let workspace_canonical = workspace_root
        .canonicalize()
        .map_err(|_| ToolError::FileNotFound(workspace_root.to_path_buf()))?;

    // Step 4: Prefix check - must start with workspace
    if !canonical.starts_with(&workspace_canonical) {
        return Err(ToolError::PathOutsideWorkspace(canonical));
    }

    Ok(canonical)
}
```

### Bash Tool Working Directory Validation

**`crates/loom-cli-tools/src/bash.rs` (lines 41-64)**
```rust
fn validate_cwd(cwd: &Path, workspace_root: &Path) -> Result<PathBuf, ToolError> {
    let absolute_path = if cwd.is_absolute() {
        cwd.to_path_buf()
    } else {
        workspace_root.join(cwd)
    };

    let canonical = absolute_path.canonicalize().map_err(|_| {
        ToolError::InvalidArguments(format!(
            "working directory does not exist: {}",
            absolute_path.display()
        ))
    })?;

    let workspace_canonical = workspace_root
        .canonicalize()
        .map_err(|_| ToolError::FileNotFound(workspace_root.to_path_buf()))?;

    if !canonical.starts_with(&workspace_canonical) {
        return Err(ToolError::PathOutsideWorkspace(canonical));
    }

    Ok(canonical)
}
```

### Server-Side Query Security

**`crates/loom-server/src/query_security.rs`**

Two complementary validators:

1. **QueryValidator** - High-level checks (lines 139-156):
```rust
pub fn validate_path(&self, path: &str) -> Result<(), SecurityError> {
    // Check for blocked paths (/etc, /root, /sys)
    for blocked in &self.blocked_paths {
        if path.starts_with(blocked) {
            return Err(SecurityError::BlockedPath(path.to_string()));
        }
    }

    // Check for path escape attempts
    if path.contains("..") {
        return Err(SecurityError::PathEscapeAttempt(path.to_string()));
    }

    Ok(())
}
```

2. **PathSanitizer** - Full canonicalization (lines 192-234):
```rust
pub fn sanitize(&self, path: &str) -> Result<PathBuf, SecurityError> {
    // Reject null bytes (prevents null byte injection)
    if path.contains('\0') {
        return Err(SecurityError::InvalidPath(path.to_string()));
    }

    // Reject absolute paths (must be relative to workspace)
    let path_buf = PathBuf::from(path);
    if path_buf.is_absolute() {
        return Err(SecurityError::InvalidPath(path.to_string()));
    }

    // Join with workspace and canonicalize
    let joined = self.workspace_root.join(&path_buf);
    let canonical = joined.canonicalize().map_err(|_| {
        SecurityError::InvalidPath(path.to_string())
    })?;

    // Verify within workspace
    if !canonical.starts_with(&self.workspace_root) {
        return Err(SecurityError::PathEscapeAttempt(path.to_string()));
    }

    Ok(canonical)
}
```

### Repository Name Validation (SCM)

**`crates/loom-server-scm/src/repo.rs` (lines 11-42)**
```rust
pub fn validate_repo_name(name: &str) -> Result<()> {
    if name.is_empty() || name.len() > 100 {
        return Err(ScmError::InvalidName("Name must be 1-100 characters".into()));
    }

    if name == "." || name == ".." {
        return Err(ScmError::InvalidName("Invalid name".into()));
    }

    if name.starts_with('.') || name.starts_with('-') {
        return Err(ScmError::InvalidName("Name cannot start with '.' or '-'".into()));
    }

    if name.contains("..") {
        return Err(ScmError::InvalidName("Name cannot contain '..'".into()));
    }

    // Whitelist: only alphanumeric, dash, underscore, dot
    if !name.chars().all(|c| c.is_ascii_alphanumeric() || c == '-' || c == '_' || c == '.') {
        return Err(ScmError::InvalidName(
            "Name can only contain letters, numbers, dash, underscore, dot".into(),
        ));
    }

    Ok(())
}
```

### Weaver Audit Sidecar (EBPF Monitoring)

**`crates/loom-weaver-audit-sidecar/src/filter.rs` (lines 4-17)**
```rust
const SENSITIVE_PATH_PREFIXES: &[&str] = &[
    "/etc/passwd",
    "/etc/shadow",
    "/etc/sudoers",
    "/etc/ssh",
    "/root",
    "/home",
    "/.ssh",
    "/.gnupg",
    "/.aws",
    "/.config",
    "/proc/",
    "/sys/",
];
```

**Key Files:**
- `crates/loom-cli-tools/src/read_file.rs` — File reading path validation
- `crates/loom-cli-tools/src/edit_file.rs` — File editing path validation (handles non-existent files)
- `crates/loom-cli-tools/src/list_files.rs` — Directory listing path validation
- `crates/loom-cli-tools/src/bash.rs` — Shell command working directory validation
- `crates/loom-server/src/query_security.rs` — Server-side PathSanitizer and QueryValidator
- `crates/loom-server/tests/query_security_tests.rs` — Comprehensive security tests
- `crates/loom-server-scm/src/repo.rs` — Repository name validation (prevents traversal in names)
- `crates/loom-weaver-audit-sidecar/src/filter.rs` — Sensitive path detection for audit
- `crates/loom-common-core/src/error.rs` — ToolError::PathOutsideWorkspace definition

## Implications

- **Workspace is the security boundary**: All file operations are sandboxed to the workspace root. The `ToolContext.workspace_root` is the source of truth.
- **Canonicalization is mandatory**: Never trust user-provided paths without canonicalizing first. This resolves symlinks and `..` components.
- **Prefix check after canonicalization**: The order matters - canonicalize THEN check prefix. Checking before canonicalization is bypassable.
- **Edit tool handles new files**: The edit tool has special logic for files that don't exist yet (validates parent directory instead).
- **Property-based testing**: Security-critical validation uses `proptest` to ensure edge cases are covered (see `query_security_tests.rs`).
- **Defense in depth**: Multiple layers (input validation, canonicalization, blocklists, EBPF monitoring) ensure a single bypass doesn't compromise security.
- **Error types are specific**: `ToolError::PathOutsideWorkspace` and `SecurityError::PathEscapeAttempt` provide clear error messages for debugging.

## Follow-up Questions

- [ ] How does the bash tool handle timeouts and output truncation?
- [ ] How does the EBPF sandbox escape detection work in detail? (syscall monitoring, event types)

## Related Discoveries

- [[004-tool-execution-system]] — Tool execution lifecycle and ToolContext
- [[014-thread-system-persistence]] — Workspace root in thread context
