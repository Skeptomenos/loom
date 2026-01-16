# Discovery: What is the expected user experience during a retry wait? Should the CLI show a spinner or countdown?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**The CLI should show a countdown timer with the retry delay, not just a spinner.** A countdown provides critical information that a spinner cannot: how long the user must wait and that progress is being made toward the next attempt. This is especially important for rate-limited scenarios where delays can be 30-60+ seconds.

The investigation reveals:

1. **The CLI currently has NO retry wait UX** — because the retry mechanism itself is not implemented (see discovery 015). When an LLM error occurs, the CLI simply logs it and returns to the prompt.

2. **The TUI already has a spinner widget** (`loom-tui-widget-spinner`) that could be adapted, but spinners alone are insufficient for retry waits because they don't communicate duration.

3. **The `indicatif` crate is already a dependency** in `loom-cli-wgtunnel`, providing a proven pattern for progress bars and spinners that could be reused.

4. **Best practice for retry UX** is a message like: `Rate limited. Retrying in 30s... [=====>     ] 15s remaining` — combining a countdown, progress bar, and clear explanation.

## Evidence

**CLI has no retry handling** (`crates/loom-cli/src/main.rs:579-581`):
```rust
LlmEvent::Error(e) => {
    error!(error = ?e, "LLM stream error");
    // No retry scheduling, no user feedback - just logs and continues
}
```

**TUI spinner exists but is for "Thinking..." state** (`crates/loom-tui-app/src/app.rs:266-276`):
```rust
if self.state.is_loading {
    let spinner = Spinner::new()
        .label("Thinking...")  // Generic loading, not retry-specific
        .text_style(self.theme.text.normal);
    // ...
    frame.render_stateful_widget(spinner, spinner_area, &mut self.state.spinner_state);
}
```

**Spinner widget is simple and lacks countdown** (`crates/loom-tui-widget-spinner/src/lib.rs:103-124`):
```rust
impl StatefulWidget for Spinner {
    fn render(self, area: Rect, buf: &mut Buffer, state: &mut Self::State) {
        let frames = self.kind.frames();
        let frame_char = frames[state.frame % frames.len()];
        let text = match &self.label {
            Some(label) => format!("{} {}", frame_char, label),
            None => frame_char.to_string(),
        };
        buf.set_string(x, y, &text, self.text_style);
    }
}
```

**indicatif is already a dependency** (`crates/loom-cli-wgtunnel/Cargo.toml`):
```toml
indicatif = "0.17"
```

**Retry-After information is available** (`crates/loom-common-core/src/error.rs:46-47`):
```rust
#[error("Rate limited: retry after {retry_after_secs:?} seconds")]
RateLimited { retry_after_secs: Option<u64> },
```

**Key Files:**
- `crates/loom-cli/src/main.rs` — CLI REPL with no retry UX
- `crates/loom-tui-widget-spinner/src/lib.rs` — Existing spinner widget (no countdown)
- `crates/loom-cli-wgtunnel/Cargo.toml` — indicatif dependency already present
- `crates/loom-common-core/src/error.rs` — `RateLimited` error with `retry_after_secs`
- `crates/loom-common-http/src/retry.rs` — Backoff calculation logic

## Recommended UX Patterns

### Option A: Countdown with Progress Bar (Recommended)

Best for longer waits (>5 seconds). Uses `indicatif` for cross-platform terminal handling:

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::time::Duration;

async fn wait_for_retry(delay_secs: u64) {
    let pb = ProgressBar::new(delay_secs);
    pb.set_style(ProgressStyle::default_bar()
        .template("{msg} [{bar:40.cyan/blue}] {pos}s remaining")
        .unwrap()
        .progress_chars("=>-"));
    pb.set_message("Rate limited. Retrying in");
    
    for remaining in (0..delay_secs).rev() {
        pb.set_position(remaining);
        tokio::time::sleep(Duration::from_secs(1)).await;
    }
    pb.finish_with_message("Retrying now...");
}
```

Output: `Rate limited. Retrying in [========>           ] 15s remaining`

### Option B: Simple Countdown (Minimal)

For shorter waits or simpler implementation:

```rust
async fn wait_for_retry(delay_secs: u64) {
    for remaining in (1..=delay_secs).rev() {
        eprint!("\rRate limited. Retrying in {}s...  ", remaining);
        tokio::time::sleep(Duration::from_secs(1)).await;
    }
    eprintln!("\rRetrying now...                    ");
}
```

Output: `Rate limited. Retrying in 15s...` (updates in place)

### Option C: Spinner with Message (Not Recommended for Retries)

Spinners are appropriate for indeterminate waits (like "Thinking...") but NOT for retry waits where the duration is known:

```rust
// DON'T DO THIS for retries - user has no idea how long to wait
let spinner = Spinner::new().label("Retrying...");
```

## Implications

1. **Countdown is essential for retry UX**: Users need to know how long they're waiting. A spinner alone creates anxiety and uncertainty.

2. **indicatif is the right tool**: Already a dependency, battle-tested, handles terminal quirks (cursor hiding, line clearing, etc.).

3. **The retry delay should come from `Retry-After`**: When the server provides a `Retry-After` header (see discovery 018), use that value. Otherwise, use the calculated exponential backoff.

4. **Cancellation should be supported**: Users should be able to press Ctrl+C during the countdown to abort the retry and return to the prompt.

5. **The TUI needs a different approach**: The TUI can't use `indicatif` (it uses ratatui). A new `CountdownWidget` or enhanced `Spinner` with countdown support would be needed.

## Follow-up Questions

- [ ] Should the CLI support cancelling a retry wait with Ctrl+C, or should it always complete the countdown?
- [ ] Should the TUI implement a dedicated `CountdownWidget` or extend the existing `Spinner` to support countdown mode?

## Related Discoveries

- [[015-cli-acp-retry-timer-intentionality]] — Confirms retry mechanism is not implemented
- [[016-shared-agent-runtime-analysis]] — Proposed AgentRuntime would encapsulate retry UX
- [[018-retry-after-header-addition]] — Server-side Retry-After header for accurate delays
- [[008-retry-timeout-mechanism]] — How the Agent state machine expects retries to work
