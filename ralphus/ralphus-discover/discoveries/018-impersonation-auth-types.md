# Discovery: How does impersonation affect API key authentication vs session authentication?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Impersonation is session-only and does NOT work with API key authentication.** This is a fundamental architectural decision that creates a clear separation between interactive admin sessions (where impersonation is needed for support) and programmatic API access (where impersonation would be a security risk).

The impersonation system requires:
1. **Session authentication** - The admin must be logged in via web session (cookie-based)
2. **`system_admin` role** - Only users with `is_system_admin = true` can impersonate
3. **Database-backed session tracking** - Impersonation sessions are stored in `impersonation_sessions` table

API keys, by contrast:
1. Authenticate as the **user who created the key** (the `created_by` field)
2. Have **no impersonation capability** - the `CurrentUser::from_api_key()` constructor never sets `impersonating_as`
3. Are scoped to specific permissions via `ApiKeyScope` (threads:read, llm:use, etc.)

This design prevents a compromised API key from being used to impersonate arbitrary users - a critical security boundary.

## Evidence

### CurrentUser Construction Paths

The `CurrentUser` struct has three constructors, each for a different auth type:

```rust
// crates/loom-server-auth/src/middleware.rs:61-90

impl CurrentUser {
    /// Create a new CurrentUser from a session-based authentication.
    pub fn from_session(user: User, session_id: SessionId) -> Self {
        Self {
            user,
            session_id: Some(session_id),
            api_key_id: None,
            impersonating_as: None,  // <-- Never set during construction
        }
    }

    /// Create a new CurrentUser from an API key authentication.
    pub fn from_api_key(user: User, api_key_id: Uuid) -> Self {
        Self {
            user,
            session_id: None,
            api_key_id: Some(api_key_id),
            impersonating_as: None,  // <-- Never set, no impersonation for API keys
        }
    }

    /// Create a new CurrentUser from an access token (CLI/VS Code).
    pub fn from_access_token(user: User) -> Self {
        Self {
            user,
            session_id: None,
            api_key_id: None,
            impersonating_as: None,  // <-- Never set during construction
        }
    }
}
```

### Impersonation is Applied Post-Authentication (Session Only)

The `with_impersonation()` method exists but is **only used in tests**:

```rust
// crates/loom-server-auth/src/middleware.rs:92-96

/// Set the user being impersonated.
pub fn with_impersonation(mut self, impersonated_user_id: UserId) -> Self {
    self.impersonating_as = Some(impersonated_user_id);
    self
}
```

Grep for actual usage shows it's only in test code:
```
crates/loom-server-auth/src/middleware.rs:441:  .with_impersonation(impersonated_id);  // TEST
crates/loom-server-auth/src/middleware.rs:461:  .with_impersonation(impersonated_id);  // TEST
crates/loom-server-auth/src/middleware.rs:472:  .with_impersonation(impersonated_id);  // TEST
```

### Impersonation State is Queried, Not Applied to Auth Context

The admin routes query impersonation state but don't modify the `CurrentUser`:

```rust
// crates/loom-server/src/routes/admin.rs:628-631

match state
    .session_repo
    .get_active_impersonation_session(&current_user.user.id)
    .await
{
    Ok(Some((_session_id, target_user_id))) => {
        // Returns state info, doesn't modify current_user
    }
}
```

### API Key Authentication Flow

API keys authenticate as the key creator, with no impersonation layer:

```rust
// crates/loom-server/src/auth_middleware.rs:256-306

async fn authenticate_api_key(
    api_key_token: &str,
    api_key_repo: &Arc<ApiKeyRepository>,
    user_repo: &Arc<UserRepository>,
) -> Option<AuthContext> {
    // ... validate token hash ...
    
    // Get the user who created the key
    let user = match user_repo.get_user_by_id(&api_key.created_by).await {
        Ok(Some(user)) => user,
        // ...
    };

    // No impersonation check here - just creates CurrentUser from key creator
    let current_user = CurrentUser::from_api_key(user, api_key.id.into_inner());
    Some(AuthContext::authenticated(current_user))
}
```

### Impersonation Requires system_admin Check

The impersonation endpoints explicitly check `is_system_admin`:

```rust
// crates/loom-server/src/routes/admin.rs:758-771

if !current_user.user.is_system_admin {
    tracing::warn!(
        actor_id = %current_user.user.id,
        target_id = %user_id,
        "Unauthorized impersonation attempt"
    );
    return (
        StatusCode::FORBIDDEN,
        Json(AdminErrorResponse {
            error: "forbidden".to_string(),
            message: t(locale, "server.api.admin.system_admin_required").to_string(),
        }),
    )
        .into_response();
}
```

## Implications

1. **Security Boundary**: API keys cannot be used for impersonation attacks. Even if an API key is compromised, the attacker can only act as the key's creator, not impersonate other users.

2. **Admin Workflow**: System admins must use the web UI (session auth) to impersonate users for support. They cannot script impersonation via API keys.

3. **Audit Trail**: Impersonation sessions are tracked in the database with the real admin's ID, ensuring accountability. The `impersonating_user_id` field in audit logs captures who was really acting.

4. **Incomplete Implementation**: The `with_impersonation()` method exists but is never called in production code. The impersonation state is queried but not applied to the `CurrentUser` context. This suggests impersonation may be partially implemented or the design expects clients to handle impersonation state separately.

5. **Three Auth Types, One Impersonation Path**:
   - **Session** (cookie): Can impersonate (if system_admin)
   - **API Key** (lk_*): Cannot impersonate
   - **Access Token** (lt_*): Cannot impersonate

## Follow-up Questions

- [ ] How does the web UI use impersonation state to modify its behavior when an admin is impersonating?
- [ ] Why is `with_impersonation()` never called in production - is impersonation state meant to be client-side only?
