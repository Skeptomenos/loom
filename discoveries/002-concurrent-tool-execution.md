# Discovery: How does the Agent state machine handle concurrent tool executions?

> Category: Follow-up (from Architecture)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The Agent state machine is **architecturally designed to support parallel tool execution**, but the **current CLI implementations execute tools sequentially**. The state machine uses an event-driven design where the `ExecutingTools` state tracks multiple `ToolExecutionStatus` objects in a vector, allowing the caller to execute tools in any order and report completions asynchronously. However, both `loom-cli` and `loom-cli-acp` iterate through tool calls with a simple `for` loop and `.await`, executing them one at a time.

The key insight is that the state machine is **synchronous and pure** - it receives events and returns actions, leaving the actual I/O (including parallel execution) to the caller. This separation of concerns means parallel execution could be implemented by modifying the caller to use `tokio::spawn` or `futures::future::join_all`, without any changes to the core state machine.

When all tools complete, the agent checks if any were "mutating" tools (`edit_file`, `bash`). If so, it transitions to `PostToolsHook` for auto-commit; otherwise, it goes directly back to `CallingLlm` with the tool results.

## Evidence

### State Machine Design (supports parallelism)

```rust
// crates/loom-common-core/src/state.rs:176-179
ExecutingTools {
    conversation: ConversationContext,
    executions: Vec<ToolExecutionStatus>,  // Tracks MULTIPLE concurrent executions
},
```

The spec explicitly states parallel execution is supported:
```
// specs/state-machine.md:39
| `ExecutingTools` | Running one or more tool calls in parallel | ...
```

### Current Sequential Implementation

```rust
// crates/loom-cli-acp/src/agent.rs:472-488
for call in &tool_calls {
    debug!(
        session_id = %session.session_id,
        tool_id = %call.id,
        tool_name = %call.tool_name,
        "executing tool"
    );

    let tool_result = self.execute_tool(call, &ctx).await;  // Sequential await

    session.messages.push(tool_result.clone());
    // ...
}
```

```rust
// crates/loom-cli/src/main.rs:612-620
for tool_call in &tool_calls {
    // ...
    let outcome = execute_tool(tool_registry, tool_call, tool_ctx).await;  // Sequential
    // ...
}
```

### Event-Driven Completion Tracking

The agent waits for all tools to complete before transitioning:

```rust
// crates/loom-common-core/src/agent.rs:308
let all_complete = executions.iter().all(|e| e.is_completed());
if all_complete {
    // Transition to PostToolsHook or CallingLlm
}
```

**Key Files:**
- `crates/loom-common-core/src/state.rs` - Defines `AgentState::ExecutingTools` with `Vec<ToolExecutionStatus>`
- `crates/loom-common-core/src/agent.rs` - State machine logic, handles `ToolCompleted` events
- `crates/loom-cli-acp/src/agent.rs` - ACP agent with sequential tool execution loop
- `crates/loom-cli/src/main.rs` - CLI with sequential tool execution loop
- `specs/state-machine.md` - Design spec documenting parallel execution intent

## Implications

When working in this codebase:

- **The state machine is ready for parallelism** - If you need to implement parallel tool execution, modify the caller (CLI), not the core state machine
- **Tool completion order doesn't matter** - The state machine tracks each tool by `call_id` and handles out-of-order completions correctly
- **Sequential execution is intentional for now** - Simpler debugging, predictable behavior, and avoids race conditions in file operations
- **PostToolsHook is the integration point for auto-commit** - After mutating tools complete, this hook runs before returning to the LLM

## Follow-up Questions

- [ ] Why was sequential execution chosen over parallel in the CLI implementations? Are there specific race conditions or ordering concerns with file-modifying tools?
- [ ] How does the PostToolsHook auto-commit feature work, and what determines if a commit should be made?

## Related Discoveries

- [[001-primary-purpose]] - Establishes Loom as an AI coding agent with tool execution capabilities
