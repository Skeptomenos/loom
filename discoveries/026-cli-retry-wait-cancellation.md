# Discovery: Should the CLI support cancelling a retry wait with Ctrl+C, or should it always complete the countdown?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, the CLI should absolutely support cancelling a retry wait with Ctrl+C.** This is both a UX best practice and consistent with existing patterns in the Loom codebase. Users should never be forced to wait through a countdown they don't want—especially for potentially long rate-limit delays (30-60+ seconds).

The investigation reveals:

1. **Existing precedent in the codebase**: The `auth.rs` device code flow already implements exactly this pattern—a `tokio::select!` that races `tokio::signal::ctrl_c()` against `tokio::time::sleep()`, allowing immediate cancellation.

2. **Global shutdown mechanism exists**: The CLI already has a `shutdown_tx/shutdown_rx` watch channel for graceful shutdown. A retry countdown could listen to this same channel.

3. **User expectation**: Ctrl+C universally means "stop what you're doing." Ignoring it during a countdown would violate the principle of least surprise and frustrate users.

4. **Two cancellation behaviors are valid**:
   - **Abort retry entirely**: Return to prompt, let user decide next action (recommended default)
   - **Skip countdown, retry immediately**: For impatient users who want to try anyway

## Evidence

**Auth flow already implements cancellable wait** (`crates/loom-cli/src/auth.rs:123-129`):
```rust
tokio::select! {
    _ = tokio::signal::ctrl_c() => {
        eprintln!("\n{}", loom_common_i18n::t(get_locale(), "client.auth.cancelled"));
        return Err(anyhow!("login cancelled by user"));
    }
    _ = tokio::time::sleep(poll_interval) => {}
}
```

**Tunnel command uses same pattern** (`crates/loom-cli-wgtunnel/src/commands/tunnel.rs:109-114`):
```rust
tokio::select! {
    _ = tokio::signal::ctrl_c() => {
        println!("\n{} Shutting down...", style("→").yellow());
    }
    _ = manager.wait() => {}
}
```

**Global shutdown channel exists** (`crates/loom-cli/src/main.rs:754-755, 1409-1418`):
```rust
let (shutdown_tx, shutdown_rx) = watch::channel(false);
setup_ctrlc_handler(shutdown_tx)?;

fn setup_ctrlc_handler(shutdown_tx: watch::Sender<bool>) -> Result<()> {
    ctrlc::set_handler(move || {
        info!("received Ctrl+C, requesting shutdown");
        let _ = shutdown_tx.send(true);
        eprintln!();
    })
    .context("failed to set Ctrl+C handler")?;
    Ok(())
}
```

**REPL already monitors shutdown** (`crates/loom-cli/src/main.rs:500-511`):
```rust
tokio::select! {
    biased;
    _ = shutdown_rx.changed() => {
        if *shutdown_rx.borrow() {
            info!("shutdown requested, saving thread");
            // ... graceful shutdown
        }
    }
    // ... other branches
}
```

**i18n strings exist for cancellation** (`crates/loom-common-i18n/locales/en/messages.po`):
```
msgid "client.auth.cancelled"
msgstr "Login cancelled"

msgid "client.repl.interrupted"
msgstr "..." # Already used for Ctrl+C in REPL
```

**Key Files:**
- `crates/loom-cli/src/auth.rs` — Best example of cancellable wait pattern
- `crates/loom-cli/src/main.rs` — Global shutdown mechanism
- `crates/loom-cli-wgtunnel/src/commands/tunnel.rs` — Another cancellable wait example
- `crates/loom-server-jobs/src/context.rs` — Server-side CancellationToken (AtomicBool)

## Recommended Implementation

### Option A: Abort Retry (Recommended Default)

When Ctrl+C is pressed during countdown, abort the retry and return to prompt:

```rust
use indicatif::{ProgressBar, ProgressStyle};
use std::time::Duration;

async fn wait_for_retry(
    delay_secs: u64,
    shutdown_rx: &mut watch::Receiver<bool>,
) -> Result<(), RetryAborted> {
    let pb = ProgressBar::new(delay_secs);
    pb.set_style(ProgressStyle::default_bar()
        .template("{msg} [{bar:40.cyan/blue}] {pos}s remaining")
        .unwrap()
        .progress_chars("=>-"));
    pb.set_message("Rate limited. Retrying in");
    
    for remaining in (1..=delay_secs).rev() {
        pb.set_position(remaining);
        
        tokio::select! {
            biased;
            
            _ = shutdown_rx.changed() => {
                if *shutdown_rx.borrow() {
                    pb.finish_and_clear();
                    eprintln!("{}", t(get_locale(), "client.retry.cancelled"));
                    return Err(RetryAborted::UserCancelled);
                }
            }
            
            _ = tokio::signal::ctrl_c() => {
                pb.finish_and_clear();
                eprintln!("{}", t(get_locale(), "client.retry.cancelled"));
                return Err(RetryAborted::UserCancelled);
            }
            
            _ = tokio::time::sleep(Duration::from_secs(1)) => {}
        }
    }
    
    pb.finish_with_message("Retrying now...");
    Ok(())
}

#[derive(Debug)]
enum RetryAborted {
    UserCancelled,
}
```

### Option B: Skip Countdown, Retry Immediately

For users who want to retry immediately despite rate limiting:

```rust
// Same as above, but on Ctrl+C:
_ = tokio::signal::ctrl_c() => {
    pb.finish_and_clear();
    eprintln!("{}", t(get_locale(), "client.retry.skipped"));
    return Ok(()); // Continue with retry immediately
}
```

### Recommendation

**Use Option A (abort) as default** because:
1. Rate limits exist for a reason—retrying immediately will likely fail again
2. User might want to do something else (check logs, modify request, etc.)
3. Consistent with auth flow behavior (Ctrl+C = cancel operation)

If needed, a `--force-retry` flag could enable Option B behavior.

## Implications

1. **Retry countdown must use `tokio::select!`**: Any countdown implementation must race against the shutdown signal or `ctrl_c()`.

2. **Consider using `shutdown_rx` over raw `ctrl_c()`**: The global shutdown channel is already set up and handles the signal once. Using it avoids potential conflicts with multiple signal handlers.

3. **Add i18n strings for retry cancellation**: New strings like `client.retry.cancelled` and `client.retry.skipped` should be added.

4. **Progress bar cleanup is critical**: When cancelled, `pb.finish_and_clear()` must be called to restore terminal state.

5. **The TUI needs a different approach**: The TUI can't use `indicatif` (it uses ratatui). It would need to handle cancellation through its existing input event loop.

## Follow-up Questions

- [ ] Should the TUI implement a dedicated `CountdownWidget` or extend the existing `Spinner` to support countdown mode?
- [ ] What trait bounds should `AgentRuntime` require for maximum flexibility while maintaining type safety?

## Related Discoveries

- [[025-cli-retry-wait-ux]] — Established need for countdown timer, not just spinner
- [[016-shared-agent-runtime-analysis]] — Proposed AgentRuntime would encapsulate retry logic
- [[015-cli-acp-retry-timer-intentionality]] — Confirms retry mechanism is not yet implemented
- [[008-retry-timeout-mechanism]] — How the Agent state machine expects retries to work
