# Discovery 009: OAuth State Store for CSRF Protection

## Question
How is the OAuth state store implemented for CSRF protection?

## Answer
Loom implements a robust **in-memory OAuth state store** (`OAuthStateStore`) that provides CSRF protection through cryptographically random state tokens, single-use consumption, provider binding, and time-limited validity. The store is located in `crates/loom-server/src/oauth_state.rs` and is shared across all OAuth flows (GitHub, Google, Okta, and Anthropic admin).

The design follows security best practices: state tokens are UUID v4 (122 bits of randomness), each state can only be consumed once (removed on read), states expire after 10 minutes, and redirect URLs are sanitized to prevent open redirect attacks.

## Evidence

### State Store Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OAuthStateStore                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  Arc<RwLock<HashMap<String, OAuthStateEntry>>>                              │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ OAuthStateEntry                                                      │    │
│  │   - provider: String        (e.g., "github", "google", "okta")      │    │
│  │   - nonce: Option<String>   (OIDC nonce or PKCE verifier)           │    │
│  │   - created_at: Instant     (for expiry calculation)                │    │
│  │   - redirect_url: Option<String>  (sanitized post-login redirect)   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### State Generation

```rust
// crates/loom-server/src/oauth_state.rs:259-263
pub fn generate_state() -> String {
    let state = uuid::Uuid::new_v4().to_string();
    tracing::debug!("Generated OAuth state");
    state
}
```

UUID v4 provides 122 bits of randomness, sufficient for CSRF protection.

### Store and Validate Flow

**1. Before OAuth redirect (store state):**
```rust
// crates/loom-server/src/routes/auth.rs:630-643
let oauth_state = generate_state();
state
    .oauth_state_store
    .store(
        oauth_state.clone(),
        "github".to_string(),
        None,  // nonce (for OIDC)
        redirect_url_param,  // sanitized redirect
    )
    .await;
```

**2. On OAuth callback (validate and consume):**
```rust
// crates/loom-server/src/routes/auth.rs:844-860
let state_entry = match state
    .oauth_state_store
    .validate_and_consume(oauth_state, "github")
    .await
{
    Some(entry) => entry,
    None => {
        return (
            StatusCode::BAD_REQUEST,
            Json(AuthErrorResponse {
                error: "invalid_state".to_string(),
                message: "Invalid or expired state parameter".to_string(),
            }),
        ).into_response();
    }
};
```

### Single-Use Consumption (Critical Security Property)

```rust
// crates/loom-server/src/oauth_state.rs:175-209
pub async fn validate_and_consume(
    &self,
    state: &str,
    expected_provider: &str,
) -> Option<OAuthStateEntry> {
    let mut states = self.states.write().await;

    // CRITICAL: Remove immediately, even before validation
    if let Some(entry) = states.remove(state) {
        let expiry = Duration::from_secs(STATE_EXPIRY_SECONDS);
        let elapsed = entry.created_at.elapsed();

        if elapsed >= expiry {
            tracing::debug!("OAuth state expired");
            return None;
        }

        if entry.provider != expected_provider {
            tracing::warn!("OAuth state provider mismatch");
            return None;
        }

        return Some(entry);
    }

    None
}
```

The state is **always removed** from the store, even if validation fails. This prevents:
- Replay attacks (state can't be reused)
- Timing attacks (can't probe for valid states)

### Expiration and Cleanup

**Expiry constant:**
```rust
// crates/loom-server/src/oauth_state.rs:59
const STATE_EXPIRY_SECONDS: u64 = 600;  // 10 minutes
```

**Background cleanup job:**
```rust
// crates/loom-server/src/jobs/oauth_state_cleanup.rs
pub struct OAuthStateCleanupJob {
    store: Arc<OAuthStateStore>,
}

impl Job for OAuthStateCleanupJob {
    async fn run(&self, ctx: &JobContext) -> Result<JobOutput, JobError> {
        let removed = self.store.cleanup_expired().await;
        Ok(JobOutput {
            message: format!("Removed {} expired OAuth states", removed),
            metadata: Some(serde_json::json!({ "removed_count": removed })),
        })
    }
}
```

**Cleanup scheduling (default 15 minutes):**
```rust
// crates/loom-server/src/main.rs:129-134
Arc::new(OAuthStateCleanupJob::new(Arc::clone(
    &state.oauth_state_store,
))),
Duration::from_secs(config.auth.oauth_state_cleanup_interval_secs),
```

### Open Redirect Protection

```rust
// crates/loom-server/src/oauth_state.rs:81-91
pub fn is_safe_redirect(url: &str) -> bool {
    url.starts_with('/') && !url.starts_with("//")
}

pub fn sanitize_redirect(url: Option<&str>) -> String {
    match url {
        Some(u) if is_safe_redirect(u) => u.to_string(),
        _ => "/".to_string(),
    }
}
```

This prevents attackers from using the OAuth flow to redirect users to malicious external sites.

### PKCE Support for Anthropic Admin OAuth

The Anthropic admin flow repurposes the `nonce` field to store the PKCE `code_verifier`:

```rust
// crates/loom-server/src/routes/admin_anthropic.rs:239-263
let pkce = Pkce::generate();
let oauth_state = generate_state();

state
    .oauth_state_store
    .store(
        oauth_state.clone(),
        ANTHROPIC_ADMIN_PROVIDER.to_string(),
        Some(pkce.verifier),  // PKCE verifier stored as "nonce"
        body.redirect_after,
    )
    .await;
```

### Security Properties Summary

| Property | Implementation |
|----------|----------------|
| **CSRF Protection** | Cryptographically random state (UUID v4, 122 bits) |
| **Replay Prevention** | Single-use consumption (removed on read) |
| **Time-Limited** | 10-minute expiry |
| **Provider Binding** | State bound to specific OAuth provider |
| **Nonce Support** | Optional nonce for OIDC replay protection |
| **PKCE Support** | Verifier stored in nonce field for Anthropic flow |
| **Open Redirect Prevention** | Redirect URLs sanitized to relative paths only |

### Thread Safety

```rust
// crates/loom-server/src/oauth_state.rs:108-110
pub struct OAuthStateStore {
    states: Arc<RwLock<HashMap<String, OAuthStateEntry>>>,
}
```

Uses `tokio::sync::RwLock` for concurrent access - multiple readers, exclusive writers.

## Implications

1. **In-memory only**: State is lost on server restart. This is acceptable because OAuth flows are short-lived (10 minutes max). Users simply restart the login flow.

2. **Single-server limitation**: The in-memory store doesn't work with multiple server instances. For horizontal scaling, would need Redis or database-backed store.

3. **No persistence needed**: Unlike sessions, OAuth state doesn't need to survive restarts. The 10-minute window is intentionally short.

4. **Cleanup is defensive**: The background job is a safety net. Most states are consumed normally; cleanup handles abandoned flows.

5. **Provider binding prevents cross-provider attacks**: A state generated for GitHub can't be used for Google callback.

## Key Files

- `crates/loom-server/src/oauth_state.rs` - Core implementation
- `crates/loom-server/src/routes/auth.rs` - Usage in user auth flows
- `crates/loom-server/src/routes/admin_anthropic.rs` - Usage in admin OAuth with PKCE
- `crates/loom-server/src/jobs/oauth_state_cleanup.rs` - Background cleanup job
- `crates/loom-server/src/api.rs` - Store initialization and injection into AppState
- `crates/loom-server-config/src/sections/auth.rs` - Cleanup interval configuration

## Follow-up Questions

- [ ] How does the impersonation system work end-to-end?
- [ ] How does the session management differ from OAuth state (persistence, expiry, revocation)?
