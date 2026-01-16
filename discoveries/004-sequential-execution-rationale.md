# Discovery: Why was sequential execution chosen over parallel in the CLI implementations?

> Category: Follow-up (from Architecture)
> Discovered: 2026-01-16
> Confidence: High

## Answer

Sequential tool execution in the CLI was chosen for **three primary reasons**: avoiding race conditions in file operations, simplifying debugging, and ensuring predictable behavior. The file-modifying tools (`edit_file`, `bash`) have **no internal synchronization mechanisms** (no file locks, mutexes, or atomic operations), making them vulnerable to **Lost Update** race conditions if executed in parallel.

The `edit_file` tool uses a classic **read-modify-write** pattern: it reads the entire file into memory, applies string replacements, then writes the result back. If two `edit_file` calls target the same file concurrently, the second write would overwrite the first's changes entirely. Similarly, `bash` can execute arbitrary shell commands that may modify files, and there's no way to predict or prevent conflicts.

Rather than implementing complex file locking (which would add overhead and potential deadlocks), the developers chose the simpler approach of **sequential execution at the caller level**. This is a pragmatic trade-off: the state machine remains "parallel-ready" for future optimization, while the current CLI avoids the complexity of concurrent file access.

## Evidence

### Read-Modify-Write Pattern in edit_file (No Locking)

```rust
// crates/loom-cli-tools/src/edit_file.rs:145-203
let mut content = if file_path.exists() {
    tokio::fs::read_to_string(&file_path).await?  // READ
} else {
    String::new()
};

// ... apply edits to `content` ...  // MODIFY

tokio::fs::write(&file_path, &content).await?;  // WRITE
```

There is no `flock`, `FileLock`, or `Mutex` protecting this sequence. If two tasks execute this concurrently on the same file:
1. Task A reads file (content = "hello")
2. Task B reads file (content = "hello")
3. Task A modifies and writes (content = "hello world")
4. Task B modifies and writes (content = "hello there") — **overwrites Task A's changes**

### Bash Tool Has No Synchronization

```rust
// crates/loom-cli-tools/src/bash.rs:130-170
let output = timeout(
    Duration::from_secs(timeout_secs),
    Command::new("bash")
        .arg("-c")
        .arg(&args.command)
        .current_dir(&cwd)
        .output(),
)
.await;
```

The `bash` tool spawns a shell process with no coordination. If two `bash` calls both run `echo "data" >> file.txt`, the output order is non-deterministic.

### Sequential Loop in CLI

```rust
// crates/loom-cli/src/main.rs:612-656
for tool_call in &tool_calls {
    let outcome = execute_tool(tool_registry, tool_call, tool_ctx).await;
    // ... process outcome ...
    messages.push(Message::tool(&tool_call.id, &tool_call.tool_name, &tool_result));
}
```

The `for` loop with `.await` ensures each tool completes before the next begins.

### State Machine Supports Parallelism (Unused)

```rust
// specs/state-machine.md:39
| `ExecutingTools` | Running one or more tool calls in parallel | ...

// crates/loom-common-core/src/state.rs:176-179
ExecutingTools {
    conversation: ConversationContext,
    executions: Vec<ToolExecutionStatus>,  // Tracks MULTIPLE concurrent executions
},
```

The architecture is ready; the CLI simply doesn't use it.

**Key Files:**
- `crates/loom-cli-tools/src/edit_file.rs` — Read-modify-write without locking
- `crates/loom-cli-tools/src/bash.rs` — Shell execution without coordination
- `crates/loom-cli/src/main.rs` — Sequential `for` loop at lines 612-656
- `crates/loom-cli-acp/src/agent.rs` — Sequential `for` loop at lines 472-488
- `specs/state-machine.md` — Documents parallel execution intent

## Implications

When working in this codebase:

- **Do not enable parallel tool execution without adding synchronization** — The current tools will corrupt files if run concurrently on the same paths. At minimum, you'd need advisory file locks or a workspace-level mutex.

- **The sequential design is intentional, not a limitation** — It's a pragmatic trade-off that simplifies debugging and avoids complex locking logic. The state machine is already parallel-ready if needed.

- **Read-only tools could be parallelized safely** — Tools like `read_file`, `list_files`, and `glob` don't modify state and could theoretically run in parallel without issues.

- **Future parallelism would require tool-level changes** — Either implement `flock`-style advisory locks in `edit_file`/`bash`, or add a centralized "workspace lock manager" that serializes access to specific paths.

- **LLM tool call ordering matters** — When the LLM requests multiple tool calls, they're executed in the order received. If the LLM expects a specific sequence (e.g., create file then edit it), sequential execution guarantees correctness.

## Follow-up Questions

- [ ] What would be the performance impact of adding advisory file locks (flock) to edit_file and bash tools?
- [ ] Are there any read-only tools that could be safely parallelized without any changes?

## Related Discoveries

- [[002-concurrent-tool-execution]] — Establishes that the state machine supports parallelism but CLI doesn't use it
- [[003-request-flow]] — Documents the tool execution phase in the request lifecycle
