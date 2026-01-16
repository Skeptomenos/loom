# Discovery: Should the parallel execution logic be extracted into a shared crate?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, the parallel execution logic should be extracted into the existing `loom-cli-tools` crate rather than creating a new crate.** The `loom-cli-tools` crate is already the shared abstraction layer for tool execution, containing the `Tool` trait and `ToolRegistry`. Adding a `ToolExecutor` abstraction here would be a natural extension that maintains the existing architecture.

The current codebase has **duplicated sequential execution logic** in two places:
1. `loom-cli/src/main.rs` (lines 612-656) — iterates over tool calls with `for tool_call in &tool_calls`
2. `loom-cli-acp/src/agent.rs` (lines 472-488) — identical pattern with `for call in &tool_calls`

Both implementations:
- Loop sequentially over tool calls
- Call `execute_tool()` or `self.execute_tool()` with await
- Collect results and push to messages
- Have no parallel execution capability

Extracting this into a shared `ToolExecutor` in `loom-cli-tools` would:
1. **Eliminate duplication** — Single implementation for both CLI and ACP
2. **Enable parallel execution** — Centralized place to implement read-only tool parallelization
3. **Improve testability** — Executor logic can be unit tested independently
4. **Follow existing patterns** — Matches how `loom-common-core` provides shared types used by 12+ crates

## Evidence

**Current Tool Execution in CLI (`loom-cli/src/main.rs:612-656`):**
```rust
for tool_call in &tool_calls {
    let outcome = execute_tool(tool_registry, tool_call, tool_ctx).await;
    // ... process outcome, push to messages ...
}
```

**Current Tool Execution in ACP (`loom-cli-acp/src/agent.rs:472-488`):**
```rust
for call in &tool_calls {
    let tool_result = self.execute_tool(call, &ctx).await;
    session.messages.push(tool_result.clone());
    // ...
}
```

**Existing `loom-cli-tools` Structure:**
```
crates/loom-cli-tools/src/
├── lib.rs          # Public exports
├── registry.rs     # Tool trait + ToolRegistry
├── bash.rs         # BashTool implementation
├── edit_file.rs    # EditFileTool implementation
├── read_file.rs    # ReadFileTool implementation
├── list_files.rs   # ListFilesTool implementation
├── oracle.rs       # OracleTool implementation
└── web_search.rs   # WebSearchTool implementations
```

**Crate Dependencies (both CLI and ACP already depend on loom-cli-tools):**
- `loom-cli/Cargo.toml:41` — `loom-cli-tools = { path = "../loom-cli-tools" }`
- `loom-cli-acp/Cargo.toml:17` — `loom-cli-tools = { path = "../loom-cli-tools" }`

**Key Files:**
- `crates/loom-cli-tools/src/registry.rs` — Contains `Tool` trait and `ToolRegistry`, natural home for `ToolExecutor`
- `crates/loom-cli/src/main.rs` — CLI's `execute_tool` function (lines 397-444) and execution loop
- `crates/loom-cli-acp/src/agent.rs` — ACP's `execute_tool` method and execution loop

## Implications

### Recommended Architecture

Add a new `executor.rs` module to `loom-cli-tools`:

```rust
// crates/loom-cli-tools/src/executor.rs

pub struct ToolExecutor<'a> {
    registry: &'a ToolRegistry,
}

impl<'a> ToolExecutor<'a> {
    pub fn new(registry: &'a ToolRegistry) -> Self {
        Self { registry }
    }

    /// Execute tool calls with optional parallelization for read-only tools.
    pub async fn execute_batch(
        &self,
        tool_calls: &[ToolCall],
        ctx: &ToolContext,
    ) -> Vec<ToolExecutionOutcome> {
        // 1. Partition into read-only vs mutating
        let (read_only, mutating): (Vec<_>, Vec<_>) = tool_calls
            .iter()
            .partition(|tc| self.is_read_only(&tc.tool_name));
        
        // 2. Execute read-only in parallel
        let read_only_results = futures::future::join_all(
            read_only.iter().map(|tc| self.execute_one(tc, ctx))
        ).await;
        
        // 3. Execute mutating sequentially
        let mut mutating_results = Vec::new();
        for tc in mutating {
            mutating_results.push(self.execute_one(tc, ctx).await);
        }
        
        // 4. Merge and sort by original order
        self.merge_results(tool_calls, read_only_results, mutating_results)
    }

    fn is_read_only(&self, tool_name: &str) -> bool {
        self.registry.get(tool_name)
            .map(|t| !t.is_mutating())
            .unwrap_or(false)
    }
}
```

### Why NOT a New Crate (`loom-tool-executor`)?

1. **Unnecessary fragmentation** — `loom-cli-tools` already owns the tool abstraction
2. **Circular dependency risk** — A new crate would need `ToolRegistry` from `loom-cli-tools`
3. **Minimal scope** — The executor is ~100-120 lines, not enough to justify a new crate
4. **Existing pattern** — The codebase extends existing crates rather than creating new ones for small features

### Migration Path

1. Add `is_mutating()` method to `Tool` trait (as proposed in discovery-020)
2. Add `executor.rs` module to `loom-cli-tools`
3. Update `loom-cli` to use `ToolExecutor::execute_batch()`
4. Update `loom-cli-acp` to use `ToolExecutor::execute_batch()`
5. Remove duplicated `execute_tool` functions from both crates

### Estimated Effort

| Task | Lines | Complexity |
|------|-------|------------|
| Add `is_mutating()` to Tool trait | ~10 | Low |
| Implement `ToolExecutor` | ~100-120 | Medium |
| Update CLI to use executor | ~30 | Low |
| Update ACP to use executor | ~30 | Low |
| Tests for parallel execution | ~100 | Medium |
| **Total** | **~270-290** | **Medium** |

## Follow-up Questions

- [ ] Should the `ToolExecutor` support configurable concurrency limits (e.g., max 4 parallel read-only tools) to prevent resource exhaustion?
- [ ] Should execution metrics (timing, success/failure counts) be collected at the executor level for observability?

## Related Discoveries

- [[discovery-011]] — Read-only tools parallelization (identifies which tools are safe)
- [[discovery-019]] — Parallel execution implementation complexity (estimates effort)
- [[discovery-020]] — Tool trait `is_mutating()` method (prerequisite for executor)
