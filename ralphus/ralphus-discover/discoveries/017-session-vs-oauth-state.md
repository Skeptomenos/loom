# Discovery 017: Session Management vs OAuth State

## Question
How does the session management differ from OAuth state (persistence, expiry, revocation)?

## Answer
Loom maintains a clear architectural separation between **User Sessions** (long-lived, persistent, authenticated state) and **OAuth State Tokens** (short-lived, in-memory, CSRF protection). Sessions represent authenticated user identity and persist across server restarts in SQLite, while OAuth state tokens exist only to secure the OAuth flow itself and are stored transiently in memory.

The key differences span storage backend, TTL model, security properties, and cleanup mechanisms. Sessions use a 60-day sliding expiry with database persistence, while OAuth state uses a 10-minute fixed expiry with in-memory storage.

## Evidence

### Architecture Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         USER SESSIONS                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  Storage: SQLite (sessions table)                                           │
│  TTL: 60 days SLIDING (each use resets the clock)                           │
│  Token: 32-byte random hex (64 chars), stored as SHA-256 hash               │
│  Survives: Server restarts, deployments                                     │
│  Cleanup: SessionCleanupJob (database DELETE)                               │
│  Revocation: Explicit (DELETE /api/sessions/{id}) or expiry                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         OAUTH STATE TOKENS                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  Storage: In-memory HashMap (Arc<RwLock<HashMap>>)                          │
│  TTL: 10 minutes FIXED (no extension)                                       │
│  Token: UUID v4 (122 bits randomness)                                       │
│  Survives: Nothing (lost on restart)                                        │
│  Cleanup: OAuthStateCleanupJob (memory removal)                             │
│  Revocation: Single-use consumption (removed on read)                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Session Struct and Expiry

```rust
// crates/loom-server-auth/src/session.rs:32-47
pub const SESSION_EXPIRY_DAYS: i64 = 60;

pub struct Session {
    pub id: SessionId,
    pub user_id: UserId,
    pub session_type: SessionType,  // Web, Cli, VsCode
    pub created_at: DateTime<Utc>,
    pub last_used_at: DateTime<Utc>,
    pub expires_at: DateTime<Utc>,
    pub ip_address: Option<String>,
    pub user_agent: Option<String>,
    pub geo_city: Option<String>,
    pub geo_country: Option<String>,
}

impl Session {
    pub fn extend(&mut self) {
        let now = Utc::now();
        self.last_used_at = now;
        self.expires_at = now + Duration::days(SESSION_EXPIRY_DAYS);
    }
}
```

### OAuth State Struct and Expiry

```rust
// crates/loom-server/src/oauth_state.rs:54-75
const STATE_EXPIRY_SECONDS: u64 = 600;  // 10 minutes

pub struct OAuthStateEntry {
    pub provider: String,           // "github", "google", "okta"
    pub nonce: Option<String>,      // OIDC nonce or PKCE verifier
    pub created_at: Instant,        // For expiry calculation
    pub redirect_url: Option<String>,
}

pub struct OAuthStateStore {
    states: Arc<RwLock<HashMap<String, OAuthStateEntry>>>,
}
```

### Storage Backend Differences

**Sessions (Database Persistence):**
```rust
// crates/loom-server-db/src/session.rs:263-290
pub async fn create_session(&self, session: &Session, token_hash: &str) -> Result<(), DbError> {
    sqlx::query(
        r#"
        INSERT INTO sessions (
            id, user_id, session_type, token_hash,
            created_at, last_used_at, expires_at,
            ip_address, user_agent, geo_city, geo_country
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        "#,
    )
    .bind(session.id.to_string())
    .bind(session.user_id.to_string())
    .bind(token_hash)  // SHA-256 hash, never plaintext
    // ...
    .execute(&self.pool)
    .await?;
}
```

**OAuth State (In-Memory):**
```rust
// crates/loom-server/src/oauth_state.rs:137-153
pub async fn store(
    &self,
    state: String,
    provider: String,
    nonce: Option<String>,
    redirect_url: Option<String>,
) {
    let entry = OAuthStateEntry {
        provider,
        nonce,
        created_at: Instant::now(),
        redirect_url,
    };
    let mut states = self.states.write().await;
    states.insert(state, entry);  // Simple HashMap insert
}
```

### Expiry Model Differences

**Sessions: Sliding Expiry (60 days)**
```rust
// crates/loom-server-db/src/session.rs:366-384
pub async fn update_session_last_used(&self, id: &SessionId) -> Result<(), DbError> {
    let now = Utc::now();
    let expires_at = now + Duration::days(loom_server_auth::SESSION_EXPIRY_DAYS);

    sqlx::query(
        r#"
        UPDATE sessions
        SET last_used_at = ?, expires_at = ?
        WHERE id = ?
        "#,
    )
    .bind(now.to_rfc3339())
    .bind(expires_at.to_rfc3339())  // Reset to 60 days from NOW
    .bind(id.to_string())
    .execute(&self.pool)
    .await?;
}
```

**OAuth State: Fixed Expiry (10 minutes)**
```rust
// crates/loom-server/src/oauth_state.rs:175-209
pub async fn validate_and_consume(&self, state: &str, expected_provider: &str) -> Option<OAuthStateEntry> {
    let mut states = self.states.write().await;

    if let Some(entry) = states.remove(state) {  // Always removed
        let expiry = Duration::from_secs(STATE_EXPIRY_SECONDS);
        let elapsed = entry.created_at.elapsed();

        if elapsed >= expiry {
            return None;  // Expired, no extension possible
        }
        // ...
        return Some(entry);
    }
    None
}
```

### Revocation Mechanisms

**Sessions: Explicit Revocation via API**
```rust
// crates/loom-server/src/routes/sessions.rs:114-176
pub async fn revoke_session(
    State(state): State<AppState>,
    Path(session_id): Path<String>,
    current_user: CurrentUser,
) -> impl IntoResponse {
    // Verify session belongs to user
    let sessions = state.session_repo.get_sessions_for_user(&current_user.user.id).await?;
    if !sessions.iter().any(|s| s.id == session_id) {
        return Err("Session not found");
    }

    // Hard delete from database
    state.session_repo.delete_session(&session_id).await?;

    // Audit log
    AuditLogBuilder::new(AuditEventType::SessionRevoked)
        .resource("session", session_id.to_string())
        .build();
}
```

**OAuth State: Single-Use Consumption**
```rust
// crates/loom-server/src/oauth_state.rs:182
if let Some(entry) = states.remove(state) {  // ALWAYS removed on read
    // Even if validation fails, state is gone
    // Prevents replay attacks and timing attacks
}
```

### Cleanup Jobs

**SessionCleanupJob (Database):**
```rust
// crates/loom-server/src/jobs/session_cleanup.rs:34-95
async fn run(&self, ctx: &JobContext) -> Result<JobOutput, JobError> {
    let sessions_deleted = self.session_repo.cleanup_expired_sessions().await?;
    let tokens_deleted = self.session_repo.cleanup_expired_access_tokens().await?;
    let device_codes_deleted = self.session_repo.cleanup_expired_device_codes().await?;
    let magic_links_deleted = self.session_repo.cleanup_expired_magic_links().await?;
    // ...
}
```

**OAuthStateCleanupJob (Memory):**
```rust
// crates/loom-server/src/oauth_state.rs:220-234
pub async fn cleanup_expired(&self) -> usize {
    let expiry = Duration::from_secs(STATE_EXPIRY_SECONDS);
    let mut states = self.states.write().await;
    let before = states.len();
    states.retain(|_, entry| entry.created_at.elapsed() < expiry);
    before - states.len()
}
```

### Token Security Comparison

| Aspect | Sessions | OAuth State |
|--------|----------|-------------|
| **Generation** | 32 bytes random (256 bits) | UUID v4 (122 bits) |
| **Storage** | SHA-256 hash in DB | Plaintext in memory |
| **Transmission** | Cookie (HttpOnly, Secure) | URL query parameter |
| **Lifetime** | 60 days sliding | 10 minutes fixed |
| **Reuse** | Unlimited (until revoked) | Single-use only |

### Session Types

```rust
// crates/loom-server-auth/src/types.rs:209-223
pub enum SessionType {
    Web,     // Browser cookie sessions
    Cli,     // CLI access tokens (lt_ prefix)
    VsCode,  // VS Code extension tokens
}
```

Each type uses the same session infrastructure but may have different token formats:
- **Web**: Cookie-based, 60-day sliding
- **CLI/VsCode**: Bearer token (`lt_` prefix), stored in `access_tokens` table

## Implications

1. **Server Restart Behavior**: Sessions survive restarts (database), OAuth flows in progress are interrupted (memory). Users simply restart the login flow.

2. **Horizontal Scaling**: Sessions work across multiple server instances (shared database). OAuth state requires sticky sessions or Redis for multi-instance deployments.

3. **Security Trade-offs**: 
   - Sessions: Longer-lived, require explicit revocation, stored hashed
   - OAuth state: Short-lived, auto-consumed, plaintext but ephemeral

4. **Cleanup Frequency**: 
   - Sessions: Less frequent (hourly), database DELETE
   - OAuth state: More frequent (15 min), memory cleanup

5. **Audit Trail**: Sessions have full audit logging (creation, revocation, expiry). OAuth state has minimal logging (just debug traces).

6. **Token Rotation**: Sessions use sliding expiry (implicit rotation). OAuth state is single-use (no rotation needed).

## Key Files

- `crates/loom-server-auth/src/session.rs` - Session struct and token generation
- `crates/loom-server-session/src/lib.rs` - Session creation service
- `crates/loom-server-db/src/session.rs` - Database persistence for sessions
- `crates/loom-server/src/oauth_state.rs` - In-memory OAuth state store
- `crates/loom-server/src/jobs/session_cleanup.rs` - Database cleanup job
- `crates/loom-server/src/jobs/oauth_state_cleanup.rs` - Memory cleanup job
- `crates/loom-server/src/routes/sessions.rs` - Session management API
- `crates/loom-server/src/auth_middleware.rs` - Session validation and extension

## Follow-up Questions

- [ ] How does impersonation affect API key authentication vs session authentication?
- [ ] How does the ABAC engine handle impersonation for organization/team-level permissions?
