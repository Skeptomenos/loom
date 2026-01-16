# Discovery: How does the PostToolsHook auto-commit feature work?

> Category: Follow-up (from Architecture)
> Discovered: 2026-01-16
> Confidence: High

## Answer

The PostToolsHook auto-commit feature is a **transparent infrastructure behavior** that automatically creates git commits after mutating tool executions (`edit_file`, `bash`). It operates as a post-tool hook in the agent's state machine, invisible to the LLM's reasoning, providing an automatic audit trail of agent-driven changes.

The system follows a **5-step decision flow**:

1. **State Machine Trigger**: After all tools complete, the agent checks `has_mutating_tools()` to see if any `edit_file` or `bash` tools succeeded. If so, it transitions to `AgentState::PostToolsHook` and emits `AgentAction::RunPostToolsHook`.

2. **Configuration Check**: The `AutoCommitService::should_run()` method verifies:
   - `config.enabled` is true (default: true, disabled via `LOOM_AUTO_COMMIT_DISABLE=true`)
   - At least one completed tool is in `trigger_tools` list and succeeded

3. **Git Repository Validation**: Checks if the workspace is a git repository using `git rev-parse --show-toplevel`.

4. **Change Detection**: Runs `git diff HEAD` (or `git diff --cached` for new repos). If the diff is empty, the commit is skipped with reason "no changes".

5. **Commit Message Generation**: The `CommitMessageGenerator` sends the diff to Claude 3 Haiku with a system prompt enforcing Conventional Commits format. If LLM fails, it falls back to `"chore: auto-commit from loom"`.

**Key design principle**: Auto-commit is **enabled by default** (opt-out model) and uses a **fail-open strategy** - failures never block the agent, they just skip the commit and log the reason.

## Evidence

### State Machine Transition (Trigger)

```rust
// crates/loom-common-core/src/agent.rs:342-358
if has_mutating_tools(executions) {
    let completed_tools = extract_completed_tools(executions);
    debug!(
        tool_count = completed_tools.len(),
        "mutating tools detected, running post-tools hook"
    );
    self.state = AgentState::PostToolsHook {
        conversation: conv,
        pending_llm_request: request,
        completed_tools: completed_tools.clone(),
    };
    AgentAction::RunPostToolsHook { completed_tools }
}
```

### Mutating Tools Definition

```rust
// crates/loom-common-core/src/agent.rs:48-63
fn has_mutating_tools(executions: &[ToolExecutionStatus]) -> bool {
    const MUTATING_TOOLS: &[&str] = &["edit_file", "bash"];
    executions.iter().any(|exec| {
        if let ToolExecutionStatus::Completed { tool_name, outcome, .. } = exec {
            MUTATING_TOOLS.contains(&tool_name.as_str())
                && matches!(outcome, ToolExecutionOutcome::Success { .. })
        } else {
            false
        }
    })
}
```

### AutoCommitService Decision Logic

```rust
// crates/loom-cli-auto-commit/src/service.rs:77-109
pub async fn run(&self, workspace_root: &Path, completed_tools: &[CompletedToolInfo]) -> AutoCommitResult {
    if !self.config.enabled {
        return AutoCommitResult::skipped("disabled");
    }
    if !self.should_run(completed_tools) {
        return AutoCommitResult::skipped("no trigger tools");
    }
    if !self.git.is_repository(workspace_root).await {
        return AutoCommitResult::skipped("not a git repository");
    }
    let diff = match self.git.diff_all(workspace_root).await { ... };
    if diff.is_empty() {
        return AutoCommitResult::skipped("no changes");
    }
    // ... generate message and commit ...
}
```

### Configuration Defaults

```rust
// crates/loom-cli-auto-commit/src/config.rs:17-25
impl Default for AutoCommitConfig {
    fn default() -> Self {
        Self {
            enabled: true,  // Enabled by default
            model: "claude-3-haiku-20240307".to_string(),
            max_diff_bytes: 32 * 1024,  // 32KB
            trigger_tools: vec!["edit_file".to_string(), "bash".to_string()],
        }
    }
}
```

### Commit Message Generation Prompt

```rust
// crates/loom-cli-auto-commit/src/generator.rs:15-30
const SYSTEM_PROMPT: &str = r#"You are an expert software engineer generating git commit messages.

Rules:
1. Use conventional commit format: <type>(<scope>): <description>
2. Types: feat, fix, refactor, docs, style, test, chore
3. Keep the first line under 72 characters
4. Be specific about what changed, not why
5. Use imperative mood ("add" not "added")
6. If multiple unrelated changes, summarize the primary one
7. Output ONLY the commit message, nothing else
..."#;
```

### PostToolsHook Completion Transition

```rust
// crates/loom-common-core/src/agent.rs:376-398
(AgentState::PostToolsHook { conversation, pending_llm_request, .. },
 AgentEvent::PostToolsHookCompleted { action_taken }) => {
    debug!(action_taken = action_taken, "post-tools hook completed");
    self.state = AgentState::CallingLlm { conversation: conv, retries: 0 };
    AgentAction::SendLlmRequest(request)
}
```

**Key Files:**
- `crates/loom-common-core/src/agent.rs` - State machine with `has_mutating_tools()` and transitions
- `crates/loom-common-core/src/state.rs` - `AgentState::PostToolsHook` definition
- `crates/loom-cli-auto-commit/src/service.rs` - `AutoCommitService` orchestration
- `crates/loom-cli-auto-commit/src/config.rs` - `AutoCommitConfig` with defaults
- `crates/loom-cli-auto-commit/src/generator.rs` - `CommitMessageGenerator` with LLM prompt
- `crates/loom-cli/src/main.rs` - CLI integration calling `run_auto_commit()`
- `specs/auto-commit-system.md` - Design specification

## Implications

When working in this codebase:

- **Auto-commit is infrastructure, not a tool** - It runs transparently after tool execution, invisible to the LLM's reasoning. The agent doesn't "decide" to commit; it happens automatically.

- **Fail-open design** - All errors result in skipping the commit and logging, never blocking the agent. This is critical for reliability.

- **Opt-out via environment variable** - Set `LOOM_AUTO_COMMIT_DISABLE=true` to disable. There's no CLI flag; it's intentionally simple.

- **Fixed model for commit messages** - Claude 3 Haiku is hardcoded for cost efficiency. Users cannot change this.

- **Diff truncation at 32KB** - Large diffs are truncated to avoid context window issues. The truncation notice is appended to the prompt.

- **Extensible hook system** - `PostToolsHook` is designed to support future hooks (auto-format, auto-lint) via `PostToolsHookConfig`.

- **State machine is pure** - The agent's state machine doesn't do I/O. The CLI receives `RunPostToolsHook` action and calls `AutoCommitService`, then sends `PostToolsHookCompleted` event back.

## Follow-up Questions

- [ ] How does the LlmService handle model resolution (e.g., mapping "default" to specific model versions) and what happens when a model is not available?
- [ ] How does the SSE error handling work when the connection drops mid-stream, and how does the client recover?

## Related Discoveries

- [[001-primary-purpose]] - Establishes Loom as an AI coding agent with auto-commit as a core workflow
- [[002-concurrent-tool-execution]] - Documents that PostToolsHook runs after all tools complete
- [[003-request-flow]] - Shows PostToolsHook in the request lifecycle after tool execution
- [[004-sequential-execution-rationale]] - Explains why tools run sequentially, which affects when auto-commit triggers
