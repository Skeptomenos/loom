# Discovery: Are there any read-only tools that could be safely parallelized without any changes?

> Category: Follow-up (from 004-sequential-execution-rationale)
> Discovered: 2026-01-16
> Confidence: High

## Answer

Yes, **four tools are read-only and could be safely parallelized without any code changes**: `read_file`, `list_files`, `web_search`, and `oracle`. These tools have no side effects on the local workspace or system state. The codebase already contains explicit infrastructure to distinguish mutating from non-mutating tools via the `has_mutating_tools()` function in `loom-common-core`, which defines `MUTATING_TOOLS` as only `["edit_file", "bash"]`.

The read-only tools fall into two categories:
1. **Filesystem readers** (`read_file`, `list_files`) — Only read file contents and directory metadata within the workspace boundary
2. **External API callers** (`web_search`, `oracle`) — Make HTTP requests to external services (Google/Serper search, OpenAI) without modifying local state

All four tools are stateless and idempotent. Multiple concurrent invocations would not interfere with each other, even when targeting the same files or queries. The only constraint is that they must respect the workspace boundary (which they already do via path validation).

## Evidence

### Explicit Mutating Tool Definition

```rust
// crates/loom-common-core/src/agent.rs:48-63
fn has_mutating_tools(executions: &[ToolExecutionStatus]) -> bool {
    const MUTATING_TOOLS: &[&str] = &["edit_file", "bash"];
    executions.iter().any(|exec| {
        if let ToolExecutionStatus::Completed {
            tool_name, outcome, ..
        } = exec
        {
            MUTATING_TOOLS.contains(&tool_name.as_str())
                && matches!(outcome, ToolExecutionOutcome::Success { .. })
        } else {
            false
        }
    })
}
```

This function explicitly lists only `edit_file` and `bash` as mutating. By exclusion, all other tools are considered non-mutating.

### read_file — Pure Read Operation

```rust
// crates/loom-cli-tools/src/read_file.rs:106-121
let metadata = tokio::fs::metadata(&canonical_path).await?;
let file_size = metadata.len();
let truncated = file_size > max_bytes;

let contents = if truncated {
    let bytes = tokio::fs::read(&canonical_path).await?;
    String::from_utf8_lossy(&bytes[..max_bytes as usize]).to_string()
} else {
    tokio::fs::read_to_string(&canonical_path).await?
};
```

Uses only `tokio::fs::metadata`, `tokio::fs::read`, and `tokio::fs::read_to_string`. No write operations.

### list_files — Pure Directory Read

```rust
// crates/loom-cli-tools/src/list_files.rs:113-132
let mut read_dir = tokio::fs::read_dir(&root_path).await?;

while let Some(entry) = read_dir.next_entry().await? {
    if entries.len() >= max_results {
        break;
    }
    let path = entry.path();
    let metadata = entry.metadata().await?;
    entries.push(FileEntry {
        path,
        is_dir: metadata.is_dir(),
    });
}
```

Uses only `tokio::fs::read_dir` and `entry.metadata()`. No write operations.

### web_search — External HTTP Only

```rust
// crates/loom-cli-tools/src/web_search.rs:108-120
let response = retry(&retry_config, || {
    let client = &self.client;
    let url = &url;
    let request_body = &request_body;
    async move {
        client
            .post(url)
            .json(request_body)
            .timeout(std::time::Duration::from_secs(30))
            .send()
            .await
    }
})
.await
```

Makes HTTP POST to `/proxy/cse` or `/proxy/serper`. No local filesystem access.

### oracle — External LLM Query Only

```rust
// crates/loom-cli-tools/src/oracle.rs:213-238
let result = retry(&retry_config, || {
    let url = url.clone();
    let request_json = request_json.clone();
    async move {
        let response = client
            .post(&url)
            .json(&request_json)
            .timeout(Duration::from_secs(60))
            .send()
            .await
            .map_err(OracleError::Request)?;
        // ...
    }
})
.await;
```

Makes HTTP POST to `/proxy/openai/complete`. No local filesystem access.

**Key Files:**
- `crates/loom-common-core/src/agent.rs` — Defines `MUTATING_TOOLS` constant
- `crates/loom-cli-tools/src/read_file.rs` — Read-only file access
- `crates/loom-cli-tools/src/list_files.rs` — Read-only directory listing
- `crates/loom-cli-tools/src/web_search.rs` — External HTTP requests only
- `crates/loom-cli-tools/src/oracle.rs` — External LLM queries only

## Implications

When working in this codebase:

- **Parallel execution of read-only tools is safe today** — No code changes are needed in the tools themselves. The parallelization logic would only need to be added to the execution loop in `loom-cli/src/main.rs` and `loom-cli-acp/src/agent.rs`.

- **Implementation approach**: Before executing a batch of tool calls, partition them into read-only and mutating sets. Execute all read-only tools in parallel using `futures::future::join_all` or `tokio::spawn`, then execute mutating tools sequentially.

- **The `has_mutating_tools()` function can be repurposed** — Currently used for auto-commit decisions, but the same `MUTATING_TOOLS` constant could drive parallel execution logic.

- **Network-bound tools benefit most from parallelization** — `web_search` and `oracle` have significant latency (HTTP round-trips). Running multiple searches or oracle queries in parallel could dramatically reduce total execution time.

- **Filesystem read tools have lower parallelization benefit** — `read_file` and `list_files` are typically fast (local I/O), but parallelizing them still helps when the LLM requests multiple file reads simultaneously.

- **Consider adding a `is_read_only()` method to the Tool trait** — This would make the read-only classification explicit and extensible, rather than relying on a hardcoded exclusion list.

## Follow-up Questions

- [ ] What would be the implementation complexity of adding parallel execution for read-only tool batches to the CLI?
- [ ] Should a `is_read_only()` or `is_mutating()` method be added to the Tool trait for explicit classification?

## Related Discoveries

- [[004-sequential-execution-rationale]] — Explains why sequential execution was chosen and identifies read-only tools as parallelization candidates
- [[002-concurrent-tool-execution]] — Documents the state machine's existing support for tracking multiple concurrent executions
- [[010-flock-performance-impact]] — Analyzes the alternative approach of adding file locks to mutating tools
