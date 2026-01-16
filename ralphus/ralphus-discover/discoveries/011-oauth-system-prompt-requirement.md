# Discovery 011: OAuth System Prompt Requirement for Claude Max

> Category: Follow-up (from Discovery 006)
> Discovered: 2026-01-16
> Confidence: High

## Answer

Anthropic enforces a **content-based validation** for OAuth tokens used with premium models (Opus, Sonnet). The system prompt in the request body **MUST** start with the exact phrase: `"You are Claude Code, Anthropic's official CLI for Claude."` — case-sensitive, punctuation-sensitive. Without this prefix, OAuth requests to Opus/Sonnet fail with: "This credential is only authorized for use with Claude Code".

This is NOT based on headers, User-Agent, or network fingerprinting — it's purely **system prompt content validation**. Haiku models work without this prefix, but Opus and Sonnet require it.

Loom automatically handles this by calling `with_oauth_system_prompt()` on every OAuth request, which prepends the required prefix to any existing system prompt (or sets it if none exists).

## Evidence

### The Magic Phrase

```rust
// crates/loom-server-llm-anthropic/src/auth/scheme.rs:47-48
pub const OAUTH_REQUIRED_SYSTEM_PROMPT_PREFIX: &str =
    "You are Claude Code, Anthropic's official CLI for Claude.";
```

### Automatic Application in Client

```rust
// crates/loom-server-llm-anthropic/src/client.rs:236-239
// Apply OAuth system prompt prefix for Opus/Sonnet access
if self.config.auth.is_oauth() {
    anthropic_request = anthropic_request.with_oauth_system_prompt();
}
```

This is applied in both `complete()` (line 236-239) and `complete_streaming()` (line 280-283).

### Implementation Logic

```rust
// crates/loom-server-llm-anthropic/src/types.rs:119-134
pub fn with_oauth_system_prompt(mut self) -> Self {
    use crate::auth::OAUTH_REQUIRED_SYSTEM_PROMPT_PREFIX;

    self.system = Some(match self.system {
        Some(existing) => {
            // Only prepend if not already present
            if existing.starts_with(OAUTH_REQUIRED_SYSTEM_PROMPT_PREFIX) {
                existing
            } else {
                format!("{OAUTH_REQUIRED_SYSTEM_PROMPT_PREFIX} {existing}")
            }
        }
        None => OAUTH_REQUIRED_SYSTEM_PROMPT_PREFIX.to_string(),
    });
    self
}
```

**Key behaviors:**
1. If no system prompt exists → sets the magic phrase as the entire system prompt
2. If system prompt exists but doesn't start with magic phrase → prepends it
3. If system prompt already starts with magic phrase → leaves it unchanged (idempotent)

### Model-Specific Behavior

| Model | System Prompt Required? |
|-------|------------------------|
| `claude-opus-4-5-*` | ✅ Yes |
| `claude-sonnet-4-*` | ✅ Yes |
| `claude-haiku-*` | ❌ No (works without prefix) |

### Required Headers (in addition to system prompt)

```rust
// crates/loom-server-llm-anthropic/src/auth/scheme.rs:24-25
pub const OAUTH_COMBINED_BETA_HEADERS: &str =
    "oauth-2025-04-20,interleaved-thinking-2025-05-14,context-management-2025-06-27";

// Also required:
// Authorization: Bearer <access_token>
// anthropic-dangerous-direct-browser-access: true
// User-Agent: claude-cli/2.0.76 (external, sdk-cli)
```

### Test Coverage

Three tests verify the behavior:

1. `test_oauth_system_prompt_sets_when_none` — Sets prefix when no system prompt exists
2. `test_oauth_system_prompt_prepends_to_existing` — Prepends to existing system prompt
3. `test_oauth_system_prompt_does_not_duplicate` — Idempotent when already present

### What Fails

From `specs/anthropic-oauth-pool.md`:
- ❌ No system prompt at all
- ❌ Phrase appears after other content: `"You are a helpful assistant. You are Claude Code..."`
- ❌ Case variations: `"you are claude code..."`
- ❌ Missing period: `"You are Claude Code, Anthropic's official CLI for Claude"`
- ❌ Shortened version: `"You are Claude Code."`

### External Reference

The implementation references: https://github.com/nsxdavid/anthropic-max-router

This is a community-discovered workaround that Loom has incorporated.

**Key Files:**
- `crates/loom-server-llm-anthropic/src/auth/scheme.rs` — Defines the constant and documents the requirement
- `crates/loom-server-llm-anthropic/src/types.rs` — Implements `with_oauth_system_prompt()`
- `crates/loom-server-llm-anthropic/src/client.rs` — Applies the prefix for OAuth requests
- `specs/anthropic-oauth-pool.md` — Full documentation of the requirement

## Implications

1. **Transparent to Users**: Loom handles this automatically — users don't need to know about the magic phrase
2. **Preserves Custom Prompts**: The prefix is prepended, not replaced, so custom system prompts still work
3. **Model-Aware**: Only matters for Opus/Sonnet; Haiku works regardless (but applying it is harmless)
4. **Fragile Dependency**: This is an undocumented Anthropic behavior that could change without notice
5. **API Key Unaffected**: Only OAuth tokens require this; API key authentication works without it

## Follow-up Questions

- [ ] How are streaming responses handled when errors occur mid-stream?
- [ ] How does the LLM proxy pattern work in detail? (ProxyLlmClient, SSE streaming)
