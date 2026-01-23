# Error Handling

This document describes a structured, type-safe approach to error handling.

---

## Overview

The design philosophy prioritizes:

- **Explicit error types** over generic errors in library code
- **Automatic error conversion** via `From` implementations
- **Recoverable vs fatal** distinction through error variants
- **Structured logging** for observability

---

## Error Type Hierarchy

### Top-Level Error

The root error type for your domain:

```rust
#[derive(Error, Debug)]
pub enum ServiceError {
    #[error("External API error: {0}")]
    Api(#[from] ApiError),

    #[error("Operation error: {0}")]
    Operation(#[from] OperationError),

    #[error("Invalid state: {0}")]
    InvalidState(String),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Operation timed out: {0}")]
    Timeout(String),

    #[error("Internal error: {0}")]
    Internal(String),
}
```

| Variant | Purpose |
|---------|---------|
| `Api` | Wraps external API errors |
| `Operation` | Wraps domain operation errors |
| `InvalidState` | Invalid state for current operation |
| `Io` | File system and I/O errors |
| `Timeout` | Operation exceeded timeout |
| `Internal` | Unexpected errors (bugs) |

### API/External Service Errors

Errors from external service interactions:

```rust
#[derive(Clone, Error, Debug)]
pub enum ApiError {
    #[error("HTTP error: {0}")]
    Http(String),

    #[error("API error: {0}")]
    Api(String),

    #[error("Request timed out")]
    Timeout,

    #[error("Invalid response: {0}")]
    InvalidResponse(String),

    #[error("Rate limited: retry after {retry_after_secs:?} seconds")]
    RateLimited { retry_after_secs: Option<u64> },
}
```

| Variant | Transient? | Description |
|---------|-----------|-------------|
| `Http` | Yes | Network/transport failures |
| `Api` | Maybe | Service returned an error |
| `Timeout` | Yes | Request exceeded deadline |
| `InvalidResponse` | No | Malformed response |
| `RateLimited` | Yes | Rate limit hit |

### Operation Errors

Errors from domain operations:

```rust
#[derive(Clone, Error, Debug)]
pub enum OperationError {
    #[error("Not found: {0}")]
    NotFound(String),

    #[error("Invalid arguments: {0}")]
    InvalidArguments(String),

    #[error("IO error: {0}")]
    Io(String),

    #[error("Operation timed out")]
    Timeout,

    #[error("Access denied: {0}")]
    AccessDenied(String),

    #[error("Validation failed: {0}")]
    Validation(String),
}
```

---

## Error Propagation

### From Implementations

Errors convert automatically via `#[from]` attribute:

```rust
// ApiError -> ServiceError
impl From<ApiError> for ServiceError { ... }

// OperationError -> ServiceError  
impl From<OperationError> for ServiceError { ... }

// std::io::Error -> ServiceError
impl From<std::io::Error> for ServiceError { ... }

// std::io::Error -> OperationError (converts to string for Clone)
impl From<std::io::Error> for OperationError {
    fn from(err: std::io::Error) -> Self {
        OperationError::Io(err.to_string())
    }
}
```

### Result Type Aliases

Each module defines its own `Result<T>` alias:

```rust
pub type Result<T> = std::result::Result<T, ServiceError>;
```

### Propagation Pattern

Errors bubble up through layers using `?` operator:

```
Operation Implementation
        ↓ OperationError
Service Layer
        ↓ ServiceError::Operation
Handler Layer
        ↓ HTTP Response / DisplayError
Client/UI Layer
```

---

## Recovery Strategies

### Automatic Retry for Transient Errors

Implement retry for transient failures:

1. **Error occurs** during operation
2. **Check if retryable** via trait or enum matching
3. **Increment retry count** and check against maximum
4. **If retries available**: wait with backoff, then retry
5. **If max retries reached**: return error to caller

```rust
async fn with_retry<F, T, E>(
    config: &RetryConfig,
    mut operation: F,
) -> Result<T, E>
where
    F: FnMut() -> Future<Output = Result<T, E>>,
    E: RetryableError,
{
    let mut attempts = 0;
    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) if e.is_retryable() && attempts < config.max_attempts => {
                attempts += 1;
                let delay = calculate_backoff(config, attempts);
                tokio::time::sleep(delay).await;
            }
            Err(e) => return Err(e),
        }
    }
}
```

### RetryableError Trait

Define which errors should trigger retries:

```rust
pub trait RetryableError {
    fn is_retryable(&self) -> bool;
}

impl RetryableError for ApiError {
    fn is_retryable(&self) -> bool {
        matches!(
            self,
            ApiError::Http(_) | ApiError::Timeout | ApiError::RateLimited { .. }
        )
    }
}
```

### Error Origin Tracking

Track where errors originated for recovery decisions:

```rust
pub enum ErrorOrigin {
    ExternalApi,  // Eligible for automatic retry
    Operation,    // May be reported for self-correction
    Io,           // Typically fatal
}

pub struct RecoverableError {
    pub error: ServiceError,
    pub origin: ErrorOrigin,
    pub retries: u32,
}
```

---

## User-Facing Errors

### Error Display

Convert errors to user-friendly messages:

```rust
impl Display for ServiceError {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        match self {
            Self::Api(e) => write!(f, "Service unavailable: {}", e),
            Self::Operation(e) => write!(f, "Operation failed: {}", e),
            Self::Timeout(msg) => write!(f, "Operation timed out: {}", msg),
            Self::Internal(_) => write!(f, "An unexpected error occurred"),
            // Don't expose internal error details to users
        }
    }
}
```

### HTTP Error Responses

Map errors to HTTP responses consistently:

```rust
impl IntoResponse for ServiceError {
    fn into_response(self) -> Response {
        let (status, error_code, message) = match &self {
            Self::Operation(OperationError::NotFound(msg)) => {
                (StatusCode::NOT_FOUND, "not_found", msg.clone())
            }
            Self::Operation(OperationError::Validation(msg)) => {
                (StatusCode::BAD_REQUEST, "validation_failed", msg.clone())
            }
            Self::Operation(OperationError::AccessDenied(msg)) => {
                (StatusCode::FORBIDDEN, "access_denied", msg.clone())
            }
            Self::Api(ApiError::RateLimited { .. }) => {
                (StatusCode::TOO_MANY_REQUESTS, "rate_limited", "Too many requests".into())
            }
            _ => {
                (StatusCode::INTERNAL_SERVER_ERROR, "internal_error", "Internal error".into())
            }
        };

        (status, Json(ErrorResponse { error: error_code.into(), message })).into_response()
    }
}
```

---

## Design Decisions

### Why thiserror Over anyhow in Libraries

| thiserror | anyhow |
|-----------|--------|
| Typed error variants | Erased error type |
| Pattern matching on errors | String-based inspection |
| Compile-time exhaustiveness | Runtime error checking |
| Clear API contracts | Flexible but opaque |

**Library code** uses `thiserror` because:
1. Callers need to match on specific error types for recovery
2. Error variants form part of the public API contract
3. Recovery logic depends on error type

### Why anyhow in Application Code

**Application/binary code** may use `anyhow` because:
1. Top-level code often just displays errors
2. No downstream callers need to match on variants
3. Simpler context chaining with `.context()`

### Clone Constraints on Error Types

Some error types need `Clone`:

```rust
#[derive(Clone, Error, Debug)]
pub enum ApiError { ... }
```

This is required when:
1. Errors are stored in state that may be cloned
2. Errors need to be duplicated for logging and returning
3. `std::io::Error` doesn't implement `Clone`, so store as `String`

---

## Logging Errors

### Structured Logging

Log errors with structured fields:

```rust
use tracing::{warn, info};

warn!(
    error = %e,                      // Display format
    retries = attempts,              // Retry count
    max_retries = config.max_attempts,
    "operation failed"
);

info!(
    from = %old_state,
    to = "Error",
    "state transition (will retry)"
);
```

### Best Practices

1. **Use structured fields** over string interpolation:
   ```rust
   // Good
   warn!(error = %e, operation = %name, "operation failed");

   // Avoid
   warn!("operation {} failed: {}", name, e);
   ```

2. **Include context** relevant to debugging:
   - Current state/phase
   - Retry counts
   - Operation identifiers
   - Timing information

3. **Choose appropriate levels**:
   - `error!` - Unrecoverable failures
   - `warn!` - Recoverable errors (retries, expected failures)
   - `info!` - State transitions, significant events
   - `debug!` - Detailed operation tracing

4. **Use Display (`%`) for errors**:
   ```rust
   warn!(error = %e, "operation failed");
   ```

5. **Never log sensitive data**:
   ```rust
   // BAD
   debug!(password = %password, "authenticating");

   // GOOD
   debug!(user = %username, "authenticating");
   ```

---

## Error Code Guidelines

### Machine-Readable Codes

Use consistent, lowercase snake_case codes:

```rust
// Good error codes
"not_found"
"validation_failed"
"rate_limited"
"already_exists"
"quota_exceeded"

// Avoid
"NotFound"           // Wrong case
"VALIDATION_FAILED"  // Wrong case
"error-occurred"     // Wrong separator
"404"                // Use HTTP status instead
```

### Generic vs Domain-Specific

| Type | Example | When to Use |
|------|---------|-------------|
| Generic | `not_found`, `bad_request` | When HTTP status is sufficient |
| Domain-specific | `already_member`, `quota_exceeded` | When action is possible |

Domain-specific codes enable programmatic handling:

```rust
match error.code {
    "quota_exceeded" => show_upgrade_prompt(),
    "already_member" => show_existing_member_ui(),
    _ => show_generic_error(error.message),
}
```

---

## See Also

- [retry-strategy.md](retry-strategy.md) - HTTP retry patterns
- [conventions.md](conventions.md) - Coding conventions
- [patterns.md](patterns.md) - Design patterns
