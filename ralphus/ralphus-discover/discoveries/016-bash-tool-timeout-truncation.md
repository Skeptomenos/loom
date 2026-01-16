# Discovery: How does the bash tool handle timeouts and output truncation?

> Category: Follow-ups (Tools)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The bash tool implements a **robust timeout and output truncation system** using Tokio's async primitives. Commands are executed via `sh -c` with configurable timeouts (default 60s, max 300s) enforced by `tokio::time::timeout`. Output is captured fully from the process, then truncated to 256KB per stream (stdout/stderr) before being returned to the agent.

**Key design decisions:**
1. **Timeout wraps the entire command execution** - not just output reading
2. **No explicit process termination** - when timeout fires, the Tokio future is dropped, but the child process may continue running (orphaned)
3. **Truncation happens post-execution** - the full output is captured in memory, then sliced
4. **UTF-8 safety via lossy conversion** - `String::from_utf8_lossy` ensures truncation mid-character doesn't cause errors
5. **Explicit status flags** - `timed_out` and `truncated` booleans in the result inform the agent of what happened

## Evidence

### Timeout Implementation

**`crates/loom-cli-tools/src/bash.rs` (lines 14-16, 124-174)**

```rust
const DEFAULT_TIMEOUT_SECS: u64 = 60;
const MAX_TIMEOUT_SECS: u64 = 300;
const MAX_OUTPUT_BYTES: usize = 256 * 1024; // 256KB per stream

// Inside invoke():
let timeout_secs = args
    .timeout_secs
    .unwrap_or(DEFAULT_TIMEOUT_SECS)
    .min(MAX_TIMEOUT_SECS);

let mut cmd = Command::new("sh");
cmd.arg("-c").arg(&args.command).current_dir(&working_dir);

let result = timeout(Duration::from_secs(timeout_secs), cmd.output()).await;

let (exit_code, stdout, stderr, timed_out, truncated) = match result {
    Ok(Ok(output)) => {
        // Success path - process completed
        let (stdout, stdout_truncated) = Self::truncate_output(&output.stdout, MAX_OUTPUT_BYTES);
        let (stderr, stderr_truncated) = Self::truncate_output(&output.stderr, MAX_OUTPUT_BYTES);
        let truncated = stdout_truncated || stderr_truncated;
        (output.status.code(), stdout, stderr, false, truncated)
    }
    Ok(Err(e)) => {
        // Process spawn/execution error
        return Err(ToolError::Io(e.to_string()));
    }
    Err(_) => {
        // Timeout fired
        tracing::warn!(
            command = %args.command,
            timeout_secs = timeout_secs,
            "bash command timed out"
        );
        (None, String::new(), String::new(), true, false)
    }
};
```

### Output Truncation Implementation

**`crates/loom-cli-tools/src/bash.rs` (lines 66-74)**

```rust
fn truncate_output(output: &[u8], max_bytes: usize) -> (String, bool) {
    if output.len() <= max_bytes {
        (String::from_utf8_lossy(output).to_string(), false)
    } else {
        let truncated_bytes = &output[..max_bytes];
        let content = String::from_utf8_lossy(truncated_bytes).to_string();
        (content, true)
    }
}
```

### Result Structure

**`crates/loom-cli-tools/src/bash.rs` (lines 26-32)**

```rust
#[derive(Debug, Serialize)]
struct BashResult {
    exit_code: Option<i32>,
    stdout: String,
    stderr: String,
    timed_out: bool,
    truncated: bool,
}
```

### Input Schema (JSON)

**`crates/loom-cli-tools/src/bash.rs` (lines 93-114)**

```json
{
    "type": "object",
    "properties": {
        "command": {
            "type": "string",
            "description": "The shell command to execute"
        },
        "cwd": {
            "type": "string",
            "description": "Working directory relative to workspace (default: workspace root)"
        },
        "timeout_secs": {
            "type": "integer",
            "minimum": 1,
            "maximum": 300,
            "description": "Timeout in seconds (default: 60, max: 300)"
        }
    },
    "required": ["command"]
}
```

### Test Coverage

**`crates/loom-cli-tools/src/bash.rs` (lines 301-334)**

```rust
#[tokio::test]
async fn bash_times_out() {
    let workspace = setup_workspace();
    let tool = BashTool::new();
    let ctx = ToolContext::new(workspace.path().to_path_buf());

    let result = tool
        .invoke(
            serde_json::json!({"command": "sleep 10", "timeout_secs": 1}),
            &ctx,
        )
        .await
        .unwrap();

    assert_eq!(result["timed_out"], true);
    assert!(result["exit_code"].is_null());
}

#[tokio::test]
async fn bash_truncates_large_output() {
    let workspace = setup_workspace();
    let tool = BashTool::new();
    let ctx = ToolContext::new(workspace.path().to_path_buf());

    // Generate output larger than MAX_OUTPUT_BYTES
    let result = tool
        .invoke(serde_json::json!({"command": "yes | head -c 300000"}), &ctx)
        .await
        .unwrap();

    assert_eq!(result["exit_code"], 0);
    assert_eq!(result["truncated"], true);
    let stdout_len = result["stdout"].as_str().unwrap().len();
    assert!(stdout_len <= MAX_OUTPUT_BYTES);
}
```

**Key Files:**
- `crates/loom-cli-tools/src/bash.rs` — Complete bash tool implementation
- `specs/tool-system.md` — Tool system specification documenting bash behavior
- `crates/loom-common-core/src/tool.rs` — ToolContext definition

## Implications

- **Orphaned processes are possible**: When a timeout fires, the child process is not explicitly killed. The Tokio future is dropped, but the underlying OS process may continue running. This is a known limitation - future work could add explicit `SIGTERM`/`SIGKILL` handling.

- **Memory usage is bounded**: Even if a command produces gigabytes of output, only 256KB per stream is returned. However, the full output is first captured in memory before truncation, so extremely large outputs could still cause memory pressure.

- **Timeout is wall-clock time**: The timeout measures total elapsed time, not CPU time. A command that sleeps for 59 seconds then does 2 seconds of work will timeout at 60s.

- **Exit code is null on timeout**: When `timed_out: true`, the `exit_code` field is `null` (JSON) / `None` (Rust) because the process didn't exit normally.

- **Truncation is per-stream**: stdout and stderr are each allowed 256KB independently. A command could return up to 512KB total (256KB stdout + 256KB stderr).

- **Working directory validation**: The `cwd` parameter is validated against the workspace root using the same canonicalize-then-prefix-check pattern as other tools (see discovery 015).

## Follow-up Questions

- [ ] How does the EBPF sandbox escape detection work in detail? (syscall monitoring, event types)
- [ ] How does the session management differ from OAuth state (persistence, expiry, revocation)?

## Related Discoveries

- [[015-path-validation-traversal-prevention]] — Working directory validation uses same pattern
- [[004-tool-execution-system]] — Tool execution lifecycle and ToolContext
