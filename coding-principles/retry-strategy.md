# HTTP Retry Strategy

This document describes retry patterns for reliable HTTP API interactions.

---

## Overview

Retry logic is critical for API reliability when interacting with external services. Network conditions, rate limits, and transient server errors can cause temporary failures that succeed on subsequent attempts. Without proper retry handling, applications fail on recoverable errors, leading to poor user experience.

---

## RetryConfig Structure

```rust
pub struct RetryConfig {
    pub max_attempts: u32,
    pub base_delay: Duration,
    pub max_delay: Duration,
    pub backoff_factor: f64,
    pub jitter: bool,
    pub retryable_statuses: Vec<StatusCode>,
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_attempts` | `u32` | `3` | Maximum attempts before giving up |
| `base_delay` | `Duration` | `200ms` | Initial delay before first retry |
| `max_delay` | `Duration` | `5s` | Maximum delay between retries (cap) |
| `backoff_factor` | `f64` | `2.0` | Multiplier for exponential growth |
| `jitter` | `bool` | `true` | Whether to add randomness to delays |
| `retryable_statuses` | `Vec<StatusCode>` | `[429, 408, 502, 503, 504]` | HTTP status codes to retry |

---

## Exponential Backoff Algorithm

The delay between retries grows exponentially to avoid overwhelming a recovering service:

```
delay = base_delay × (backoff_factor ^ attempt)
```

### Calculation Steps

1. **Compute exponential delay**: `base_delay × backoff_factor^attempt`
2. **Apply cap**: `min(exponential_delay, max_delay)`
3. **Apply jitter** (if enabled): `capped_delay × (0.5 + random(0..1))`

This produces delays in the range `[capped_delay × 0.5, capped_delay × 1.5]`.

### Example Progression (defaults, no jitter)

| Attempt | Calculation | Delay |
|---------|-------------|-------|
| 0 | 200ms × 2^0 | 200ms |
| 1 | 200ms × 2^1 | 400ms |
| 2 | 200ms × 2^2 | 800ms |
| 3 | 200ms × 2^3 | 1600ms |
| 4 | 200ms × 2^4 | 3200ms |
| 5 | 200ms × 2^5 | 5000ms (capped) |

### Implementation

```rust
fn calculate_delay(config: &RetryConfig, attempt: u32) -> Duration {
    let base = config.base_delay.as_secs_f64();
    let exponential = base * config.backoff_factor.powi(attempt as i32);
    let capped = exponential.min(config.max_delay.as_secs_f64());
    
    if config.jitter {
        let jitter = rand::thread_rng().gen_range(0.5..1.5);
        Duration::from_secs_f64(capped * jitter)
    } else {
        Duration::from_secs_f64(capped)
    }
}
```

---

## RetryableError Trait

Define which errors should trigger a retry:

```rust
pub trait RetryableError {
    fn is_retryable(&self) -> bool;
}
```

### Implementation for HTTP Errors

```rust
impl RetryableError for reqwest::Error {
    fn is_retryable(&self) -> bool {
        // Network-level failures
        if self.is_timeout() || self.is_connect() {
            return true;
        }

        // HTTP status-based retries
        if let Some(status) = self.status() {
            let retryable_statuses = [
                StatusCode::TOO_MANY_REQUESTS,     // 429
                StatusCode::REQUEST_TIMEOUT,       // 408
                StatusCode::INTERNAL_SERVER_ERROR, // 500
                StatusCode::BAD_GATEWAY,           // 502
                StatusCode::SERVICE_UNAVAILABLE,   // 503
                StatusCode::GATEWAY_TIMEOUT,       // 504
            ];
            return retryable_statuses.contains(&status);
        }

        false
    }
}
```

### Custom Error Wrapper

For custom error types, implement the trait:

```rust
#[derive(Debug)]
pub struct ClientError {
    message: String,
    retryable: bool,
}

impl RetryableError for ClientError {
    fn is_retryable(&self) -> bool {
        self.retryable
    }
}
```

---

## Retryable Conditions

### HTTP Status Codes

| Code | Name | Reason |
|------|------|--------|
| 408 | Request Timeout | Client took too long; server may accept faster retry |
| 429 | Too Many Requests | Rate limited; backoff gives quota time to reset |
| 500 | Internal Server Error | Transient server issue may resolve |
| 502 | Bad Gateway | Upstream server issue; may recover quickly |
| 503 | Service Unavailable | Server overloaded or in maintenance |
| 504 | Gateway Timeout | Upstream timeout; may succeed on retry |

### Network Errors

- **Timeouts**: Request exceeded deadline
- **Connection errors**: Failed to establish connection
- **DNS failures**: May be transient

### Non-Retryable Conditions

- 400 Bad Request (client error, won't change)
- 401 Unauthorized (credentials issue)
- 403 Forbidden (permission issue)
- 404 Not Found (resource doesn't exist)
- 422 Unprocessable Entity (validation failure)

---

## retry() Function

Generic async retry wrapper:

```rust
pub async fn retry<F, Fut, T, E>(config: &RetryConfig, mut f: F) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
    E: RetryableError + std::fmt::Debug,
{
    let mut attempt = 0;
    
    loop {
        match f().await {
            Ok(result) => return Ok(result),
            Err(err) => {
                attempt += 1;
                
                if !err.is_retryable() {
                    warn!(error = ?err, "non-retryable error");
                    return Err(err);
                }
                
                if attempt >= config.max_attempts {
                    warn!(
                        error = ?err,
                        attempts = attempt,
                        "max retry attempts reached"
                    );
                    return Err(err);
                }
                
                let delay = calculate_delay(config, attempt);
                warn!(
                    error = ?err,
                    attempt = attempt,
                    max_attempts = config.max_attempts,
                    delay_ms = delay.as_millis(),
                    "retrying after error"
                );
                
                tokio::time::sleep(delay).await;
            }
        }
    }
}
```

### Behavior

1. Execute the closure
2. On success, return immediately
3. On error:
   - If not retryable, log and return error
   - If max attempts reached, log and return error
   - Otherwise, calculate delay, log, sleep, and retry

---

## Design Decisions

### Why a Separate Module/Crate?

1. **Reusability**: Multiple clients share the same retry logic
2. **Testability**: Retry logic can be unit tested with mock errors
3. **Single responsibility**: HTTP retry concerns are separate from API logic
4. **Configurability**: Each client can customize retry behavior

### Why Not External Libraries?

1. **Simplicity**: Specific needs don't require full-featured libraries
2. **Control**: Direct implementation allows precise logging and error classification
3. **Minimal dependencies**: Smaller binary and faster compilation
4. **Transparency**: Code is auditable and understandable

### Jitter Importance

Without jitter, clients that fail simultaneously retry simultaneously, causing the **thundering herd problem**:

```
Time 0:    [Client A fails] [Client B fails] [Client C fails]
Time 200ms: [A retries]     [B retries]      [C retries]     ← Server overwhelmed again
Time 400ms: [A retries]     [B retries]      [C retries]     ← Still synchronized
```

With jitter, retries spread out:

```
Time 0:    [Client A fails] [Client B fails] [Client C fails]
Time 150ms: [A retries]
Time 250ms:                 [B retries]
Time 180ms:                                  [C retries]      ← Load distributed
```

---

## Configuration Examples

### Conservative (Production APIs)

```rust
let config = RetryConfig {
    max_attempts: 3,
    base_delay: Duration::from_millis(500),
    max_delay: Duration::from_secs(30),
    backoff_factor: 2.0,
    jitter: true,
    retryable_statuses: vec![
        StatusCode::TOO_MANY_REQUESTS,
        StatusCode::SERVICE_UNAVAILABLE,
        StatusCode::GATEWAY_TIMEOUT,
    ],
};
```

### Aggressive (Internal Services)

```rust
let config = RetryConfig {
    max_attempts: 5,
    base_delay: Duration::from_millis(100),
    max_delay: Duration::from_secs(5),
    backoff_factor: 1.5,
    jitter: true,
    retryable_statuses: vec![
        StatusCode::TOO_MANY_REQUESTS,
        StatusCode::REQUEST_TIMEOUT,
        StatusCode::INTERNAL_SERVER_ERROR,
        StatusCode::BAD_GATEWAY,
        StatusCode::SERVICE_UNAVAILABLE,
        StatusCode::GATEWAY_TIMEOUT,
    ],
};
```

### No Retry (Critical Operations)

```rust
let config = RetryConfig {
    max_attempts: 1,
    ..Default::default()
};
```

---

## Usage in Request Methods

Wrap request logic with `retry()`:

```rust
let response = retry(&self.retry_config, || {
    let req = request.clone();
    let client = self.client.clone();
    async move { client.send_request(&req).await }
})
.await
.map_err(|e| ServiceError::from(e))?;
```

The closure is invoked on each attempt, allowing fresh request construction.

---

## Respecting Retry-After Headers

When servers return `Retry-After`, respect it:

```rust
async fn request_with_retry_after(&self, req: Request) -> Result<Response, Error> {
    let response = self.client.send(req.clone()).await?;
    
    if response.status() == StatusCode::TOO_MANY_REQUESTS {
        if let Some(retry_after) = response.headers().get("Retry-After") {
            let secs: u64 = retry_after.to_str()?.parse()?;
            // Add jitter to prevent synchronized retries
            let jitter = rand::thread_rng().gen_range(0.8..1.2);
            let delay = Duration::from_secs_f64(secs as f64 * jitter);
            tokio::time::sleep(delay).await;
            return self.client.send(req).await;
        }
    }
    
    Ok(response)
}
```

---

## Testing Retry Logic

### Mock Error for Testing

```rust
#[derive(Debug)]
struct MockError {
    retryable: bool,
}

impl RetryableError for MockError {
    fn is_retryable(&self) -> bool {
        self.retryable
    }
}

#[tokio::test]
async fn test_non_retryable_fails_immediately() {
    let config = RetryConfig::default();
    let mut attempts = 0;
    
    let result = retry(&config, || {
        attempts += 1;
        async { Err::<(), _>(MockError { retryable: false }) }
    }).await;
    
    assert!(result.is_err());
    assert_eq!(attempts, 1); // Only one attempt
}

#[tokio::test]
async fn test_retryable_respects_max_attempts() {
    let config = RetryConfig {
        max_attempts: 3,
        base_delay: Duration::from_millis(1), // Fast for tests
        ..Default::default()
    };
    let mut attempts = 0;
    
    let result = retry(&config, || {
        attempts += 1;
        async { Err::<(), _>(MockError { retryable: true }) }
    }).await;
    
    assert!(result.is_err());
    assert_eq!(attempts, 3);
}
```

---

## See Also

- [error-handling.md](error-handling.md) - Error handling patterns
- [patterns.md](patterns.md) - Retry with Backoff pattern
- [gotchas.md](gotchas.md) - Common retry-related pitfalls
