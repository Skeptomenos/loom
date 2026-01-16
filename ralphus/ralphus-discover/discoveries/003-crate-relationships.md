# Discovery: What is the relationship between loom-core, loom-server, and loom-cli?

> Category: Architecture (Follow-up)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The three crates form a **layered architecture** with clear separation of concerns:

1. **loom-common-core** (foundation layer) - Defines core abstractions, types, and the state machine. Has NO dependencies on other loom crates. Provides `LlmClient` trait, `Agent` struct, `AgentState`/`AgentEvent` enums, `Message`, `ToolDefinition`, and error types.

2. **loom-server** (middle layer) - HTTP server that owns API keys and proxies LLM requests. Depends on `loom-common-core` plus many server-specific crates (`loom-server-*`). Provides endpoints like `/proxy/anthropic/complete`, thread persistence, authentication, and weaver management.

3. **loom-cli** (top layer) - CLI binary that orchestrates user interaction. Depends on `loom-common-core` and uses `loom-server-llm-proxy` to communicate with the server. Never handles API keys directly.

The key architectural insight is the **server-side LLM proxy pattern**: API keys are stored ONLY on the server. The CLI uses `ProxyLlmClient` to make HTTP calls to the server, which then calls the actual LLM providers.

## Evidence

**Dependency Graph (from specs/architecture.md):**
```
loom-core (bottom layer)
    ↑
loom-http (utility layer)
    ↑
loom-llm-anthropic, loom-llm-openai (provider layer, server-only)
    ↑
loom-llm-service (server-side provider abstraction)
    ↑
loom-server (HTTP server with proxy endpoints)
    ↑
loom-llm-proxy (client-side LlmClient via HTTP)
    ↑
loom-tools (tool layer, depends only on loom-core)
    ↑
loom-cli (top layer, orchestrates everything)
```

**loom-common-core exports (lib.rs):**
```rust
pub mod agent;
pub mod config;
pub mod error;
pub mod llm;
pub mod message;
pub mod server_query;
pub mod state;
pub mod tool;
```

**loom-cli dependencies (Cargo.toml:31-41):**
```toml
loom-common-core = { path = "../loom-common-core" }
loom-server-llm-proxy = { path = "../loom-server-llm-proxy" }
loom-common-thread = { path = "../loom-common-thread" }
loom-cli-tools = { path = "../loom-cli-tools" }
```

**loom-server dependencies (Cargo.toml:29):**
```toml
loom-common-core = { path = "../loom-common-core" }
loom-server-llm-service = { path = "../loom-server-llm-service" }
loom-server-llm-anthropic = { path = "../loom-server-llm-anthropic" }
```

**Key Files:**
- `crates/loom-common-core/` — Core abstractions (26 lines in Cargo.toml, minimal deps)
- `crates/loom-server/` — HTTP server (123 lines in Cargo.toml, many deps)
- `crates/loom-cli/` — CLI binary (58 lines in Cargo.toml)
- `specs/architecture.md` — Comprehensive architecture documentation

## Implications

- **Adding new LLM providers**: Only requires changes to server-side crates. CLI automatically gets access via proxy.
- **Security**: API keys never leave the server. Easier credential rotation and audit logging.
- **Testing**: Core types can be tested in isolation. Mock implementations for `LlmClient` trait.
- **Key constraint**: Lower-layer crates never depend on higher-layer crates.
- **Naming convention**: `loom-common-*` for shared code, `loom-server-*` for server-only, `loom-cli-*` for CLI-only.

## Follow-up Questions

- [ ] How does the LLM proxy pattern work in detail? (ProxyLlmClient, SSE streaming)
- [ ] What is the thread system and how does it persist conversations?

## Related Discoveries

- [[001-primary-purpose]] — Establishes the three core principles
- [[002-state-machine-orchestration]] — Details the Agent state machine in loom-common-core
