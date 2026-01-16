# Discovery 008: Web UI Admin Integration

## Question
How does the web UI (loom-web) integrate with admin endpoints?

## Answer
The loom-web frontend provides a **complete admin dashboard** at `/admin` with dedicated pages for managing Anthropic accounts, users, jobs, logs, and audit trails. The integration follows a **role-based access pattern** where the admin navigation and features are only visible to users with `system_admin` in their `global_roles` array. API calls use the same session cookies as regular authenticated requests, with authorization enforced server-side.

## Evidence

### Admin Route Structure

```
web/loom-web/src/routes/(app)/admin/
├── +page.svelte                    # Admin dashboard (health, logs preview)
├── anthropic-accounts/+page.svelte # OAuth pool management
├── users/+page.svelte              # User management
├── jobs/+page.svelte               # Background job monitoring
├── logs/+page.svelte               # Server log viewer
└── audit-logs/+page.svelte         # Security audit trail
```

### Role-Based Navigation

The app layout conditionally renders the admin link based on user roles:

```svelte
// web/loom-web/src/routes/(app)/+layout.svelte
const isSystemAdmin = $derived(data.user?.global_roles?.includes('system_admin') ?? false);

{#if isSystemAdmin}
    <a href="/admin" class="nav-link nav-link-admin">
        {i18n._('nav.admin')}
    </a>
{/if}
```

The admin link is styled distinctly with `color: var(--color-warning)` to indicate elevated access.

### API Client Layer

Admin API calls are organized in two locations:

**1. Anthropic-specific API (`$lib/api/anthropic.ts`):**
```typescript
export async function listAnthropicAccounts(): Promise<AnthropicAccountsResponse> {
    const response = await fetch('/api/admin/anthropic/accounts', {
        headers: { 'Content-Type': 'application/json' },
    });
    // ...
}

export async function initiateAnthropicOAuth(redirectAfter?: string): Promise<InitiateOAuthResponse>
export async function completeAnthropicOAuth(code: string, state: string): Promise<AddAccountResponse>
export async function removeAnthropicAccount(accountId: string): Promise<void>
```

**2. General admin API (`$lib/api/client.ts`):**
```typescript
class ApiClient {
    async getImpersonationState(): Promise<ImpersonationState>
    async startImpersonation(userId: string, reason: string): Promise<ImpersonateResponse>
    async stopImpersonation(): Promise<StopImpersonationResponse>
    async listAdminUsers(params): Promise<AdminUserListResponse>
    async updateUserRoles(userId: string, data: UpdateUserRolesRequest): Promise<UpdateUserRolesResponse>
    async deleteUser(userId: string): Promise<DeleteUserResponse>
}
```

### TypeScript Types Mirror Backend

```typescript
// web/loom-web/src/lib/api/types.ts
interface AdminUser {
    is_system_admin: boolean;
    // ...
}

interface UpdateUserRolesRequest {
    is_system_admin?: boolean;
}

// web/loom-web/src/lib/api/anthropic.ts
export type AnthropicAccountStatus = 'available' | 'cooling_down' | 'disabled';

export interface AnthropicAccount {
    id: string;
    status: AnthropicAccountStatus;
    cooldown_remaining_secs?: number;
    last_error?: string;
    expires_at?: string;
}
```

### Admin Dashboard Features

| Section | Endpoint | Features |
|---------|----------|----------|
| **Health** | `/health` | Component status, latency, version |
| **Logs** | `/api/admin/logs` + SSE stream | Real-time log viewer with filtering |
| **Users** | `/api/admin/users` | List, search, role management, impersonation |
| **Anthropic** | `/api/admin/anthropic/*` | OAuth pool management (add/remove accounts) |
| **Jobs** | `/api/admin/jobs` | Background job status, manual triggers |
| **Audit** | `/api/admin/audit-logs` | Security event history |

### OAuth Flow in UI

The Anthropic accounts page implements the two-step OAuth flow:

```svelte
// 1. Initiate OAuth - opens claude.ai in new tab
async function handleAddAccount() {
    const response = await initiateAnthropicOAuth('/admin/anthropic-accounts');
    oauthState = response.state;
    window.open(response.redirect_url, '_blank');
    showCodeModal = true;  // Show modal to enter code
}

// 2. Complete OAuth - user pastes code from claude.ai
async function handleSubmitCode() {
    const response = await completeAnthropicOAuth(authCode.trim(), oauthState);
    successMessage = i18n._('admin.anthropic.account_added', { id: response.account_id });
    await loadAccounts();  // Refresh list
}
```

### Real-Time Log Streaming

The admin dashboard uses Server-Sent Events for live logs:

```svelte
eventSource = new EventSource('/api/admin/logs/stream', { withCredentials: true });

eventSource.onmessage = (event) => {
    const entry: LogEntry = JSON.parse(event.data);
    logs = [...logs, entry].slice(-100);  // Keep last 100
};
```

### Impersonation Banner

System admins can impersonate users for debugging. When active, a banner appears:

```svelte
// Layout checks impersonation state on load
$effect(() => {
    if (isSystemAdmin) {
        loadImpersonationState();
    }
});

{#if impersonationState?.is_impersonating}
    <ImpersonationBanner impersonation={impersonationState} onStop={loadImpersonationState} />
{/if}
```

### Authorization Flow

```
┌─────────────┐     Cookie Auth     ┌─────────────┐     Role Check     ┌─────────────┐
│  loom-web   │ ──────────────────▶ │ loom-server │ ─────────────────▶ │  Database   │
│  /admin/*   │                     │  Middleware │                     │  users.     │
│             │                     │             │                     │  is_system_ │
│             │ ◀────────────────── │             │ ◀───────────────── │  admin      │
│             │   200 OK / 403      │             │   true/false       │             │
└─────────────┘                     └─────────────┘                     └─────────────┘
```

**Key Points:**
1. Frontend only hides UI elements - no security enforcement
2. All admin endpoints require `is_system_admin = true` server-side
3. Session cookies provide authentication
4. 403 Forbidden returned for non-admins

### i18n Support

All admin UI strings are internationalized:

```svelte
{i18n._('admin.anthropic.title')}
{i18n._('admin.anthropic.add_account')}
{i18n._('admin.anthropic.status.available')}
```

## Implications

1. **Client-side role check is cosmetic**: The `isSystemAdmin` check in the layout only hides the nav link. Server enforces authorization.

2. **No dedicated admin layout**: Admin pages use the same `(app)` layout as regular pages, just with conditional nav.

3. **SSE for real-time updates**: Log streaming uses EventSource, not WebSocket. Simple but effective.

4. **OAuth flow is manual**: Users must copy-paste the authorization code because Anthropic's public OAuth client doesn't support custom redirect URIs.

5. **Type duplication**: TypeScript types in frontend mirror Rust types in backend. No code generation - manual sync required.

6. **Impersonation is audited**: All impersonation actions are logged to audit trail.

## Key Files

- `web/loom-web/src/routes/(app)/admin/+page.svelte` - Admin dashboard
- `web/loom-web/src/routes/(app)/admin/anthropic-accounts/+page.svelte` - OAuth pool UI
- `web/loom-web/src/lib/api/anthropic.ts` - Anthropic API client
- `web/loom-web/src/lib/api/client.ts` - General admin API methods
- `web/loom-web/src/lib/api/types.ts` - TypeScript type definitions
- `web/loom-web/src/routes/(app)/+layout.svelte` - Role-based nav rendering

## Follow-up Questions

- [ ] How is the OAuth state store implemented for CSRF protection?
- [ ] How does the impersonation system work end-to-end?
