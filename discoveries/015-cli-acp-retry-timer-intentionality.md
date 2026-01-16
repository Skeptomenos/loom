# Discovery: Should the CLI and ACP agent be updated to implement the retry timer, or is the current behavior (immediate failure/logging) intentional?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**The current behavior is a known gap, not an intentional design choice.** The Agent state machine's retry mechanism was designed with the expectation that consumers (CLI, ACP, TUI) would implement the timer, but this implementation work was never completed. The evidence strongly suggests this is technical debt rather than a deliberate decision:

1. **The spec explicitly documents the expected behavior**: `specs/error-handling.md:253-255` states "External systems (runtime/scheduler) fire `AgentEvent::RetryTimeoutFired` after a backoff delay" — this is prescriptive, not descriptive of an optional feature.

2. **The state machine is fully implemented and tested**: The `Agent` in `loom-common-core` has complete retry logic with 4 unit tests specifically for `RetryTimeoutFired` handling. This represents significant engineering investment that would be wasted if the feature was intentionally disabled.

3. **The backoff logic exists and is reusable**: `loom-common-http/src/retry.rs` provides `calculate_delay()` with exponential backoff and jitter — the exact logic needed for agent-level retries. This was clearly designed for reuse.

4. **No explicit "disabled" or "deferred" markers**: Unlike rate limiting (explicitly marked "deferred from v1" in TODO.md), there are no comments, TODOs, or documentation indicating the retry timer was intentionally omitted.

5. **The behavior differs between consumers in a way that suggests incompleteness**:
   - CLI: Logs error and returns to prompt (user can manually retry)
   - ACP: Returns error immediately (IDE must handle)
   - Neither implements automatic retry

**Recommendation**: The CLI and ACP should be updated to implement the retry timer. This would:
- Improve reliability for transient LLM failures (rate limits, network blips)
- Utilize the already-implemented state machine logic
- Provide consistent behavior across all consumers

## Evidence

**Spec prescribes external timer** (`specs/error-handling.md:253-255`):
```markdown
### RetryTimeoutFired Event

External systems (runtime/scheduler) fire `AgentEvent::RetryTimeoutFired` after a backoff delay
```

**Agent state machine is fully implemented** (`crates/loom-common-core/src/agent.rs:255-284`):
```rust
// Error + RetryTimeoutFired -> CallingLlm (retry)
(
    AgentState::Error {
        conversation,
        retries,
        origin: ErrorOrigin::Llm,
        ..
    },
    AgentEvent::RetryTimeoutFired,
) => {
    let request = LlmRequest { ... };
    self.state = AgentState::CallingLlm {
        conversation: conv,
        retries: *retries,
    };
    AgentAction::SendLlmRequest(request)
}
```

**Four dedicated unit tests exist** (`crates/loom-common-core/src/agent.rs:775-1100`):
- `test_retry_timeout_fired_in_error_state_transitions_to_calling_llm`
- `test_retry_timeout_fired_preserves_retry_count`
- `test_max_retries_exceeded_displays_error`
- `test_retry_timeout_fired_with_tool_error_origin_is_ignored`

**Backoff logic is ready for reuse** (`crates/loom-common-http/src/retry.rs:66-78`):
```rust
fn calculate_delay(cfg: &RetryConfig, attempt: u32) -> Duration {
    let exponential_delay = cfg.base_delay.as_secs_f64() * cfg.backoff_factor.powi(attempt as i32);
    let capped_delay = exponential_delay.min(cfg.max_delay.as_secs_f64());
    let final_delay = if cfg.jitter {
        let jitter_factor = 0.5 + fastrand::f64();
        capped_delay * jitter_factor
    } else {
        capped_delay
    };
    Duration::from_secs_f64(final_delay)
}
```

**CLI does not implement timer** (`crates/loom-cli/src/main.rs:579-581`):
```rust
LlmEvent::Error(e) => {
    error!(error = ?e, "LLM stream error");
    // No retry scheduling - just logs and continues
}
```

**ACP returns error immediately** (`crates/loom-cli-acp/src/agent.rs:436-438`):
```rust
LlmEvent::Error(e) => {
    error!(error = ?e, "LLM stream error");
    return Err(AcpError::Llm(e));  // No retry - immediate failure
}
```

**Rate limiting explicitly deferred, retry timer not mentioned** (`TODO.md:279`):
```markdown
5. **Rate Limiting** - Add per-IP/per-user rate limits (deferred from v1)
```

**Key Files:**
- `crates/loom-common-core/src/agent.rs` — Agent state machine with complete retry logic
- `crates/loom-common-core/src/state.rs` — `AgentEvent::RetryTimeoutFired` definition
- `crates/loom-common-http/src/retry.rs` — Reusable exponential backoff with jitter
- `crates/loom-cli/src/main.rs` — CLI REPL (does not implement timer)
- `crates/loom-cli-acp/src/agent.rs` — ACP agent (does not implement timer)
- `specs/error-handling.md` — Specification documenting expected behavior
- `specs/state-machine.md:134` — State transition table showing Error → CallingLlm on RetryTimeoutFired

## Implications

- **The retry mechanism is currently dead code**: Despite being fully implemented and tested in the state machine, no production consumer fires `RetryTimeoutFired`. This means transient LLM errors cause immediate failure (ACP) or require manual retry (CLI).

- **Implementation is straightforward**: Adding retry support requires:
  1. Detecting when `AgentAction::WaitForInput` is returned after an LLM error
  2. Calculating delay using `loom_common_http::retry::calculate_delay()`
  3. Sleeping for the delay duration
  4. Firing `AgentEvent::RetryTimeoutFired`

- **A shared "Agent Runtime" abstraction would reduce duplication**: Rather than implementing the timer in both CLI and ACP, a shared runtime could encapsulate the retry loop, making it easier to add to future consumers (TUI, web workers, etc.).

- **The `Retry-After` header enhancement (from discovery 009) would complement this**: If the server returns `Retry-After` and the client parses it, the retry timer could use provider-suggested delays instead of generic exponential backoff.

## Follow-up Questions

- [ ] Would a shared "Agent runtime" abstraction be useful to encapsulate the retry timer logic for reuse across CLI, ACP, and future consumers?
- [ ] What is the expected user experience during a retry wait? Should the CLI show a spinner or countdown?

## Related Discoveries

- [[008-retry-timeout-mechanism]] — Documents the passive state machine design
- [[009-proxy-llm-client-retry-after]] — Retry-After header parsing enhancement
- [[007-sse-error-handling]] — How SSE errors propagate to the Agent
- [[013-proxy-llm-client-503-handling]] — How 503 errors are currently handled
