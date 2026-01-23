# Common Gotchas

This document captures non-obvious behaviors, quirks, and pitfalls that developers should know.

---

## Critical Gotchas

### 1. API Keys Should Never Leave the Server

**What**: Client applications should never have direct access to third-party API keys. All requests should go through a server-side proxy.

**Why It Matters**: 
- API keys in client code can be extracted and abused
- Server-side proxies enable centralized rate limiting and credential rotation
- When debugging, check both client-side and server-side logs

**Pattern**:
```
Client (ProxyClient) → HTTP → Server (ProxyService) → External API
```

---

### 2. Sequential Execution May Be Intentional

**What**: Some systems execute operations sequentially even when parallelization is possible.

**Why**: Without proper locking mechanisms, parallel execution of file operations or database writes can cause race conditions and data corruption.

**Implication**: Don't enable parallel execution without implementing proper synchronization (locks, transactions, etc.).

---

### 3. Passive State Machines Need External Drivers

**What**: State machines that return actions but don't drive themselves require an external event loop.

**Why It Matters**: The caller must implement the event loop:

```rust
loop {
    let action = state_machine.handle_event(event);
    match action {
        Action::SendRequest(req) => { /* caller sends request */ }
        Action::ProcessData(data) => { /* caller processes */ }
        Action::Wait => { /* caller waits */ }
    }
}
```

**Implication**: If you're building a new consumer, you must implement the full event loop.

---

### 4. Retry Mechanisms May Be Incomplete

**What**: Retry logic may exist in code but never be triggered.

**Why It Matters**: 
- Event-driven retries require external timers to fire retry events
- If no consumer fires `RetryTimeoutFired`, retries never happen

**Check**: Verify retry paths are actually exercised in your runtime.

---

### 5. Build Tool File Tracking Limitations

**What**: Some build tools don't track files included at compile time (e.g., `include_str!`, embedded resources).

**Why**: 
- Adding a migration or embedded file may not trigger rebuilds
- Deployed binaries may not include new files

**Fix**: Force rebuild after adding embedded files:
```bash
# Touch a tracked file or clean build
cargo clean -p project-name && cargo build
```

---

### 6. Framework Version Mixing Causes Subtle Bugs

**What**: Mixing framework versions (e.g., Svelte 4 and Svelte 5 patterns) causes subtle, hard-to-debug issues.

**Svelte 5 Only**:

| Correct (Svelte 5) | Incorrect (Svelte 4) |
|-------------------|---------------------|
| `let { foo } = $props();` | `export let foo;` |
| `const x = $derived(y * 2);` | `$: x = y * 2;` |
| `onclick={handler}` | `on:click={handler}` |
| `{@render children()}` | `<slot />` |

---

## Implementation Gotchas

### 7. Hardcoded Classification Lists

**What**: Lists of items with special behavior (e.g., "mutating operations") may be hardcoded.

**Why It Matters**: Adding a new item with that behavior requires updating the list manually.

**Better Solution**: Add a method to the interface (e.g., `is_mutating()`) so implementations self-describe.

```rust
// Instead of hardcoded list
const MUTATING_ITEMS: &[&str] = &["write", "delete", "update"];

// Use trait method
trait Operation {
    fn is_mutating(&self) -> bool;
}
```

---

### 8. Retry-After Headers May Be Ignored

**What**: When servers return 429/503 with `Retry-After`, clients may ignore it.

**Implication**: Clients can't implement intelligent backoff based on server hints.

**Fix**: Parse and respect `Retry-After` header:
```rust
if let Some(retry_after) = response.headers().get("Retry-After") {
    let secs: u64 = retry_after.to_str()?.parse()?;
    tokio::time::sleep(Duration::from_secs(secs)).await;
}
```

---

### 9. Streaming Connections May Lose Partial Data

**What**: If a streaming connection (SSE, WebSocket) drops mid-message, partial data is lost.

**Why It Matters**: 
- Long responses may be truncated without error indication
- Most clients don't implement automatic reconnection with resume

**Mitigation**: Implement heartbeats and consider message acknowledgment.

---

### 10. Generic vs Domain-Specific Error Codes

**What**: API error codes fall into two categories:
- **Generic** (redundant with HTTP status): `not_found`, `bad_request`
- **Domain-specific** (actionable): `already_member`, `quota_exceeded`

**Implication**: When displaying errors:
- Include domain-specific codes (they add information)
- Omit generic codes (they duplicate status code)

---

## Performance Gotchas

### 11. Read-Only Operations May Run Sequentially

**What**: Even read-only operations (like file reads, searches) may run sequentially.

**Opportunity**: Read-only operations can safely run in parallel.

**Implementation**: Classify operations and parallelize read-only ones:
```rust
if operations.iter().all(|op| !op.is_mutating()) {
    futures::future::join_all(operations.map(|op| op.execute())).await
} else {
    for op in operations {
        op.execute().await?;
    }
}
```

---

### 12. Full-Text Search Requires Specific Query Syntax

**What**: Full-text search engines (SQLite FTS5, Elasticsearch) have specific query syntax.

**Gotcha**: Queries with special characters may fail or return unexpected results.

**Fix**: 
- Escape special characters
- Use phrase queries with quotes
- Validate/sanitize user input

---

## Debugging Gotchas

### 13. Multiple Error Sources

**What**: Errors can occur at multiple layers:
1. **Client side**: HTTP errors, connection issues, timeouts
2. **Proxy/Gateway**: Routing errors, auth failures
3. **Server side**: Business logic errors, database errors
4. **External service**: Rate limits, validation failures

**Tip**: Always check logs at each layer when debugging.

---

### 14. Ephemeral Resources May Exit Immediately

**What**: Containers or processes may show "completed" status immediately after starting.

**Cause**: The entrypoint has no long-running process.

**Fix**: Ensure containers have a persistent process (e.g., `tail -f /dev/null` for debugging, or your actual service).

**Debug**: Check container logs and describe output.

---

### 15. Image Pull Failures

**What**: Container orchestrators can't pull images.

**Causes**:
- Image doesn't exist in registry
- Image is private and no pull secret configured
- Typo in image name/tag
- Registry is unreachable

**Debug**: Check orchestrator events for exact error message.

---

## Security Gotchas

### 16. Secret Wrappers Only Prevent Accidental Logging

**What**: `Secret<T>` auto-redacts in Debug/Display, but `.expose()` returns raw value.

**Not Protected Against**:
- Intentional exposure via `.expose()`
- Memory dumps
- Core dumps
- Serialization (unless explicitly handled)

**Rule**: Never log the result of `.expose()`.

---

### 17. Dev Mode Bypasses Security

**What**: Development modes may disable authentication entirely.

**Danger**: Never enable in production.

**Safeguards**:
```rust
if cfg!(not(debug_assertions)) && env::var("DEV_MODE").is_ok() {
    panic!("DEV_MODE cannot be enabled in release builds");
}
```

---

## Build System Gotchas

### 18. Embedded File Changes Not Tracked

**What**: Files loaded via `include_str!`, `include_bytes!`, or similar may not trigger rebuilds.

**Why**: Build systems track source files, not data files.

**Fix**: 
- Add build script that touches a tracked file
- Clean and rebuild after adding embedded files
- Use build tool plugins for asset tracking

---

### 19. Multiple Remotes Confusion

**What**: Repositories may have multiple remotes (origin, fork, upstream).

**Gotcha**: Pushing to the wrong remote fails or goes to unexpected location.

**Fix**: 
- Use explicit remote names: `git push fork feature-branch`
- Set upstream tracking: `git push -u fork feature-branch`
- Check remotes: `git remote -v`

---

## Quick Reference: Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Forgot to rebuild after embedded file change | File not in deployed binary | Clean rebuild or touch tracked file |
| Mixed framework versions | Subtle reactivity bugs | Use only latest version patterns |
| Pushed to wrong remote | Permission denied or wrong repo | Use explicit remote name |
| Expected parallel execution | Operations run slowly | May be intentional; check for locks |
| Expected auto-retry | Errors not retried | Verify retry timer is implemented |
| Added new operation type | Special behavior not triggered | Update classification list or add method |
| Constructed HTTP client directly | Missing headers, no retries | Use factory function |
| Logged secret directly | Credentials in logs | Use Secret wrapper type |
| Enabled dev mode in production | Security bypass | Add production safeguards |

---

## See Also

- [conventions.md](conventions.md) - Coding conventions
- [patterns.md](patterns.md) - Design patterns
- [error-handling.md](error-handling.md) - Error handling
- [testing.md](testing.md) - Testing strategy
