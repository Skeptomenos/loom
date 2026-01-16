# Discovery: What would be the performance impact of adding advisory file locks (flock) to edit_file and bash tools?

> Category: Follow-up (from Architecture)
> Discovered: 2026-01-16
> Confidence: High

## Answer

Adding advisory file locks (`flock`) to `edit_file` and `bash` tools would have **minimal direct performance overhead** (~1-5 microseconds per lock/unlock syscall) but introduces **significant complexity and potential blocking behavior** that could degrade perceived performance under contention. The real cost is not the syscall itself, but the **waiting time when locks are contested** and the **architectural complexity** of integrating blocking syscalls into an async runtime.

The current sequential execution approach is a deliberate trade-off that avoids these complexities. If parallel tool execution is desired in the future, a **workspace-level lock manager** or **path-based locking** would be more appropriate than per-tool flock integration.

## Evidence

### 1. Current Implementation Has No Locking

The `edit_file` tool uses a classic read-modify-write pattern without any synchronization:

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

The `bash` tool spawns shell processes with no coordination:

```rust
// crates/loom-cli-tools/src/bash.rs:141-144
let mut cmd = Command::new("sh");
cmd.arg("-c").arg(&args.command).current_dir(&working_dir);
let result = timeout(Duration::from_secs(timeout_secs), cmd.output()).await;
```

### 2. flock Syscall Overhead is Minimal

Research on Rust file locking crates confirms:

| Operation | Typical Latency | Notes |
|-----------|-----------------|-------|
| `flock(LOCK_EX)` (uncontested) | ~1-5 μs | Single syscall, negligible |
| `flock(LOCK_EX)` (contested) | **Unbounded** | Blocks until lock released |
| `flock(LOCK_UN)` | ~1-2 μs | Single syscall |

The overhead of the syscall itself is negligible compared to file I/O operations (which are typically 10-100+ μs for small files, much more for large files or slow storage).

### 3. Async Integration Complexity

The real challenge is integrating blocking `flock` calls into Tokio's async runtime. Blocking the executor thread starves other tasks:

```rust
// WRONG: Blocks the async executor thread
async fn edit_with_lock(path: &Path) {
    let file = File::open(path).await?;
    file.lock_exclusive()?;  // BLOCKS THE EXECUTOR!
    // ...
}

// CORRECT: Use spawn_blocking to avoid starving the executor
async fn edit_with_lock(path: &Path) {
    let file = File::open(path).await?;
    tokio::task::spawn_blocking(move || {
        file.lock_exclusive()  // Blocks a dedicated thread, not the executor
    }).await??;
    // ...
}
```

This adds complexity and thread pool overhead.

### 4. Existing Atomic Write Pattern in Codebase

The codebase already uses atomic writes (temp file + rename) for thread storage, which is a simpler pattern that avoids locking:

```rust
// crates/loom-common-thread/src/store.rs:167-176
async fn save(&self, thread: &Thread) -> Result<(), ThreadStoreError> {
    let path = self.thread_path(&thread.id);
    let tmp_path = self.threads_dir.join(format!("{}.json.tmp", thread.id));

    let json = serde_json::to_string_pretty(thread)?;

    tokio::fs::write(&tmp_path, &json).await?;
    tokio::fs::rename(&tmp_path, &path).await?;  // Atomic on most filesystems
    // ...
}
```

This pattern ensures writes are atomic but doesn't prevent read-modify-write race conditions.

### 5. flock is Already Used in Infrastructure

The NixOS auto-update script uses `flock` to prevent concurrent deployments:

```bash
# infra/nixos-modules/nixos-auto-update.nix:102-104
# Use flock to prevent concurrent updates - exit silently if already running
exec 200>/var/lock/nixos-auto-update.lock
if ! flock -n 200; then
```

This demonstrates the pattern works for coarse-grained coordination but is overkill for per-file tool operations.

### 6. Available Rust Crates

| Crate | Status | Async Support | Notes |
|-------|--------|---------------|-------|
| `fs4` | Active, recommended | Yes (via traits) | Modern fork of `fs2`, uses `rustix` |
| `fs2` | Legacy, unmaintained | No | Still widely used |
| `fd-lock` | Active | No | Simple, used by Deno |
| `file-lock` | Active | No | Cross-platform, auto-unlock on Drop |

### 7. Performance Impact Analysis

| Scenario | Without flock | With flock (uncontested) | With flock (contested) |
|----------|---------------|--------------------------|------------------------|
| Single edit_file | ~1-10 ms | ~1-10 ms + 5 μs | ~1-10 ms + 5 μs |
| 10 sequential edits | ~10-100 ms | ~10-100 ms + 50 μs | ~10-100 ms + 50 μs |
| 10 parallel edits (same file) | **Race condition** | ~10-100 ms (serialized) | ~10-100 ms (serialized) |
| 10 parallel edits (different files) | ~10-100 ms | ~10-100 ms + 50 μs | ~10-100 ms + 50 μs |

**Key insight**: The performance impact is negligible for uncontested locks. The "cost" of flock is really the **serialization of contested operations**, which is exactly what sequential execution already provides without the complexity.

**Key Files:**
- `crates/loom-cli-tools/src/edit_file.rs` — Read-modify-write without locking
- `crates/loom-cli-tools/src/bash.rs` — Shell execution without coordination
- `crates/loom-common-thread/src/store.rs` — Atomic write pattern (temp + rename)
- `infra/nixos-modules/nixos-auto-update.nix` — flock usage example
- `discoveries/004-sequential-execution-rationale.md` — Documents why sequential was chosen

## Implications

When working in this codebase:

- **flock overhead is not a concern** — The ~5 μs per lock/unlock is negligible compared to file I/O. Performance is not a valid reason to avoid flock.

- **Contention is the real cost** — If parallel execution is enabled, contested locks would serialize operations anyway, providing no speedup over sequential execution for same-file operations.

- **Async integration requires care** — Blocking `flock` calls must be wrapped in `spawn_blocking` to avoid starving the Tokio executor. This adds complexity.

- **Sequential execution is simpler and equivalent** — For the current use case (single agent, single workspace), sequential execution provides the same guarantees as flock with less complexity.

- **flock would enable safe parallelism** — If future requirements demand parallel tool execution (e.g., multiple agents, background tasks), flock would be necessary. The `fs4` crate is the recommended choice.

- **Consider a WorkspaceLockManager instead** — Rather than adding flock to each tool, a centralized lock manager that tracks which paths are being modified would be cleaner and allow for smarter scheduling (e.g., parallelize operations on different files).

## Alternative Approaches

| Approach | Pros | Cons |
|----------|------|------|
| **Sequential execution (current)** | Simple, no locking code | No parallelism |
| **Per-file flock in tools** | Enables safe parallelism | Async complexity, each tool must implement |
| **WorkspaceLockManager** | Centralized, smarter scheduling | More architecture work |
| **Atomic writes only** | Simple, no blocking | Doesn't prevent read-modify-write races |
| **Copy-on-write / OT** | Conflict resolution | Major architecture change |

## Follow-up Questions

- [ ] Are there any read-only tools that could be safely parallelized without any changes?
- [ ] Should a WorkspaceLockManager be implemented as a prerequisite for parallel tool execution?

## Related Discoveries

- [[004-sequential-execution-rationale]] — Documents why sequential execution was chosen over file locking
- [[002-concurrent-tool-execution]] — Establishes that the state machine supports parallelism but CLI doesn't use it
