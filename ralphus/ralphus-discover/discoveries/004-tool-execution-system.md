# Discovery: How does the tool execution system work?

> Category: Architecture (Follow-up)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The tool system enables the Loom agent to interact with the filesystem and execute commands. It follows a **registry pattern** with async execution and a **discriminated union** for tracking execution state.

**Key Components:**
1. **Tool trait** - Interface all tools implement (`name`, `description`, `input_schema`, `invoke`)
2. **ToolRegistry** - HashMap-based registration and dispatch
3. **ToolExecutionStatus** - State machine for tracking execution (Pending -> Running -> Completed)
4. **ToolContext** - Provides workspace root as security boundary

**Execution Flow:**
1. LLM returns `ToolCall` in response
2. Agent transitions to `ExecutingTools` state with `Vec<ToolExecutionStatus>`
3. Each tool starts as `Pending`, moves to `Running`, then `Completed`
4. Multiple tools can execute in parallel (tracked independently)
5. Results are wrapped in `ToolExecutionOutcome` (Success or Error)
6. Tool outputs become `Message::tool` and are sent back to LLM

**Built-in Tools:** `read_file`, `list_files`, `edit_file`, `bash`, `oracle`, `web_search`

## Evidence

**ToolExecutionStatus lifecycle (state.rs:38-58):**
```rust
pub enum ToolExecutionStatus {
    Pending {
        call_id: String,
        tool_name: String,
        requested_at: Instant,
    },
    Running {
        call_id: String,
        tool_name: String,
        started_at: Instant,
        last_update_at: Instant,
        progress: Option<ToolProgress>,
    },
    Completed {
        call_id: String,
        tool_name: String,
        started_at: Instant,
        completed_at: Instant,
        outcome: ToolExecutionOutcome,
    },
}
```

**Tool trait (specs/tool-system.md):**
```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn input_schema(&self) -> serde_json::Value;
    fn to_definition(&self) -> ToolDefinition;
    async fn invoke(
        &self,
        args: serde_json::Value,
        ctx: &ToolContext,
    ) -> Result<serde_json::Value, ToolError>;
}
```

**ToolRegistry (specs/tool-system.md):**
```rust
pub struct ToolRegistry {
    tools: HashMap<String, Box<dyn Tool>>,
}

impl ToolRegistry {
    fn register(&mut self, tool: Box<dyn Tool>);
    fn get(&self, name: &str) -> Option<&dyn Tool>;
    fn definitions(&self) -> Vec<ToolDefinition>;
}
```

**Key Files:**
- `crates/loom-common-core/src/state.rs` — ToolExecutionStatus, ToolProgress, ToolExecutionOutcome
- `crates/loom-common-core/src/tool.rs` — ToolDefinition, ToolContext
- `crates/loom-cli-tools/src/registry.rs` — ToolRegistry, Tool trait
- `crates/loom-cli-tools/src/*.rs` — Individual tool implementations
- `specs/tool-system.md` — Comprehensive design documentation

## Implications

- **Adding new tools**: Implement `Tool` trait, register in `ToolRegistry`, export from `lib.rs`
- **Security**: All paths validated against `workspace_root` to prevent traversal attacks
- **Async execution**: Tools use `tokio::fs` for non-blocking I/O
- **Progress reporting**: Long-running tools can report `ToolProgress` (fraction, message, units)
- **Snippet-based editing**: `edit_file` uses `old_str -> new_str` pattern (LLMs struggle with line numbers)
- **Mutating tools**: `edit_file` and `bash` trigger `PostToolsHook` for auto-commit

## Follow-up Questions

- [ ] How does path validation prevent traversal attacks? (canonicalization, prefix checking)
- [ ] How does the bash tool handle timeouts and output truncation?

## Related Discoveries

- [[002-state-machine-orchestration]] — ExecutingTools state and ToolCompleted events
- [[003-crate-relationships]] — Tools execute in CLI, not server
