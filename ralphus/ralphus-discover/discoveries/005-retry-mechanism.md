# Discovery 005: Retry Mechanism for LLM Errors

## Question
How is the retry mechanism implemented for LLM errors?

## Answer
Loom implements a **generic, configurable retry mechanism** in `loom-common-http` that uses **exponential backoff with jitter**. The system classifies errors as retryable or non-retryable via the `RetryableError` trait, and each LLM provider wraps its errors to implement this trait.

## Evidence

### Core Retry Infrastructure (`loom-common-http/src/retry.rs`)

```rust
#[derive(Debug, Clone)]
pub struct RetryConfig {
    pub max_attempts: u32,           // Default: 3
    pub base_delay: Duration,        // Default: 200ms
    pub max_delay: Duration,         // Default: 5s
    pub backoff_factor: f64,         // Default: 2.0
    pub jitter: bool,                // Default: true
    pub retryable_statuses: Vec<StatusCode>,
}
```

**Exponential Backoff Algorithm:**
```
delay = base_delay × (backoff_factor ^ attempt)
capped_delay = min(delay, max_delay)
final_delay = capped_delay × (0.5 + random(0..1))  // if jitter enabled
```

### RetryableError Trait

```rust
pub trait RetryableError {
    fn is_retryable(&self) -> bool;
}
```

Each error type implements this trait to classify itself:

| Error Type | Retryable Conditions |
|------------|---------------------|
| `reqwest::Error` | `is_timeout()`, `is_connect()`, or status in [408, 429, 500, 502, 503, 504] |
| `OpenAIRequestError` | `LlmError::Http`, `LlmError::Timeout`, `LlmError::RateLimited` |
| `ClientError` (Anthropic) | Based on `ClientErrorKind::Transient` |

### Error Classification (Anthropic)

The Anthropic client has sophisticated error classification:

```rust
pub enum ClientErrorKind {
    Transient,      // Retry on same account (408, 429, 500, 502, 503, 504)
    QuotaExceeded,  // Failover to next account (429 + quota message)
    Permanent,      // Disable account (401, 403)
}

fn classify_error(status: u16, message: &str) -> ClientErrorKind {
    if status == 401 || status == 403 {
        return ClientErrorKind::Permanent;
    }
    if status == 429 && is_quota_message(message) {
        return ClientErrorKind::QuotaExceeded;
    }
    if matches!(status, 408 | 429 | 500 | 502 | 503 | 504) {
        return ClientErrorKind::Transient;
    }
    ClientErrorKind::Permanent
}
```

### Quota Detection

Special handling for Claude subscription 5-hour rolling limits:

```rust
pub fn is_quota_message(msg: &str) -> bool {
    let lower = msg.to_ascii_lowercase();
    lower.contains("5-hour")
        || lower.contains("rolling window")
        || lower.contains("usage limit for your plan")
        || lower.contains("subscription usage limit")
}
```

### Usage Pattern in LLM Clients

```rust
// Anthropic client
let response = retry(&self.retry_config, || {
    let req = anthropic_request_clone.clone();
    let c = client.clone();
    async move { c.send_request(&req).await }
})
.await
.map_err(LlmError::from)?;
```

### Provider-Specific Configurations

| Provider | Base Delay | Max Delay | Max Attempts |
|----------|------------|-----------|--------------|
| Anthropic | 200ms | 5s | 3 |
| OpenAI | 500ms | 30s | 3 |

### Jitter Purpose

Prevents **thundering herd problem** where synchronized clients overwhelm a recovering service:

```
Without jitter: All clients retry at exactly 200ms, 400ms, 800ms...
With jitter:    Client A at 150ms, Client B at 250ms, Client C at 180ms...
```

## Implications

1. **Resilience**: Transient failures (network issues, rate limits, server errors) are automatically retried
2. **Pool Failover**: Quota exhaustion triggers failover to next OAuth account, not just retry
3. **Fast Failure**: Non-retryable errors (auth failures) fail immediately without wasting attempts
4. **Customizable**: Each client can tune retry behavior for its provider's characteristics
5. **Observable**: All retry attempts are logged with structured tracing

## Key Files

- `crates/loom-common-http/src/retry.rs` - Core retry infrastructure
- `crates/loom-server-llm-anthropic/src/client.rs` - Anthropic error classification
- `crates/loom-server-llm-openai/src/client.rs` - OpenAI retry wrapper
- `specs/retry-strategy.md` - Design specification

## Follow-up Questions

- [ ] How does the OAuth pool failover work when quota is exceeded?
- [ ] How are streaming responses handled when errors occur mid-stream?
