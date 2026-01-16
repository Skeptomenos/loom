# Discovery: How does the state machine orchestrate conversation flow and tool execution?

> Category: Architecture (Follow-up)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The Loom agent uses an **explicit, event-driven state machine** implemented in `loom-common-core` to manage conversation flow and tool execution. The design follows an **inversion of control** pattern: the state machine receives `AgentEvent`s and returns `AgentAction`s that the caller must execute. This keeps the state machine synchronous and pure while allowing the caller to manage async operations (LLM calls, tool execution).

The state machine has 7 states that form a well-defined lifecycle:
1. **WaitingForUserInput** - Idle, ready for user messages
2. **CallingLlm** - Making a request to the LLM provider (with retry counter)
3. **ProcessingLlmResponse** - Transient state for examining LLM response
4. **ExecutingTools** - Running one or more tool calls in parallel
5. **PostToolsHook** - Running post-tool hooks (e.g., auto-commit after file mutations)
6. **Error** - Recoverable error with retry capability
7. **ShuttingDown** - Terminal state

## Evidence

**Key Files:**
- `crates/loom-common-core/src/state.rs` — State and event type definitions
- `crates/loom-common-core/src/agent.rs` — State machine implementation (`handle_event`)
- `specs/state-machine.md` — Comprehensive design documentation

**State Definition (state.rs:164-193):**
```rust
pub enum AgentState {
    WaitingForUserInput { conversation: ConversationContext },
    CallingLlm { conversation: ConversationContext, retries: u32 },
    ProcessingLlmResponse { conversation: ConversationContext, response: LlmResponse },
    ExecutingTools { conversation: ConversationContext, executions: Vec<ToolExecutionStatus> },
    PostToolsHook { conversation: ConversationContext, pending_llm_request: LlmRequest, completed_tools: Vec<CompletedToolInfo> },
    Error { conversation: ConversationContext, error: AgentError, retries: u32, origin: ErrorOrigin },
    ShuttingDown,
}
```

**Event Types (state.rs:128-143):**
```rust
pub enum AgentEvent {
    UserInput(Message),
    LlmEvent(LlmEvent),
    ToolProgress(ToolProgressEvent),
    ToolCompleted { call_id: String, outcome: ToolExecutionOutcome },
    PostToolsHookCompleted { action_taken: bool },
    RetryTimeoutFired,
    ShutdownRequested,
}
```

**Action Types (agent.rs:28-46):**
```rust
pub enum AgentAction {
    SendLlmRequest(LlmRequest),
    ExecuteTools(Vec<ToolCall>),
    RunPostToolsHook { completed_tools: Vec<CompletedToolInfo> },
    WaitForInput,
    DisplayMessage(String),
    DisplayError(String),
    Shutdown,
}
```

**Key Transition Logic (agent.rs:161):**
```rust
pub fn handle_event(&mut self, event: AgentEvent) -> AgentResult<AgentAction>
```

**Mutating Tool Detection (agent.rs:48-63):**
The state machine tracks which tools are "mutating" (`edit_file`, `bash`) to trigger post-tool hooks like auto-commit:
```rust
const MUTATING_TOOLS: &[&str] = &["edit_file", "bash"];
```

## Implications

- **Testability**: Every state and transition can be unit tested in isolation. Property-based tests verify invariants.
- **Determinism**: Given the same sequence of events, the state machine produces the same sequence of actions.
- **Separation of Concerns**: The state machine decides _what_ to do; the caller decides _how_ to do it (async, parallel, etc.).
- **No Hidden State**: All context is carried explicitly in state variants. Each state carries its own `ConversationContext`.
- **Exhaustive Matching**: Rust's `match` ensures all state/event combinations are handled.
- **Post-Tool Hooks**: The `PostToolsHook` state enables features like auto-commit after file-modifying tools.

## Follow-up Questions

- [ ] How does the tool execution system work? (ToolExecutionStatus lifecycle, parallel execution)
- [ ] How is the retry mechanism implemented for LLM errors?

## Related Discoveries

- [[001-primary-purpose]] — Establishes the three core principles (modularity, extensibility, reliability)
