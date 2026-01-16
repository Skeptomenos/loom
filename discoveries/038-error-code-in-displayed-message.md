# Discovery: Should the error code field be included in the displayed message for certain error types?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Selective inclusion recommended.** Error codes should be included in displayed messages only when they provide actionable context beyond the HTTP status code. The implementation should use a simple heuristic: include the code when it's more specific than the generic HTTP status equivalent.

### Recommended Approach

**Include code for domain-specific errors:**
```
already_member: You are already a member of this organization
last_owner: Cannot remove the last owner from the organization
slug_exists: An organization with this slug already exists
grace_period_expired: The deletion grace period has expired
```

**Omit code for generic errors (redundant with HTTP status):**
```
Version mismatch: expected 5, got 3  (not "conflict: Version mismatch...")
Thread not found: abc123            (not "not_found: Thread not found...")
A database error occurred           (not "internal_error: A database error...")
```

### Implementation

```rust
// In parse_error_response helper (proposed in discovery-021)
fn format_error_message(error: &str, message: &str) -> String {
    // Generic codes that are redundant with HTTP status
    const GENERIC_CODES: &[&str] = &[
        "bad_request",
        "not_found", 
        "conflict",
        "forbidden",
        "unauthorized",
        "internal_error",
        "service_unavailable",
    ];
    
    if GENERIC_CODES.contains(&error) {
        message.to_string()
    } else {
        format!("{}: {}", error, message)
    }
}
```

## Evidence

**Analysis of 582 error code usages across 26 server route files:**

| Category | Examples | Include Code? |
|----------|----------|---------------|
| Generic HTTP equivalents | `not_found`, `bad_request`, `conflict`, `forbidden`, `unauthorized`, `internal_error` | No - redundant |
| Domain-specific state | `already_member`, `last_owner`, `already_revoked`, `already_exists`, `already_handled` | Yes - actionable |
| Validation failures | `invalid_name`, `invalid_url`, `invalid_pattern`, `invalid_scopes`, `invalid_email` | Yes - specific |
| Configuration issues | `not_configured`, `not_implemented` | Yes - actionable |
| Business logic | `slug_exists`, `grace_period_expired`, `cannot_delete_personal`, `pending_request` | Yes - context |
| Auth-specific | `token_validation_failed`, `svid_validation_failed`, `email_not_verified` | Yes - diagnostic |

**Error code distribution (sample):**
- `internal_error`: 89 occurrences - always omit (generic)
- `not_found`: 78 occurrences - always omit (generic)
- `forbidden`: 45 occurrences - always omit (generic)
- `not_configured`: 28 occurrences - include (actionable: "feature X is not configured")
- `already_exists`: 8 occurrences - include (specific: "already_exists: Secret with this name exists")
- `last_owner`: 3 occurrences - include (critical: "last_owner: Cannot remove yourself")

**Current ErrorResponse structure** (`crates/loom-server/src/error.rs:76-83`):
```rust
pub struct ErrorResponse {
    pub error: String,      // Machine-readable code
    pub message: String,    // Human-readable explanation
    // ...
}
```

**Example transformations:**

| Raw JSON | Current Display | Proposed Display |
|----------|-----------------|------------------|
| `{"error":"conflict","message":"Version mismatch: expected 5, got 3"}` | `proxy returned status 409: {"error":"conflict"...}` | `Version mismatch: expected 5, got 3` |
| `{"error":"already_member","message":"User is already a member"}` | `proxy returned status 409: {"error":"already_member"...}` | `already_member: User is already a member` |
| `{"error":"not_configured","message":"GitHub App is not configured"}` | `proxy returned status 503: {"error":"not_configured"...}` | `not_configured: GitHub App is not configured` |

## Implications

1. **User Experience**: Users get cleaner messages for common errors while retaining diagnostic context for domain-specific issues.

2. **Debuggability**: Domain-specific codes help users understand *what kind* of error occurred, not just that an error occurred. "already_member" is more actionable than a generic 409.

3. **Consistency**: The heuristic is simple and deterministic - no per-endpoint configuration needed.

4. **Extensibility**: New domain-specific error codes automatically get included; new generic codes can be added to the exclusion list.

5. **Minimal Complexity**: The implementation is ~10 lines of code in the shared `parse_error_response()` helper.

## Follow-up Questions

None - this completes the error display thread started in discovery-021.

## Related Discoveries

- [[021-cli-json-error-parsing]] - Parent discovery proposing JSON error parsing
- [[022-llmerror-provider-not-configured-variant]] - Related error handling improvements
