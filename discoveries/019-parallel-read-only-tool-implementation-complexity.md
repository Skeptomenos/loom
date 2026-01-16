# Discovery: What would be the implementation complexity of adding parallel execution for read-only tool batches to the CLI?

> Category: Follow-up (from 011-read-only-tools-parallelization)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Implementation complexity is LOW to MEDIUM.** The core infrastructure already exists: the `futures` crate is a dependency in both `loom-cli` and `loom-cli-acp`, the state machine supports tracking multiple concurrent tool executions, and read-only tools are already implicitly identified via the `MUTATING_TOOLS` exclusion list. The main work involves:

1. **Partitioning tool calls** (~20 lines): Split incoming `tool_calls` into read-only and mutating sets
2. **Parallel execution** (~15 lines): Use `futures::future::join_all` for read-only tools
3. **Result ordering** (~10 lines): Maintain original order when collecting results for the LLM
4. **Testing** (~50 lines): Add tests for parallel execution correctness

**Estimated total: ~100 lines of code changes per execution site** (CLI and ACP agent), plus ~50 lines for a shared helper if extracted. The changes are localized to two files and don't require modifications to the state machine, Tool trait, or individual tool implementations.

## Evidence

### Infrastructure Already in Place

**1. `futures` crate is already a dependency:**
```toml
# crates/loom-cli/Cargo.toml:23
futures = { workspace = true }

# crates/loom-cli-acp/Cargo.toml:42
futures = { workspace = true }
```

**2. Read-only tool identification exists:**
```rust
// crates/loom-common-core/src/agent.rs:50-51
fn has_mutating_tools(executions: &[ToolExecutionStatus]) -> bool {
    const MUTATING_TOOLS: &[&str] = &["edit_file", "bash"];
    // ...
}
```

By exclusion, read-only tools are: `read_file`, `list_files`, `web_search`, `oracle`, `glob`.

**3. State machine supports parallel tracking:**
```rust
// crates/loom-common-core/src/state.rs:176-179
ExecutingTools {
    conversation: ConversationContext,
    executions: Vec<ToolExecutionStatus>,  // Already tracks multiple concurrent executions
},
```

### Current Sequential Implementation (to be modified)

```rust
// crates/loom-cli/src/main.rs:612-656
for tool_call in &tool_calls {
    let outcome = execute_tool(tool_registry, tool_call, tool_ctx).await;
    // ... process outcome ...
    messages.push(Message::tool(&tool_call.id, &tool_call.tool_name, &tool_result));
}
```

### Proposed Implementation

```rust
// Pseudocode for parallel read-only execution
use futures::future::join_all;

const MUTATING_TOOLS: &[&str] = &["edit_file", "bash"];

// 1. Partition tool calls
let (read_only, mutating): (Vec<_>, Vec<_>) = tool_calls
    .iter()
    .partition(|tc| !MUTATING_TOOLS.contains(&tc.tool_name.as_str()));

// 2. Execute read-only tools in parallel
let read_only_futures = read_only.iter().map(|tc| {
    let registry = tool_registry;
    let ctx = tool_ctx;
    async move {
        let outcome = execute_tool(registry, tc, ctx).await;
        (tc.clone(), outcome)
    }
});
let read_only_results: Vec<_> = join_all(read_only_futures).await;

// 3. Execute mutating tools sequentially (preserve safety)
let mut mutating_results = Vec::new();
for tc in &mutating {
    let outcome = execute_tool(tool_registry, tc, tool_ctx).await;
    mutating_results.push(((*tc).clone(), outcome));
}

// 4. Merge results in original order
let mut all_results: Vec<_> = read_only_results.into_iter().chain(mutating_results).collect();
all_results.sort_by_key(|(tc, _)| tool_calls.iter().position(|orig| orig.id == tc.id));

// 5. Process results as before
for (tool_call, outcome) in all_results {
    // ... existing processing logic ...
}
```

### Complexity Breakdown

| Component | Lines | Difficulty | Notes |
|-----------|-------|------------|-------|
| Partition logic | ~10 | Easy | Simple `partition()` call |
| Parallel execution | ~15 | Easy | `join_all` is straightforward |
| Result ordering | ~10 | Easy | Sort by original position |
| Error handling | ~15 | Medium | Ensure partial failures don't break batch |
| Integration | ~20 | Medium | Wire into existing flow without breaking |
| Tests | ~50 | Medium | Test parallel correctness, ordering, mixed batches |
| **Total per site** | **~120** | | |

### Considerations

**1. Borrow checker challenges:**
The `ToolRegistry` and `ToolContext` need to be shared across parallel futures. Both are `&` references, which works with `join_all` since futures run on the same task. No `Arc` wrapping needed.

**2. Order preservation:**
LLMs may expect tool results in the same order as requests. The implementation must sort results back to original order before sending to the LLM.

**3. Error handling:**
If one read-only tool fails, others should still complete. `join_all` collects all results (success or error), which is the desired behavior.

**4. Auto-commit timing:**
Auto-commit runs after ALL tools complete. Parallel read-only execution doesn't change this—mutating tools still run sequentially after read-only tools finish.

**Key Files:**
- `crates/loom-cli/src/main.rs:612-656` — CLI tool execution loop (needs modification)
- `crates/loom-cli-acp/src/agent.rs:472-488` — ACP agent tool execution loop (needs modification)
- `crates/loom-common-core/src/agent.rs:50-51` — `MUTATING_TOOLS` constant (can be reused or extracted)

## Implications

1. **Low-risk change**: The modification is isolated to two execution loops. The state machine, Tool trait, and individual tools remain unchanged.

2. **Significant latency reduction for network-bound tools**: When the LLM requests multiple `web_search` or `oracle` calls, parallel execution could reduce total time from `N * latency` to `max(latency)`.

3. **Minimal benefit for filesystem tools**: `read_file` and `list_files` are already fast (local I/O), so parallelization provides marginal improvement.

4. **Consider extracting a helper**: If implementing in both CLI and ACP agent, extract a shared `execute_tools_with_parallelism()` function to avoid duplication.

5. **Feature flag candidate**: This could be gated behind a feature flag for gradual rollout and easy rollback.

## Follow-up Questions

- [ ] Should the parallel execution logic be extracted into a shared crate (e.g., `loom-cli-tools` or a new `loom-tool-executor`)?
- [ ] Should parallel execution be gated behind a feature flag or CLI option for gradual rollout?

## Related Discoveries

- [[011-read-only-tools-parallelization]] — Identifies which tools are safe to parallelize
- [[004-sequential-execution-rationale]] — Explains why sequential execution was originally chosen
- [[002-concurrent-tool-execution]] — Documents the state machine's parallel execution support
