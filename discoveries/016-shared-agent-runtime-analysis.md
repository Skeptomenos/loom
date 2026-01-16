# Discovery: Would a shared "Agent runtime" abstraction be useful to encapsulate the retry timer logic for reuse across CLI, ACP, and future consumers?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, a shared Agent runtime abstraction would be highly valuable.** The investigation reveals that the `loom-common-core::Agent` state machine is a well-designed, fully-tested abstraction that is currently **underutilized** by production code. Both the CLI REPL and ACP agent implement their own manual event loops, resulting in:

1. **Duplicated logic** for LLM streaming, tool execution, and message management
2. **Dead code** in the state machine (the retry mechanism via `RetryTimeoutFired` is never triggered)
3. **Missing features** in some consumers (ACP lacks auto-commit support that CLI has)
4. **Inconsistent behavior** across frontends

A shared `AgentRuntime` would:
- Drive the `Agent` state machine via `handle_event()` / `AgentAction` pattern
- Encapsulate the async loop for LLM streaming and tool execution
- Implement the retry timer logic (exponential backoff → `RetryTimeoutFired`)
- Provide hooks for UI integration (progress callbacks, cancellation)
- Ensure consistent behavior across CLI, ACP, TUI, and future consumers

## Evidence

**The Agent state machine is designed for external driving** (`specs/state-machine.md:16-20`):
```markdown
The state machine receives `AgentEvent`s and returns `AgentAction`s that the caller must execute.
This inversion of control allows the caller to manage async operations (LLM calls, tool execution)
while the state machine remains synchronous and pure.
```

**CLI implements manual loop instead of using Agent** (`crates/loom-cli/src/main.rs:457-702`):
```rust
async fn run_repl(...) -> Result<()> {
    loop {
        // Manual stdin handling
        // Manual LLM streaming
        // Manual tool execution
        // Manual auto-commit
    }
}
```

**ACP implements nearly identical manual loop** (`crates/loom-cli-acp/src/agent.rs:369-491`):
```rust
async fn run_prompt_loop(&self, session: &mut SessionState) -> Result<StopReason, AcpError> {
    loop {
        // Same pattern: LLM call → stream → tools → repeat
    }
}
```

**Retry mechanism is fully implemented but unused** (`crates/loom-common-core/src/agent.rs:255-284`):
```rust
// Error + RetryTimeoutFired -> CallingLlm (retry)
(
    AgentState::Error { conversation, retries, origin: ErrorOrigin::Llm, .. },
    AgentEvent::RetryTimeoutFired,
) => {
    // Retry logic is here, but no runtime fires this event
    AgentAction::SendLlmRequest(request)
}
```

**Backoff logic exists and is reusable** (`crates/loom-common-http/src/retry.rs:66-78`):
```rust
fn calculate_delay(cfg: &RetryConfig, attempt: u32) -> Duration {
    let exponential_delay = cfg.base_delay.as_secs_f64() * cfg.backoff_factor.powi(attempt as i32);
    // ... jitter logic
}
```

**Key Files:**
- `crates/loom-common-core/src/agent.rs` — The passive Agent state machine
- `crates/loom-common-core/src/state.rs` — `AgentState`, `AgentEvent`, `AgentAction` definitions
- `crates/loom-cli/src/main.rs` — CLI REPL with manual loop (lines 457-702)
- `crates/loom-cli-acp/src/agent.rs` — ACP agent with manual loop (lines 369-491)
- `crates/loom-common-http/src/retry.rs` — Reusable exponential backoff logic
- `crates/loom-cli-auto-commit/src/service.rs` — Post-tool hook that should be integrated
- `specs/state-machine.md` — Specification documenting the intended design

## Proposed Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        loom-agent-runtime                        │
├─────────────────────────────────────────────────────────────────┤
│  AgentRuntime<L: LlmClient, T: ToolExecutor>                    │
│  ├── agent: Agent (from loom-common-core)                       │
│  ├── llm_client: L                                              │
│  ├── tool_executor: T                                           │
│  ├── retry_config: RetryConfig                                  │
│  └── hooks: Vec<Box<dyn PostToolHook>>                          │
│                                                                  │
│  Methods:                                                        │
│  ├── async fn run(&mut self, input: Message) -> Result<()>      │
│  ├── async fn run_with_callbacks(&mut self, ..., cb: Callbacks) │
│  └── fn cancel(&mut self)                                        │
│                                                                  │
│  Internal Loop:                                                  │
│  1. agent.handle_event(UserInput) → SendLlmRequest              │
│  2. Stream LLM response, feed TextDelta/Completed events        │
│  3. On Error → schedule retry timer → fire RetryTimeoutFired    │
│  4. On ExecuteTools → run tools (parallel or sequential)        │
│  5. On RunPostToolsHook → run hooks (auto-commit, etc.)         │
│  6. Repeat until WaitForInput                                    │
└─────────────────────────────────────────────────────────────────┘
           │                    │                    │
           ▼                    ▼                    ▼
    ┌──────────┐         ┌──────────┐         ┌──────────┐
    │ loom-cli │         │loom-cli- │         │ loom-tui │
    │  (REPL)  │         │   acp    │         │  (TUI)   │
    └──────────┘         └──────────┘         └──────────┘
```

## Implementation Complexity

| Component | Effort | Notes |
|-----------|--------|-------|
| Create `loom-agent-runtime` crate | Low | New crate with clear boundaries |
| Extract common loop logic | Medium | Generalize from CLI/ACP patterns |
| Implement retry timer | Low | Reuse `loom-common-http::calculate_delay` |
| Add callback/progress hooks | Medium | For UI integration (spinners, etc.) |
| Refactor CLI to use runtime | Medium | Replace `run_repl` internals |
| Refactor ACP to use runtime | Medium | Replace `run_prompt_loop` internals |
| Add parallel tool execution | High | Optional enhancement |

**Total estimated effort**: 2-3 days for core runtime + 1-2 days per consumer refactor

## Benefits

1. **DRY**: Eliminate ~300 lines of duplicated loop logic between CLI and ACP
2. **Correctness**: Enable the retry mechanism that's currently dead code
3. **Consistency**: Ensure all consumers behave identically for LLM errors, tool execution, etc.
4. **Extensibility**: New consumers (TUI, web workers, etc.) get full functionality for free
5. **Testability**: The runtime can be tested in isolation with mock LLM/tools
6. **Feature parity**: ACP would automatically gain auto-commit support

## Implications

- **The current architecture is intentional but incomplete**: The `Agent` state machine was designed for external driving, but the "driver" (runtime) was never extracted into a shared component.

- **This is not a refactor for its own sake**: The retry mechanism is genuinely broken in production. Users experiencing transient LLM errors (rate limits, network blips) get immediate failures instead of automatic retries.

- **The TUI would benefit most**: The TUI (`loom-tui-app`) currently has no LLM integration at all — it's a UI shell. A shared runtime would make adding agent functionality trivial.

- **Parallel tool execution becomes feasible**: With a centralized runtime, implementing parallel execution for read-only tools (discovery 011) becomes a single-point change.

## Follow-up Questions

- [ ] What trait bounds should `AgentRuntime` require for maximum flexibility while maintaining type safety?
- [ ] Should the runtime be generic over the progress/callback mechanism, or use a fixed trait like `AgentRuntimeCallbacks`?

## Related Discoveries

- [[008-retry-timeout-mechanism]] — Documents the passive state machine design
- [[015-cli-acp-retry-timer-intentionality]] — Confirms retry timer is missing, not intentionally disabled
- [[011-read-only-tools-parallelization]] — Parallel execution would be easier with shared runtime
- [[005-post-tools-hook-auto-commit]] — Auto-commit hook that should be integrated into runtime
