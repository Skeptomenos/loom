# Discovery: Should parallel execution be gated behind a feature flag or CLI option for gradual rollout?

> Category: Follow-ups
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, parallel execution should be gated behind a CLI option (`--parallel-tools`) rather than a server-side feature flag.** The rationale is:

1. **CLI option is simpler**: The parallel execution logic is entirely client-side (in `loom-cli` and `loom-cli-acp`). Using the server-side feature flags system (`loom-flags`) would require:
   - Fetching flag state from the server on startup
   - Handling offline/disconnected scenarios
   - Adding SSE subscription for real-time updates
   - This is significant complexity for a local execution behavior

2. **Environment variable fallback**: Following existing patterns (e.g., `LOOM_AUTO_COMMIT_DISABLE`), the CLI option should have an environment variable equivalent (`LOOM_PARALLEL_TOOLS=1`) for CI/automation scenarios.

3. **Default to disabled initially**: During the rollout phase, parallel execution should be opt-in. Once proven stable, the default can flip to enabled with an opt-out (`--no-parallel-tools`).

4. **No server coordination needed**: Unlike feature flags that need consistent behavior across users/orgs, parallel tool execution is a local performance optimization with no server-side implications.

## Evidence

### Existing CLI Option Patterns

The CLI already uses boolean flags for feature toggles:

```rust
// crates/loom-cli/src/main.rs:91-94
#[arg(long)]
json_logs: bool,

// crates/loom-cli-spool/src/commands/wind.rs (--git flag)
#[arg(long, short)]
git: bool,
```

### Environment Variable Pattern for Features

```rust
// crates/loom-cli-auto-commit/src/lib.rs:45-50
pub fn from_env() -> Self {
    let disabled = std::env::var("LOOM_AUTO_COMMIT_DISABLE")
        .map(|v| v == "1" || v.to_lowercase() == "true")
        .unwrap_or(false);
    // ...
}
```

### Server-Side Feature Flags Are Overkill

The `loom-flags` system is designed for:
- Multi-variant experiments with percentage rollouts
- Per-environment configuration (dev/staging/prod)
- User/org targeting with complex conditions
- Real-time updates via SSE

None of these capabilities are needed for parallel tool execution, which is a simple on/off toggle affecting only local CLI behavior.

### Proposed Implementation

```rust
// crates/loom-cli/src/main.rs

#[derive(Parser, Debug)]
struct Args {
    // ... existing fields ...

    /// Enable parallel execution of read-only tools (experimental)
    #[arg(long, env = "LOOM_PARALLEL_TOOLS")]
    parallel_tools: bool,
}

// In run_repl or tool execution:
let executor = if args.parallel_tools {
    ToolExecutor::new(registry).with_parallel(true)
} else {
    ToolExecutor::new(registry)
};
```

**Key Files:**
- `crates/loom-cli/src/main.rs` — Add `--parallel-tools` flag to `Args` struct
- `crates/loom-cli-acp/src/agent.rs` — Add equivalent configuration for ACP agent
- `crates/loom-cli-tools/src/executor.rs` — Accept parallel execution config in `ToolExecutor`

## Implications

### Recommended Rollout Strategy

| Phase | Default | Flag Behavior | Duration |
|-------|---------|---------------|----------|
| 1. Alpha | Disabled | `--parallel-tools` enables | 2-4 weeks |
| 2. Beta | Disabled | `--parallel-tools` enables, add telemetry | 2-4 weeks |
| 3. GA | Enabled | `--no-parallel-tools` disables | Permanent |

### Why NOT a Server-Side Feature Flag

| Consideration | CLI Option | Server Feature Flag |
|---------------|------------|---------------------|
| Implementation complexity | Low (~10 lines) | High (~100+ lines) |
| Offline support | Works always | Requires fallback logic |
| Latency | Zero | Adds startup delay |
| Consistency | Per-invocation | Per-session (SSE updates) |
| Debugging | `--parallel-tools` visible in logs | Requires flag evaluation trace |

### Configuration Hierarchy

Following the existing pattern in `loom-cli-config`:

1. CLI flag (`--parallel-tools`) — highest priority
2. Environment variable (`LOOM_PARALLEL_TOOLS=1`)
3. Config file (`loom.toml` → `[tools] parallel = true`)
4. Default (false during alpha, true after GA)

### Integration with ToolExecutor

The `ToolExecutor` proposed in Discovery 034 should accept this configuration:

```rust
pub struct ToolExecutor<'a> {
    registry: &'a ToolRegistry,
    parallel_enabled: bool,
}

impl<'a> ToolExecutor<'a> {
    pub fn with_parallel(mut self, enabled: bool) -> Self {
        self.parallel_enabled = enabled;
        self
    }

    pub async fn execute_batch(&self, tool_calls: &[ToolCall], ctx: &ToolContext) -> Vec<ToolExecutionOutcome> {
        if self.parallel_enabled {
            self.execute_with_parallelism(tool_calls, ctx).await
        } else {
            self.execute_sequentially(tool_calls, ctx).await
        }
    }
}
```

## Follow-up Questions

- [ ] Should the CLI log a message when parallel execution is enabled (e.g., "Parallel tool execution enabled (experimental)")?
- [ ] Should there be a `--parallel-tools-max-concurrency N` option to limit the number of concurrent read-only tools?

## Related Discoveries

- [[discovery-019]] — Parallel execution implementation complexity (estimates effort)
- [[discovery-034]] — Shared ToolExecutor crate proposal (where the flag would be consumed)
- [[discovery-011]] — Read-only tools parallelization (identifies safe tools)
