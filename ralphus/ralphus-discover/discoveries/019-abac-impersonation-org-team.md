# Discovery: How does the ABAC engine handle impersonation for organization/team-level permissions?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**The ABAC engine does NOT currently handle impersonation for organization/team-level permissions.** This is a significant architectural gap. The `build_subject_attrs()` function always uses `user.user.id` (the admin's ID) rather than `user.effective_user_id()` (the impersonated user's ID). This means when an admin impersonates a user, the ABAC checks still evaluate against the **admin's** org/team memberships, not the impersonated user's.

The impersonation system stores state in the database (`impersonation_sessions` table) and provides `effective_user_id()` and `actor_user_id()` methods on `CurrentUser`, but these are **never used** in the ABAC permission evaluation path. The `build_subject_attrs()` function:
1. Creates `SubjectAttrs::new(user.user.id)` - using the admin's ID
2. Queries org memberships for `user.user.id` - the admin's orgs
3. Queries team memberships for `user.user.id` - the admin's teams

This design has a practical consequence: **system admins bypass ABAC entirely** because `check_global_roles()` returns `true` for `SystemAdmin` before any org/team checks occur. So while impersonation doesn't affect ABAC evaluation, it also doesn't need to - admins have full access regardless.

## Evidence

### build_subject_attrs Uses Admin's ID, Not Effective ID

```rust
// crates/loom-server/src/abac_middleware.rs:556-602

#[instrument(skip(user, org_repo, team_repo), fields(user_id = %user.user.id))]
pub async fn build_subject_attrs(
    user: &CurrentUser,
    org_repo: &OrgRepository,
    team_repo: &TeamRepository,
) -> SubjectAttrs {
    let mut subject = SubjectAttrs::new(user.user.id);  // <-- Uses admin's ID, not effective_user_id()

    if user.user.is_system_admin {
        subject.global_roles.push(GlobalRole::SystemAdmin);
    }
    // ... adds admin's global roles ...

    if let Ok(orgs) = org_repo.list_orgs_for_user(&user.user.id).await {  // <-- Admin's orgs
        for org in orgs {
            if let Ok(Some(membership)) = org_repo.get_membership(&org.id, &user.user.id).await {
                subject.org_memberships.push(OrgMembershipAttr {
                    org_id: membership.org_id,
                    role: membership.role,
                });
            }
        }
    }

    if let Ok(teams) = team_repo.get_teams_for_user(&user.user.id).await {  // <-- Admin's teams
        // ...
    }

    subject
}
```

### effective_user_id() Exists But Is Never Used in ABAC

```rust
// crates/loom-server-auth/src/middleware.rs:103-108

/// The effective user ID (the one being impersonated, or self).
///
/// Use this when checking permissions for resource access.
pub fn effective_user_id(&self) -> &UserId {
    self.impersonating_as.as_ref().unwrap_or(&self.user.id)
}
```

Grep shows `effective_user_id` is only used in tests:
```
./crates/loom-server-auth/src/middleware.rs:106:    pub fn effective_user_id(&self) -> &UserId {
./crates/loom-server-auth/src/middleware.rs:448:    fn effective_user_id_returns_self_when_not_impersonating() {
./crates/loom-server-auth/src/middleware.rs:453:        assert_eq!(current_user.effective_user_id(), &user_id);
./crates/loom-server-auth/src/middleware.rs:457:    fn effective_user_id_returns_impersonated_when_impersonating() {
./crates/loom-server-auth/src/middleware.rs:463:        assert_eq!(current_user.effective_user_id(), &impersonated_id);
```

### SystemAdmin Bypasses All ABAC Checks

```rust
// crates/loom-server-auth/src/abac/engine.rs:47-76

pub fn is_allowed(subject: &SubjectAttrs, action: Action, resource: &ResourceAttrs) -> bool {
    if check_global_roles(subject, action, resource) {
        return true;  // <-- SystemAdmin returns true here, skips org/team checks
    }

    match resource.resource_type {
        ResourceType::Thread => thread::evaluate(subject, action, resource),
        ResourceType::Organization => org::evaluate_org(subject, action, resource),
        // ... other resource types ...
    }
}

fn check_global_roles(subject: &SubjectAttrs, action: Action, _resource: &ResourceAttrs) -> bool {
    if subject.is_system_admin() {
        return true;  // <-- Full access for SystemAdmin
    }

    if subject.is_auditor() {
        return matches!(action, Action::Read);
    }

    false
}
```

### with_impersonation() Never Called in Production

```rust
// crates/loom-server-auth/src/middleware.rs:92-96

/// Set the user being impersonated.
pub fn with_impersonation(mut self, impersonated_user_id: UserId) -> Self {
    self.impersonating_as = Some(impersonated_user_id);
    self
}
```

All usages are in test code only - never in production request handling.

**Key Files:**
- `crates/loom-server/src/abac_middleware.rs` - `build_subject_attrs()` function (lines 556-602)
- `crates/loom-server-auth/src/abac/engine.rs` - `is_allowed()` and `check_global_roles()` (lines 47-76)
- `crates/loom-server-auth/src/middleware.rs` - `effective_user_id()` method (lines 103-108)
- `crates/loom-server-auth/src/abac/types.rs` - `SubjectAttrs` struct definition

## Implications

1. **Impersonation is UI-only, not permission-affecting**: When an admin impersonates a user, they still have full admin permissions. The impersonation is for viewing the UI as that user, not for testing that user's actual permissions.

2. **No "see what they see" for permissions**: If an admin wants to verify what permissions a user has, they cannot impersonate and test - they would need to use a separate permission debugging tool.

3. **Audit trail is preserved**: The `actor_user_id()` method ensures audit logs capture the real admin, even during impersonation. This is the primary use case for impersonation state.

4. **Design decision, not a bug**: The fact that `effective_user_id()` exists but isn't used in ABAC suggests this was a conscious choice. SystemAdmin already has full access, so impersonation doesn't need to affect permissions.

5. **If impersonation-aware ABAC were needed**: The fix would be to modify `build_subject_attrs()` to use `user.effective_user_id()` instead of `user.user.id` for org/team lookups. However, this would require careful consideration of whether admins should lose their admin powers during impersonation.

## Follow-up Questions

- [ ] How does the web UI display resources when an admin is impersonating - does it filter by the impersonated user's permissions or show everything?
- [ ] Is there a permission debugging tool that allows admins to see what a specific user can access without impersonating?

## Related Discoveries

- [[010-impersonation-system]] - Core impersonation architecture
- [[018-impersonation-auth-types]] - Impersonation and auth type interactions
