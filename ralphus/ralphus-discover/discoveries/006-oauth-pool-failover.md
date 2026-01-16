# Discovery 006: OAuth Pool Failover

## Question
How does the OAuth pool failover work when quota is exceeded?

## Answer
Loom implements an **AnthropicPool** that manages multiple Claude Pro/Max OAuth subscriptions with **automatic failover**. When an account hits its 5-hour rolling quota limit, it's marked as "cooling down" and the pool automatically selects the next available account using round-robin selection.

## Evidence

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            AnthropicPool                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │ Account 1       │  │ Account 2       │  │ Account 3       │              │
│  │ claude-max-1    │  │ claude-max-2    │  │ claude-max-3    │              │
│  │ ✓ Available     │  │ ⏳ Cooling Down │  │ ✓ Available     │              │
│  │                 │  │ (1h 30m left)   │  │                 │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
│           │                                         │                        │
│           └──────────────┬──────────────────────────┘                        │
│                          ▼                                                   │
│                 ┌─────────────────┐                                          │
│                 │ Account Selector│                                          │
│                 │ (Round Robin)   │                                          │
│                 └─────────────────┘                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Account States

```rust
pub enum AccountStatus {
    /// Account is ready to use
    Available,
    /// Account hit quota, cooling down until specified instant
    CoolingDown { until: Instant },
    /// Account permanently disabled (e.g., invalid credentials)
    Disabled,
}
```

### Error Classification Triggers Failover

The pool uses error classification from Discovery #5 to determine failover behavior:

| Error Type | Action |
|------------|--------|
| `QuotaExceeded` (429 + quota message) | Mark account `CoolingDown`, failover to next |
| `Permanent` (401, 403) | Mark account `Disabled`, failover to next |
| `Transient` (500, 502, etc.) | Retry on same account (via loom-common-http) |

### Failover Flow in LlmClient Implementation

```rust
#[async_trait]
impl LlmClient for AnthropicPool {
    async fn complete(&self, request: LlmRequest) -> Result<LlmResponse, LlmError> {
        // 1. Select available account (round-robin)
        let index = self.select_account_index().await
            .ok_or(LlmError::RateLimited { retry_after_secs: None })?;

        // 2. Attempt request
        match account.client.complete(request.clone()).await {
            Ok(response) => Ok(response),
            Err(e) => {
                let error_msg = e.to_string();
                
                // 3. Classify error and update account state
                if is_quota_message(&error_msg) {
                    self.mark_cooling(index, &error_msg).await;
                } else if is_permanent_auth_message(&error_msg) {
                    self.mark_disabled(index, &error_msg).await;
                }
                Err(e)
            }
        }
    }
}
```

### Account Selection Algorithm

```rust
async fn select_account_index(&self) -> Option<usize> {
    let now = Instant::now();
    
    // 1. Refresh expired cooldowns
    for runtime in &mut state.runtimes {
        if let AccountStatus::CoolingDown { until } = runtime.status {
            if now >= until {
                runtime.status = AccountStatus::Available;
                runtime.last_error = None;
            }
        }
    }
    
    // 2. Round-robin selection
    let start = state.next_index;
    for i in 0..n {
        let idx = (start + i) % n;
        if state.runtimes[idx].status == AccountStatus::Available {
            state.next_index = (idx + 1) % n;
            return Some(idx);
        }
    }
    None  // All accounts exhausted
}
```

### Pool Configuration

```rust
pub struct AnthropicPoolConfig {
    /// How long to cool down an account after quota exhaustion (default: 2 hours)
    pub cooldown: Duration,
    /// Account selection strategy
    pub strategy: AccountSelectionStrategy,
}

pub enum AccountSelectionStrategy {
    RoundRobin,      // Distribute load across accounts
    FirstAvailable,  // Always use first available
}
```

### Quota Detection

The pool detects Claude's 5-hour rolling limit via error message patterns:

```rust
pub fn is_quota_message(msg: &str) -> bool {
    let lower = msg.to_ascii_lowercase();
    lower.contains("5-hour")
        || lower.contains("5 hour")
        || lower.contains("rolling window")
        || lower.contains("usage limit for your plan")
        || lower.contains("subscription usage limit")
}
```

### Proactive Token Refresh

The pool spawns a background task to refresh OAuth tokens before they expire:

```rust
pub fn spawn_refresh_task(
    self: Arc<Self>,
    interval: Duration,    // Check interval
    threshold: Duration,   // Refresh if expires within threshold
) -> tokio::task::JoinHandle<()>
```

If token refresh fails, the account is marked `Disabled`.

### Health Reporting

```rust
pub struct PoolStatus {
    pub accounts_total: usize,
    pub accounts_available: usize,
    pub accounts_cooling: usize,
    pub accounts_disabled: usize,
    pub accounts: Vec<AccountHealthInfo>,
}
```

Exposed via `/health` endpoint:
```json
{
  "mode": "oauth_pool",
  "pool": {
    "accounts_total": 3,
    "accounts_available": 1,
    "accounts_cooling": 2,
    "accounts_disabled": 0
  }
}
```

## Implications

1. **High Availability**: Multiple accounts provide redundancy against quota exhaustion
2. **Automatic Recovery**: Cooling accounts automatically become available after cooldown expires
3. **Graceful Degradation**: Service continues with reduced capacity when some accounts are cooling
4. **Observable**: Health endpoint shows real-time pool status for monitoring
5. **Dynamic Management**: Accounts can be added/removed at runtime via admin API

## Key Files

- `crates/loom-server-llm-anthropic/src/pool.rs` - Pool implementation
- `crates/loom-server-llm-anthropic/src/client.rs` - Error classification
- `specs/anthropic-oauth-pool.md` - Design specification
- `specs/anthropic-max-pool-management.md` - Admin UI specification

## Follow-up Questions

- [ ] How does the admin API manage pool accounts at runtime?
- [ ] How does the OAuth system prompt requirement work for Claude Max?
