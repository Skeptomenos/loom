# Coding Conventions

This document captures coding conventions for building maintainable, consistent software.

---

## Code Style

### Formatting

Consistent formatting reduces cognitive load and eliminates style debates. Choose a formatter and enforce it.

**Rust (rustfmt.toml)**:
```toml
hard_tabs = true
tab_spaces = 2
max_width = 100
```

**TypeScript/JavaScript (Prettier)**:
```json
{
  "useTabs": false,
  "tabWidth": 2,
  "singleQuote": true,
  "printWidth": 100,
  "trailingComma": "es5"
}
```

- Run formatters before committing (use pre-commit hooks)
- Configure your editor to format on save

### Linting

Treat lint warnings as errors. Fix them, don't suppress them.

**Rust**:
```bash
cargo clippy --workspace -- -D warnings
```

**TypeScript**:
```bash
eslint --max-warnings 0 .
```

---

## Naming Conventions

### Packages/Crates

Use a consistent prefix pattern: `{project}-{domain}-{component}`

| Domain | Examples |
|--------|----------|
| `server` | `{project}-server`, `{project}-server-auth` |
| `client` | `{project}-client`, `{project}-client-sdk` |
| `common` | `{project}-common-core`, `{project}-common-http` |
| `cli` | `{project}-cli`, `{project}-cli-tools` |

### Files

- **Rust files**: `snake_case.rs` (e.g., `user_service.rs`)
- **TypeScript files**: `kebab-case.ts` or `PascalCase.tsx` for components
- **Migrations**: `NNN_description.sql` (e.g., `001_initial_schema.sql`)
- **Specs/docs**: `kebab-case.md` (e.g., `error-handling.md`)

### Code Elements

| Element | Rust | TypeScript |
|---------|------|------------|
| Functions | `snake_case` | `camelCase` |
| Types/Structs/Classes | `PascalCase` | `PascalCase` |
| Interfaces/Traits | `PascalCase` | `PascalCase` (prefix `I` optional) |
| Enums | `PascalCase` | `PascalCase` |
| Enum Variants | `PascalCase` | `PascalCase` |
| Constants | `SCREAMING_CASE` | `SCREAMING_CASE` |
| Type Aliases | `PascalCase` | `PascalCase` |

### API Error Codes

Machine-readable error codes use `snake_case`:

```rust
error: "not_found"
error: "already_exists"
error: "invalid_input"
error: "rate_limited"
```

---

## Import Organization

Order imports in logical groups, separated by blank lines:

### Rust

```rust
// 1. Standard library
use std::collections::HashMap;
use std::sync::Arc;

// 2. External crates
use anyhow::{Context, Result};
use serde::{Deserialize, Serialize};
use tokio::sync::Mutex;

// 3. Internal project crates
use project_common_core::{Request, Response};
use project_common_secret::SecretString;
```

### TypeScript

```typescript
// 1. Node built-ins
import { readFile } from 'fs/promises';
import path from 'path';

// 2. External packages
import express from 'express';
import { z } from 'zod';

// 3. Internal modules
import { UserService } from './services/user';
import type { Config } from './types';
```

---

## Error Handling

### Crate-Level Error Types

Each module/crate defines its own error enum:

```rust
// src/error.rs
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("Database error: {0}")]
    Db(#[from] sqlx::Error),
    
    #[error("Not found: {0}")]
    NotFound(String),
    
    #[error("Validation failed: {0}")]
    Validation(String),
}
```

### Result Type Aliases

Define a `Result<T>` alias per module:

```rust
pub type Result<T> = std::result::Result<T, ServiceError>;
```

### Error Propagation

Use context for better error messages:

```rust
let data = fs::read_to_string(&path)
    .context("failed to read config file")?;
```

### Never Suppress Errors

```rust
// BAD - error silently ignored
let _ = do_something();

// GOOD - error explicitly handled
if let Err(e) = do_something() {
    tracing::warn!(error = %e, "operation failed, continuing");
}
```

---

## Async Conventions

### Runtime (Rust)

All async code uses **Tokio**:

```rust
#[tokio::main]
async fn main() -> Result<()> {
    // ...
}
```

### Async Traits (Rust)

Use `async-trait` for async trait methods:

```rust
use async_trait::async_trait;

#[async_trait]
pub trait Repository: Send + Sync {
    async fn find_by_id(&self, id: &str) -> Result<Option<Entity>>;
}
```

### Send + Sync Bounds

Traits used across threads require `Send + Sync`:

```rust
pub trait Service: Send + Sync { ... }
```

---

## Logging & Instrumentation

### Structured Logging

Use structured fields, not string interpolation:

```rust
use tracing::{debug, info, warn, error, instrument};

// GOOD - structured fields
info!(user_id = %user.id, action = "login", "user logged in");
warn!(error = %e, "operation failed");

// AVOID - string interpolation
warn!("user {} failed: {}", user.id, e);
```

### Function Instrumentation

```rust
#[instrument(skip(self, request), fields(model = %request.model))]
async fn process(&self, request: Request) -> Result<Response> {
    // ...
}
```

**Rules**:
- Always `skip(self)` (not useful in logs)
- Always skip secrets and large arguments
- Add meaningful fields with `%` (Display) or `?` (Debug)

### Never Log Secrets

```rust
// BAD
debug!(api_key = %key, "using key");

// GOOD - Secret<T> auto-redacts
debug!(api_key = ?secret_key, "using key");  // Logs "[REDACTED]"
```

---

## HTTP Conventions

### Client Construction

Never construct HTTP clients directly. Use a centralized factory:

```rust
// BAD
let client = reqwest::Client::new();

// GOOD
let client = project_http::new_client();
```

This ensures consistent:
- User-Agent headers
- Timeout configuration
- Retry behavior
- Connection pooling

### Error Responses

All HTTP errors return JSON with consistent structure:

```rust
pub struct ErrorResponse {
    pub error: String,      // Machine-readable: "not_found"
    pub message: String,    // Human-readable: "User not found: abc123"
}
```

### Route Organization

Separate routes by authentication requirement:

```rust
// Public routes (no auth required)
let public = Router::new()
    .route("/health", get(health_handler))
    .route("/version", get(version_handler));

// Protected routes (auth required)
let protected = Router::new()
    .route("/api/users", get(list_users))
    .route("/api/users/:id", get(get_user))
    .layer(auth_middleware);
```

---

## Database Conventions

### Migrations

- **Location**: Single directory for all migrations
- **Naming**: `NNN_description.sql` (check existing for next number)
- **Idempotency**: Migrations should be safe to run multiple times where possible

### Queries

Use compile-time checked queries:

```rust
let user = sqlx::query_as!(
    UserRow,
    "SELECT id, name, email FROM users WHERE id = ?",
    user_id
)
.fetch_optional(&pool)
.await?;
```

---

## Testing Conventions

### Unit Tests

Place in `#[cfg(test)]` module within the source file:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_validation() {
        // ...
    }
    
    #[tokio::test]
    async fn test_async_operation() {
        // ...
    }
}
```

### Integration Tests

Place in `tests/` directory for cross-module testing.

### Test Documentation

Every test should document:
1. What it tests
2. Why it's important
3. What invariant it verifies

---

## Documentation Conventions

### Doc Comments

Use `///` for public API documentation:

```rust
/// Executes a command with the given arguments.
///
/// # Arguments
/// * `args` - Command arguments
/// * `timeout` - Maximum execution time
///
/// # Errors
/// Returns `CommandError` if execution fails or times out.
pub async fn execute(&self, args: &[String], timeout: Duration) -> Result<Output> {
    // ...
}
```

### Code Comments

From AGENTS.md:
> **No comments** unless code is complex and requires context for future developers.

- Don't comment obvious code
- Do comment non-obvious decisions
- Explain "why", not "what"

---

## Svelte 5 Conventions (Frontend)

**Always use Svelte 5 runes. Never use Svelte 4 patterns.**

| Category | Svelte 5 | Svelte 4 (DO NOT USE) |
|----------|----------|----------------------|
| State | `let count = $state(0);` | `let count = 0;` |
| Derived | `const doubled = $derived(count * 2);` | `$: doubled = count * 2;` |
| Effects | `$effect(() => { ... });` | `$: { ... }` |
| Props | `let { foo, bar } = $props();` | `export let foo;` |
| Events | `onclick={handler}` | `on:click={handler}` |
| Custom events | Pass callback props: `onsave={fn}` | `createEventDispatcher` |
| Slots | `{@render children()}` | `<slot />` |

---

## Pre-Submission Checklist

Before submitting code:

- [ ] Formatter passes (`cargo fmt`, `prettier`)
- [ ] Linter passes with zero warnings
- [ ] Tests pass
- [ ] Imports organized (std → external → internal)
- [ ] Error types properly defined
- [ ] Secrets wrapped in redacting types
- [ ] Functions instrumented with `#[instrument]`
- [ ] HTTP clients use factory function
- [ ] No sensitive data in logs
- [ ] Documentation for public APIs
