# Discovery 007: Admin API Pool Management

## Question
How does the admin API manage pool accounts at runtime?

## Answer
Loom provides a **REST API** for system administrators to dynamically manage Claude Max OAuth accounts in the pool. The API supports listing accounts with status, adding new accounts via OAuth flow, and removing accounts—all without server restart (hot-reload).

## Evidence

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/admin/anthropic/accounts` | List accounts with status |
| `POST` | `/api/admin/anthropic/oauth/initiate` | Start OAuth flow, get auth URL |
| `POST` | `/api/admin/anthropic/oauth/complete` | Submit code, add account to pool |
| `DELETE` | `/api/admin/anthropic/accounts/{id}` | Remove account from pool |

All endpoints require `SystemRole::Admin` (system administrator).

### OAuth Flow (Two-Step)

Since Anthropic's public OAuth client doesn't support custom redirect URIs, the flow is manual:

```
1. Admin calls POST /api/admin/anthropic/oauth/initiate
   → Returns { redirect_url, state }

2. Admin opens redirect_url in browser, authorizes on claude.ai
   → Anthropic displays authorization code on their page

3. Admin copies code and calls POST /api/admin/anthropic/oauth/complete
   → Body: { code, state }
   → Server exchanges code for tokens
   → Server adds account to pool (hot-reload)
   → Returns { account_id: "claude-max-1736123456" }
```

### Initiate OAuth Implementation

```rust
pub async fn initiate_oauth(...) -> impl IntoResponse {
    // Generate PKCE challenge
    let pkce = Pkce::generate();
    let oauth_state = generate_state();

    // Build authorization URL
    let mut auth_url = Url::parse("https://claude.ai/oauth/authorize")?;
    params.append_pair("client_id", CLIENT_ID);
    params.append_pair("scope", "user:inference user:profile user:sessions:claude_code");
    params.append_pair("code_challenge", &pkce.challenge);
    params.append_pair("code_challenge_method", "S256");
    params.append_pair("state", &oauth_state);

    // Store state for validation
    state.oauth_state_store.store(oauth_state.clone(), ...).await;

    Json(InitiateOAuthResponse {
        redirect_url: auth_url.to_string(),
        state: oauth_state,
    })
}
```

### Complete OAuth Implementation

```rust
pub async fn complete_oauth(...) -> impl IntoResponse {
    // Validate state
    let entry = state.oauth_state_store
        .validate_and_consume(&body.state, ANTHROPIC_ADMIN_PROVIDER)
        .await?;

    // Exchange code for tokens
    let exchange_result = exchange_code(code, &body.state, &verifier).await?;
    let (access, refresh, expires) = match exchange_result {
        ExchangeResult::Success { access, refresh, expires } => (access, refresh, expires),
        ExchangeResult::Failed { error } => return Err(...),
    };

    // Generate unique account ID
    let account_id = format!("claude-max-{}", chrono::Utc::now().timestamp());
    let credentials = OAuthCredentials::new(
        SecretString::new(refresh),
        SecretString::new(access),
        expires,
    );

    // Add to pool (hot-reload)
    llm_service.add_anthropic_account(account_id.clone(), credentials).await?;

    Json(AddAccountResponse { account_id })
}
```

### List Accounts Response

```json
{
  "accounts": [
    {
      "id": "claude-max-1736123456",
      "status": "available",
      "expires_at": "2026-01-16T15:30:00Z"
    },
    {
      "id": "claude-max-1736123789",
      "status": "cooling_down",
      "cooldown_remaining_secs": 3600,
      "last_error": "Usage limit exceeded"
    }
  ],
  "summary": {
    "total": 2,
    "available": 1,
    "cooling_down": 1,
    "disabled": 0
  }
}
```

### Remove Account Implementation

```rust
pub async fn remove_account(...) -> impl IntoResponse {
    // Validate admin role
    if !current_user.user.is_system_admin {
        return Err(StatusCode::FORBIDDEN);
    }

    // Remove from pool and credential file
    llm_service.remove_anthropic_account(&id).await?;

    Json(RemoveAccountResponse { account_id: id })
}
```

### Hot-Reload via LlmService

The `LlmService` exposes methods that delegate to `AnthropicPool`:

```rust
impl LlmService {
    pub async fn add_anthropic_account(
        &self,
        account_id: String,
        credentials: OAuthCredentials,
    ) -> Result<(), LlmError>;

    pub async fn remove_anthropic_account(
        &self,
        account_id: &str,
    ) -> Result<(), LlmError>;

    pub async fn anthropic_account_details(&self) -> Option<Vec<AccountDetails>>;

    pub fn is_anthropic_oauth_pool(&self) -> bool;
}
```

### Credential Persistence

Accounts are persisted to a JSON file:

```json
{
  "claude-max-1736123456": {
    "type": "oauth",
    "refresh": "rt_abc123...",
    "access": "at_xyz789...",
    "expires": 1735500000000
  }
}
```

- Default path: `/var/lib/loom-server/anthropic-credentials.json`
- Permissions: `0600` (owner read/write only)
- Tokens wrapped in `SecretString` (auto-redacts in logs)

### Security

1. **Admin-only access**: All endpoints require `is_system_admin = true`
2. **PKCE flow**: Prevents authorization code interception
3. **State validation**: Prevents CSRF attacks
4. **No token exposure**: Health/list endpoints never expose tokens
5. **Audit logging**: All operations logged with actor ID

## Implications

1. **Zero-downtime management**: Add/remove accounts without restarting server
2. **Web UI integration**: Designed for loom-web admin panel
3. **Observable**: Account status visible via list endpoint
4. **Secure**: OAuth best practices (PKCE, state validation)
5. **Persistent**: Credentials survive server restarts

## Key Files

- `crates/loom-server/src/routes/admin_anthropic.rs` - Admin API routes
- `crates/loom-server-llm-anthropic/src/pool.rs` - Pool add/remove methods
- `crates/loom-server-api/src/admin.rs` - Request/response types
- `specs/anthropic-max-pool-management.md` - Design specification

## Follow-up Questions

- [ ] How does the web UI (loom-web) integrate with these admin endpoints?
- [ ] How is the OAuth state store implemented for CSRF protection?
