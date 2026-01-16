# Discovery: How does the impersonation system work end-to-end?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

Loom implements a comprehensive, auditable user impersonation system that allows system administrators to assume another user's identity for debugging and support purposes. The system is designed around three core principles: **strict authorization** (only system admins can impersonate), **full auditability** (every impersonation action is logged with the real admin's identity preserved), and **session-based state** (impersonation is tracked in a dedicated database table, not in cookies or tokens).

The impersonation flow works as follows: An admin initiates impersonation via `POST /api/admin/users/{id}/impersonate` with a required `reason` field. This creates a record in the `impersonation_sessions` table linking the admin to the target user. On subsequent requests, the system checks for an active impersonation session and populates `CurrentUser.impersonating_as`. Code that needs to check permissions uses `effective_user_id()` (returns the impersonated user), while audit logging uses `actor_user_id()` (returns the real admin). The admin can end impersonation via `POST /api/admin/impersonate/stop`, which sets `ended_at` on the session record.

The frontend displays a prominent warning banner when impersonation is active, showing both the admin's name and the impersonated user's name, with a button to stop impersonation immediately.

## Evidence

### Core Data Structures

**ImpersonationSession** (`crates/loom-server-auth/src/admin.rs`):
```rust
pub struct ImpersonationSession {
    pub id: Uuid,
    pub admin_user_id: UserId,    // The real admin
    pub target_user_id: UserId,   // The user being impersonated
    pub started_at: DateTime<Utc>,
    pub ended_at: Option<DateTime<Utc>>,
    pub reason: Option<String>,   // Required for audit trail
}
```

**CurrentUser** (`crates/loom-server-auth/src/middleware.rs`):
```rust
pub struct CurrentUser {
    pub user: User,                        // The authenticated user (admin)
    pub session_id: Option<SessionId>,
    pub api_key_id: Option<Uuid>,
    pub impersonating_as: Option<UserId>,  // Target user if impersonating
}

impl CurrentUser {
    /// The effective user ID (the one being impersonated, or self).
    /// Use this when checking permissions for resource access.
    pub fn effective_user_id(&self) -> &UserId {
        self.impersonating_as.as_ref().unwrap_or(&self.user.id)
    }

    /// The real actor (admin doing the impersonation, or self).
    /// Use this for audit logging to track who actually performed an action.
    pub fn actor_user_id(&self) -> &UserId {
        &self.user.id
    }
}
```

### Database Schema

**Migration 015** (`crates/loom-server/migrations/015_impersonation_sessions.sql`):
```sql
CREATE TABLE IF NOT EXISTS impersonation_sessions (
    id TEXT PRIMARY KEY NOT NULL,
    admin_user_id TEXT NOT NULL REFERENCES users(id),
    target_user_id TEXT NOT NULL REFERENCES users(id),
    reason TEXT NOT NULL,
    started_at TEXT NOT NULL,
    ended_at TEXT,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_impersonation_admin ON impersonation_sessions(admin_user_id, ended_at);
CREATE INDEX IF NOT EXISTS idx_impersonation_target ON impersonation_sessions(target_user_id, ended_at);
```

### API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/admin/impersonate/state` | GET | Get current impersonation status |
| `/api/admin/users/{id}/impersonate` | POST | Start impersonating a user |
| `/api/admin/impersonate/stop` | POST | End current impersonation |

### Authorization Check

```rust
// crates/loom-server-auth/src/admin.rs
pub fn check_can_impersonate(actor: &User) -> Result<(), AuthError> {
    if !actor.is_system_admin() {
        return Err(AuthError::Forbidden(
            "Only system admins can impersonate users".into(),
        ));
    }
    Ok(())
}
```

### Audit Events

Impersonation actions are logged with dedicated event types:
- `AuditEventType::ImpersonationStarted` — When admin begins impersonation
- `AuditEventType::ImpersonationEnded` — When admin stops impersonation

All audit entries include an `impersonating_user_id` field that tracks the real admin when actions are performed during impersonation.

**Key Files:**
- `crates/loom-server-auth/src/admin.rs` — ImpersonationSession struct and permission checks
- `crates/loom-server-auth/src/middleware.rs` — CurrentUser with effective_user_id/actor_user_id
- `crates/loom-server/src/routes/admin.rs` — HTTP handlers for impersonation endpoints
- `crates/loom-server-db/src/session.rs` — Database operations for impersonation sessions
- `crates/loom-server-api/src/admin.rs` — API request/response DTOs
- `crates/loom-server/migrations/015_impersonation_sessions.sql` — Database schema
- `web/loom-web/src/lib/ui/ImpersonationBanner.svelte` — Frontend warning banner

## Implications

- **Permission checks must use `effective_user_id()`**: When checking if a user can access a resource, use `current_user.effective_user_id()` to respect impersonation.

- **Audit logging must use `actor_user_id()`**: When logging who performed an action, use `current_user.actor_user_id()` to capture the real human actor.

- **Impersonation is session-based, not token-based**: The impersonation state is stored in the database, not in the session cookie. This means the admin's session token doesn't change during impersonation.

- **Only one active impersonation per admin**: An admin must stop their current impersonation before starting a new one (409 Conflict returned otherwise).

- **Cannot impersonate yourself**: The system explicitly prevents self-impersonation (400 Bad Request).

- **Reason is required**: Every impersonation must include a justification for the audit trail.

## Follow-up Questions

- [ ] How does impersonation affect API key authentication vs session authentication?
- [ ] How does the ABAC engine handle impersonation for organization/team-level permissions?

## Related Discoveries

- [[007-admin-api-pool-management]] — Admin API patterns
- [[008-web-ui-admin-integration]] — Web UI admin integration
- [[009-oauth-state-store-csrf]] — Session management patterns
