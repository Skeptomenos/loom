# Discovery: Should a WorkspaceLockManager be implemented as a prerequisite for parallel tool execution?

> Category: Follow-up (from 010-flock-performance-impact)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**No, a WorkspaceLockManager is NOT a prerequisite for parallel tool execution.** The codebase already has sufficient infrastructure to enable safe parallelism through a simpler approach: **partitioning tools by mutation status**. The existing `has_mutating_tools()` function and `MUTATING_TOOLS` constant provide explicit classification of which tools modify state (`edit_file`, `bash`) versus which are read-only (`read_file`, `list_files`, `web_search`, `oracle`).

The recommended approach is a **two-tier execution strategy**:
1. **Read-only tools**: Execute in parallel with `tokio::spawn` or `futures::join_all` — no locking needed
2. **Mutating tools**: Continue sequential execution — no locking needed

This approach avoids the complexity of a centralized lock manager while still providing meaningful parallelism benefits. A WorkspaceLockManager would only be necessary if the requirements expanded to include:
- Multiple concurrent agents operating on the same workspace
- Background tasks that modify files while the agent is running
- Fine-grained parallelism of mutating tools on different files

For the current single-agent, single-workspace model, tool partitioning is simpler, safer, and sufficient.

## Evidence

### Existing Tool Classification Infrastructure

The codebase already distinguishes mutating from non-mutating tools:

```rust
// crates/loom-common-core/src/agent.rs:48-63
fn has_mutating_tools(executions: &[ToolExecutionStatus]) -> bool {
    const MUTATING_TOOLS: &[&str] = &["edit_file", "bash"];
    executions.iter().any(|exec| {
        if let ToolExecutionStatus::Completed {
            tool_name, outcome, ..
        } = exec
        {
            MUTATING_TOOLS.contains(&tool_name.as_str())
                && matches!(outcome, ToolExecutionOutcome::Success { .. })
        } else {
            false
        }
    })
}
```

This function is currently used for auto-commit decisions but can be repurposed for parallel execution partitioning.

### State Machine Already Supports Parallelism

The spec explicitly documents parallel execution support:

```markdown
// specs/state-machine.md:39
| `ExecutingTools` | Running one or more tool calls in parallel | `conversation: ConversationContext`, `executions: Vec<ToolExecutionStatus>` |

// specs/state-machine.md:63
Tracks multiple concurrent tool executions via `Vec<ToolExecutionStatus>`. Each execution progresses
through `Pending` → `Running` → `Completed`.
```

And the tool-system spec confirms the design intent:

```markdown
// specs/tool-system.md:568-570
1. **Non-blocking I/O**: File operations use `tokio::fs` to avoid blocking the runtime
2. **Progress reporting**: Long operations can yield and report progress
3. **Concurrent execution**: Multiple independent tool calls can execute in parallel
```

### No Existing Locking Patterns in Tool Layer

Searching the codebase reveals that while `Mutex` and `RwLock` are used extensively in server-side components (OAuth pools, caches, connection managers), **no locking exists in the tool execution layer**:

- `crates/loom-cli-tools/src/edit_file.rs` — No locks
- `crates/loom-cli-tools/src/bash.rs` — No locks
- `crates/loom-cli-tools/src/read_file.rs` — No locks
- `crates/loom-cli-tools/src/list_files.rs` — No locks

This confirms that the current design relies on sequential execution rather than locking.

### Workspace Boundary Already Enforced

Each tool validates paths against the workspace root, providing implicit isolation:

```rust
// crates/loom-cli-tools/src/edit_file.rs:39-68
fn validate_path(path: &PathBuf, workspace_root: &Path) -> Result<PathBuf, ToolError> {
    // ... canonicalize and check starts_with workspace_root ...
    if !canonical.starts_with(&workspace_canonical) {
        return Err(ToolError::PathOutsideWorkspace(path.clone()));
    }
    // ...
}
```

This means tools already operate within a defined boundary — a lock manager wouldn't add new safety guarantees for single-agent scenarios.

### Comparison: Lock Manager vs Tool Partitioning

| Approach | Complexity | Safety | Parallelism Benefit |
|----------|------------|--------|---------------------|
| **Sequential (current)** | None | Full | None |
| **Tool Partitioning** | Low (~20 lines) | Full | High for read-only batches |
| **WorkspaceLockManager** | High (~200+ lines) | Full | High for all tools |
| **Per-file flock** | Medium (~50 lines per tool) | Full | Medium |

Tool partitioning provides the best complexity-to-benefit ratio for the current use case.

**Key Files:**
- `crates/loom-common-core/src/agent.rs` — `has_mutating_tools()` and `MUTATING_TOOLS` constant
- `crates/loom-cli/src/main.rs:612-656` — Sequential tool execution loop
- `specs/state-machine.md` — Documents parallel execution support in state machine
- `specs/tool-system.md:568-570` — Documents concurrent execution as a design goal
- `discoveries/011-read-only-tools-parallelization.md` — Identifies safe-to-parallelize tools

## Implications

When working in this codebase:

- **Implement tool partitioning first** — Before considering a lock manager, implement the simpler approach of partitioning tool calls into read-only and mutating sets, then parallelize only the read-only set.

- **The `MUTATING_TOOLS` constant is the source of truth** — Any new tool that modifies files must be added to this list. Consider adding a `is_mutating()` method to the Tool trait for explicit classification.

- **A lock manager would be over-engineering for current requirements** — The single-agent, single-workspace model doesn't benefit from fine-grained locking. Sequential execution of mutating tools is simpler and equally safe.

- **Future multi-agent scenarios would change this calculus** — If Loom ever supports multiple concurrent agents on the same workspace (e.g., background refactoring while user is coding), a WorkspaceLockManager would become necessary.

- **Network-bound tools benefit most from parallelism** — `web_search` and `oracle` have significant latency. Parallelizing these provides the biggest wins without any locking concerns.

## Follow-up Questions

- [ ] What would be the implementation complexity of adding parallel execution for read-only tool batches to the CLI?
- [ ] Should a `is_read_only()` or `is_mutating()` method be added to the Tool trait for explicit classification?

## Related Discoveries

- [[004-sequential-execution-rationale]] — Explains why sequential execution was chosen
- [[010-flock-performance-impact]] — Analyzes file locking as an alternative approach
- [[011-read-only-tools-parallelization]] — Identifies which tools are safe to parallelize
- [[002-concurrent-tool-execution]] — Documents state machine support for parallelism
