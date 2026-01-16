# Discovery: Should jitter be added to Retry-After values to prevent thundering herd retries?

> Category: Follow-ups (Rate Limiting / Retry Strategy)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, but the implementation should be nuanced: client-side jitter is mandatory, while server-side jitter on `Retry-After` values is recommended as defense-in-depth.**

The Loom codebase already has robust jitter support in its client-side retry logic (`loom-common-http::retry`), which uses a 0.5x to 1.5x jitter factor by default. This is the primary defense against thundering herds. However, the question of adding jitter to *server-generated* `Retry-After` values is more nuanced:

1. **Client-side jitter (ALREADY IMPLEMENTED)**: The `loom-common-http::retry` module applies jitter to calculated delays. This is the gold standard per AWS/Google/Azure recommendations.

2. **Server-side jitter on `Retry-After` (RECOMMENDED)**: When the server returns a `Retry-After` header, it should add ±10-20% jitter to protect against "naive" clients that follow the header exactly without adding their own jitter. This is defense-in-depth.

3. **Current gap**: The proposed `ServerError::RateLimited` variant (discoveries 017, 018, 030) would pass through the upstream provider's `retry_after_secs` value directly. If the upstream returns a fixed value (e.g., `30`), all Loom clients would retry at exactly the same time.

**Recommendation**: Add server-side jitter when generating `Retry-After` values, especially when the value comes from upstream providers. Use a ±10-20% range (e.g., 30 seconds becomes 27-33 seconds).

## Evidence

### Existing Client-Side Jitter Implementation

**File**: `crates/loom-common-http/src/retry.rs` (lines 66-77)

```rust
fn calculate_delay(cfg: &RetryConfig, attempt: u32) -> Duration {
    let exponential_delay = cfg.base_delay.as_secs_f64() * cfg.backoff_factor.powi(attempt as i32);
    let capped_delay = exponential_delay.min(cfg.max_delay.as_secs_f64());

    let final_delay = if cfg.jitter {
        let jitter_factor = 0.5 + fastrand::f64(); // Random factor between 0.5 and 1.5
        capped_delay * jitter_factor
    } else {
        capped_delay
    };

    Duration::from_secs_f64(final_delay)
}
```

This implements "equal jitter" with a 50% randomization factor, producing delays in the range `[delay × 0.5, delay × 1.5]`.

### Jitter Is Enabled by Default

**File**: `crates/loom-cli-config/src/defaults.rs` (lines 103-104)

```toml
# Add randomization to delays to prevent thundering herd
jitter = true
```

### Other Jitter Implementations in Codebase

The codebase has several localized jitter implementations:

1. **Audit HTTP Sink** (`crates/loom-server-audit/src/sink/http.rs:62-65`):
   ```rust
   // Exponential backoff with jitter to prevent timing attacks and thundering herd
   let base_backoff_ms = 100 * 2u64.pow(attempt - 1);
   let jitter_ms = rand::rng().random_range(0..=base_backoff_ms / 2);
   let backoff = Duration::from_millis(base_backoff_ms + jitter_ms);
   ```

2. **WireGuard Tunnel Upgrades** (`crates/loom-wgtunnel-conn/src/upgrade.rs:88-91`):
   ```rust
   pub fn upgrade_interval_with_jitter() -> Duration {
       let jitter_ms = fastrand::u64(0..5000);
       UPGRADE_INTERVAL + Duration::from_millis(jitter_ms)
   }
   ```

### Current Retry-After Flow (No Server-Side Jitter)

**File**: `crates/loom-server/src/llm_proxy.rs` (lines 418-425)

```rust
LlmError::RateLimited { retry_after_secs } => {
    tracing::warn!(retry_after = ?retry_after_secs, "LLM rate limited");
    let msg = match retry_after_secs {
        Some(secs) => format!("LLM rate limited; retry after {secs} seconds"),
        None => "LLM rate limited; try again later".to_string(),
    };
    ServerError::ServiceUnavailable(msg)  // Passes through exact value
}
```

The upstream `retry_after_secs` is passed through without modification.

### Industry Best Practices

Per AWS, Google Cloud, and Azure documentation:

| Algorithm | Formula | Use Case |
|-----------|---------|----------|
| **Full Jitter** | `sleep = random(0, backoff)` | Maximum spread, best for breaking synchronization |
| **Equal Jitter** | `sleep = backoff/2 + random(0, backoff/2)` | Predictable minimum wait (Loom uses this) |
| **Decorrelated Jitter** | `sleep = min(cap, random(base, last_sleep * 3))` | Highly contested resources |

**Server-side recommendation**: Add ±10-20% jitter to `Retry-After` values to force desynchronization of simple clients.

### Proposed Implementation

When generating `Retry-After` values in `ServerError::RateLimited`:

```rust
fn apply_server_jitter(base_secs: u64) -> u64 {
    // Apply ±15% jitter (range: 0.85 to 1.15)
    let jitter_factor = 0.85 + (fastrand::f64() * 0.30);
    ((base_secs as f64) * jitter_factor).round() as u64
}

// Usage in IntoResponse for ServerError::RateLimited
ServerError::RateLimited { message, retry_after_secs } => {
    let mut headers = HeaderMap::new();
    if let Some(secs) = retry_after_secs {
        let jittered_secs = apply_server_jitter(secs);
        headers.insert("Retry-After", HeaderValue::from(jittered_secs));
    }
    // ...
}
```

**Key Files:**
- `crates/loom-common-http/src/retry.rs` — Client-side jitter (already implemented)
- `crates/loom-server/src/error.rs` — Add server-side jitter when returning `Retry-After`
- `crates/loom-server/src/llm_proxy.rs` — Update to use `ServerError::RateLimited`

## Implications

1. **Client-side is already covered**: The `loom-common-http::retry` module with `jitter: true` (default) handles client-side jitter. No changes needed for existing retry logic.

2. **Server-side jitter is additive protection**: Even if clients add their own jitter, server-side jitter provides defense-in-depth against:
   - Naive clients that don't implement jitter
   - Clients that use the `Retry-After` value as a minimum (adding jitter on top)
   - Third-party integrations that may not follow best practices

3. **Jitter range selection**: A ±15% range is conservative enough to not significantly impact user experience while providing meaningful desynchronization. For a 30-second `Retry-After`, this produces values between 25.5 and 34.5 seconds.

4. **Consistency with existing patterns**: The codebase already uses jitter in multiple places (retry, audit, WireGuard). Adding it to `Retry-After` generation follows established patterns.

5. **Missing jitter in some areas**: The job scheduler (`loom-server-jobs/src/scheduler.rs`) and webhook retry logic (`loom-server/src/jobs/webhook_retry.rs`) use exponential backoff without jitter. These should be updated for consistency.

## Follow-up Questions

- [ ] Should the parallel execution logic be extracted into a shared crate (e.g., `loom-cli-tools` or a new `loom-tool-executor`)?
- [ ] Should parallel execution be gated behind a feature flag or CLI option for gradual rollout?

## Related Discoveries

- [[018-retry-after-header-addition]] — Proposes adding `Retry-After` HTTP header
- [[030-servererror-ratelimited-implementation-complexity]] — Proposes `ServerError::RateLimited` variant
- [[032-errorresponse-retry-after-field]] — Proposes `retry_after_secs` field in JSON body
- [[010-flock-performance-impact]] — Discusses thundering herd in context of file locks
