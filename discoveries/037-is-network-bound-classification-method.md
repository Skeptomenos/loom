# Discovery: Should there be additional classification methods like `is_network_bound()` to enable smarter execution strategies?

> Category: Follow-up (from 020-tool-trait-is-mutating-method, 036-tooldefinition-is-mutating-field)
> Discovered: 2026-01-16
> Confidence: Medium

## Answer

**Not recommended at this time.** While an `is_network_bound()` method would provide additional classification granularity, the practical benefits are marginal compared to the already-proposed `is_mutating()` method. The primary value of tool classification—enabling parallel execution of read-only tools—is fully captured by the mutating/non-mutating distinction.

Adding `is_network_bound()` would:
1. **Add complexity without proportional benefit**: The main execution strategy (parallel read-only, sequential mutating) doesn't change based on network vs local I/O
2. **Create maintenance burden**: Each new tool would need to declare multiple classification flags
3. **Solve a problem that doesn't exist yet**: There's no current use case in the codebase that would benefit from this distinction

**When it WOULD be valuable**: If Loom later implements:
- Concurrency limits (e.g., max 4 concurrent network requests to avoid rate limiting)
- Differentiated timeout policies (longer for network, shorter for local)
- Resource-aware scheduling (prioritize network tools when CPU is busy)

Until then, the simpler `is_mutating()` classification is sufficient.

## Evidence

### Current Tool Characteristics

| Tool | Mutating | I/O Type | Typical Latency | Timeout |
|------|----------|----------|-----------------|---------|
| `read_file` | No | Local filesystem | <10ms | None |
| `list_files` | No | Local filesystem | <10ms | None |
| `edit_file` | **Yes** | Local filesystem | <10ms | None |
| `bash` | **Yes** | Local (subprocess) | Variable | 60-300s |
| `web_search` | No | **Network (HTTP)** | 1-5s | 30s |
| `oracle` | No | **Network (HTTP)** | 2-10s | 60s |

### Network-Bound Tools Already Have Distinct Patterns

```rust
// crates/loom-cli-tools/src/web_search.rs:107-120
let retry_config = RetryConfig::default();
let response = retry(&retry_config, || {
    // ...
    client
        .post(url)
        .json(request_body)
        .timeout(std::time::Duration::from_secs(30))  // Explicit timeout
        .send()
        .await
})
.await

// crates/loom-cli-tools/src/oracle.rs:200-207
let retry_config = RetryConfig {
    max_attempts: 3,
    base_delay: Duration::from_millis(500),
    max_delay: Duration::from_secs(10),
    backoff_factor: 2.0,
    jitter: true,
    ..Default::default()
};
```

Network-bound tools already implement their own timeout and retry logic. An `is_network_bound()` flag wouldn't change this—each tool knows its own characteristics.

### Parallel Execution Doesn't Need This Distinction

The proposed parallel execution strategy (from discovery-019) is:

```rust
// Partition by mutating status only
let (read_only, mutating): (Vec<_>, Vec<_>) = tool_calls
    .iter()
    .partition(|tc| !MUTATING_TOOLS.contains(&tc.tool_name.as_str()));

// Execute read-only in parallel (regardless of network vs local)
let read_only_results = join_all(read_only_futures).await;

// Execute mutating sequentially
for tc in &mutating {
    // ...
}
```

Whether a read-only tool is network-bound or local doesn't affect this strategy. Both benefit equally from parallel execution.

### No Concurrency Limits Exist Today

Searching for `Semaphore` in the codebase reveals no usage for tool execution limiting:

```
$ grep -r "Semaphore" crates/loom-cli-tools/
(no results)

$ grep -r "Semaphore" crates/loom-cli/
(no results)
```

The codebase doesn't currently limit concurrent tool executions. If it did, `is_network_bound()` could inform those limits (e.g., max 4 concurrent HTTP requests).

### Rate Limiting Is Handled Server-Side

```rust
// crates/loom-server/src/query_security.rs:242-257
/// Token bucket rate limiter for per-session query rate limiting.
pub struct RateLimiter {
    buckets: DashMap<String, TokenBucket>,
    rate: u32,
    burst: u32,
}
```

Rate limiting for external APIs (Google CSE, Serper, OpenAI) is handled at the server proxy level, not in the CLI tools. The tools don't need to know they're network-bound for rate limiting purposes.

**Key Files:**
- `crates/loom-cli-tools/src/web_search.rs` — Network-bound tool with explicit timeout/retry
- `crates/loom-cli-tools/src/oracle.rs` — Network-bound tool with explicit timeout/retry
- `crates/loom-cli-tools/src/read_file.rs` — Local I/O tool (no timeout needed)
- `crates/loom-server/src/query_security.rs` — Server-side rate limiting

## Implications

1. **Defer this classification**: The `is_mutating()` method should be implemented first (as proposed in discovery-020). Only add `is_network_bound()` if a concrete use case emerges.

2. **Alternative: Execution hints enum**: If multiple classification dimensions become needed, consider a single `execution_hints()` method returning a struct:
   ```rust
   struct ExecutionHints {
       is_mutating: bool,
       is_network_bound: bool,
       expected_latency_ms: Option<u32>,
       max_concurrency: Option<u32>,
   }
   ```
   This would be more extensible than adding individual boolean methods.

3. **Current tools are self-aware**: Network-bound tools already configure their own timeouts and retries. They don't need external classification to behave correctly.

4. **Future use cases that would justify this**:
   - Implementing a `--max-concurrent-network-tools N` CLI option
   - Adding a "network health" indicator to the TUI
   - Implementing circuit breakers for external services
   - Resource-aware scheduling in a multi-agent environment

## Follow-up Questions

<!-- No new follow-up questions generated - this discovery concludes the classification thread -->

## Related Discoveries

- [[020-tool-trait-is-mutating-method]] — Proposes the primary classification method (prerequisite)
- [[036-tooldefinition-is-mutating-field]] — Extends classification to ToolDefinition
- [[011-read-only-tools-parallelization]] — Identifies parallelization candidates
- [[019-parallel-read-only-tool-implementation-complexity]] — Implementation plan that uses classification
