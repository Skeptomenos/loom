# Discovery: Should a `is_read_only()` or `is_mutating()` method be added to the Tool trait for explicit classification?

> Category: Follow-up (from 011-read-only-tools-parallelization)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**Yes, adding an `is_mutating()` method to the Tool trait is recommended.** The current implementation uses a hardcoded string list (`MUTATING_TOOLS: &[&str] = &["edit_file", "bash"]`) in `loom-common-core/src/agent.rs` to classify tools. This approach has several drawbacks:

1. **Fragile**: Adding a new mutating tool requires updating a separate constant, which is easy to forget
2. **Not self-documenting**: Tool implementations don't declare their own behavior characteristics
3. **Scattered logic**: Classification lives in `agent.rs` while tool definitions live in `loom-cli-tools`
4. **No compile-time enforcement**: A typo in the tool name string would silently fail

The recommended approach is to add a method with a **safe default** to the Tool trait:

```rust
fn is_mutating(&self) -> bool {
    false  // Default: tools are read-only unless they override
}
```

This design choice (defaulting to `false`) is intentional: most tools are read-only, and mutating tools are the exception that must explicitly opt-in. This follows the principle of least privilege.

## Evidence

### Current Hardcoded Classification

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

### Current Tool Trait (No Classification Method)

```rust
// crates/loom-cli-tools/src/registry.rs:8-29
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn input_schema(&self) -> serde_json::Value;
    fn to_definition(&self) -> ToolDefinition { ... }
    async fn invoke(&self, args: serde_json::Value, ctx: &ToolContext) -> Result<serde_json::Value, ToolError>;
    // NOTE: No is_mutating() or is_read_only() method exists
}
```

### Tool Registration (Shows All Current Tools)

```rust
// crates/loom-cli/src/main.rs:374-390
fn create_tool_registry() -> ToolRegistry {
    let mut registry = ToolRegistry::new();
    registry.register(Box::new(ReadFileTool::new()));      // read-only
    registry.register(Box::new(ListFilesTool::new()));     // read-only
    registry.register(Box::new(EditFileTool::new()));      // MUTATING
    registry.register(Box::new(BashTool::new()));          // MUTATING
    registry.register(Box::new(OracleTool::default()));    // read-only (external API)
    registry.register(Box::new(WebSearchToolGoogle::default())); // read-only (external API)
    registry.register(Box::new(WebSearchToolSerper::default())); // read-only (external API)
    registry
}
```

### Proposed Implementation

```rust
// crates/loom-cli-tools/src/registry.rs
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &str;
    fn description(&self) -> &str;
    fn input_schema(&self) -> serde_json::Value;
    
    /// Returns true if this tool modifies the workspace or system state.
    /// Default is false (read-only). Mutating tools must override this.
    fn is_mutating(&self) -> bool {
        false
    }
    
    fn to_definition(&self) -> ToolDefinition { ... }
    async fn invoke(&self, args: serde_json::Value, ctx: &ToolContext) -> Result<serde_json::Value, ToolError>;
}

// crates/loom-cli-tools/src/edit_file.rs
impl Tool for EditFileTool {
    fn is_mutating(&self) -> bool { true }  // Override
    // ... rest of implementation
}

// crates/loom-cli-tools/src/bash.rs
impl Tool for BashTool {
    fn is_mutating(&self) -> bool { true }  // Override
    // ... rest of implementation
}
```

**Key Files:**
- `crates/loom-cli-tools/src/registry.rs` — Tool trait definition (needs modification)
- `crates/loom-common-core/src/agent.rs` — Current hardcoded classification (can be simplified)
- `crates/loom-cli-tools/src/edit_file.rs` — Mutating tool (needs override)
- `crates/loom-cli-tools/src/bash.rs` — Mutating tool (needs override)

## Implications

1. **Backward compatible**: Adding a default method to the trait doesn't break existing implementations. All current tools will automatically be classified as read-only, which is correct for 5 of 7 tools.

2. **Self-documenting**: Each tool declares its own behavior. When reading `EditFileTool`, you immediately see `is_mutating() -> true`.

3. **Enables parallel execution**: With explicit classification, the execution loop can partition tools:
   ```rust
   let (read_only, mutating): (Vec<_>, Vec<_>) = tools
       .iter()
       .partition(|t| !registry.get(t.name).unwrap().is_mutating());
   ```

4. **Extensibility**: New tools automatically get the safe default. Only mutating tools need explicit annotation.

5. **Migration path**: 
   - Phase 1: Add `is_mutating()` to trait with default
   - Phase 2: Add overrides to `EditFileTool` and `BashTool`
   - Phase 3: Update `has_mutating_tools()` to use the trait method
   - Phase 4: Remove `MUTATING_TOOLS` constant

6. **Alternative considered**: `is_read_only()` with default `true` is semantically equivalent but `is_mutating()` is preferred because:
   - "Mutating" is the exceptional case that needs explicit marking
   - Double negatives are avoided (`!is_read_only()` vs `is_mutating()`)
   - Matches the existing `has_mutating_tools()` function name

## Follow-up Questions

- [ ] Should `ToolDefinition` be extended to include an `is_mutating` field so the LLM can be informed of tool characteristics?
- [ ] Should there be additional classification methods like `is_network_bound()` to enable smarter execution strategies (e.g., prioritizing network-bound tools for parallelization)?

## Related Discoveries

- [[011-read-only-tools-parallelization]] — Identifies which tools are safe to parallelize and recommends this trait extension
- [[019-parallel-read-only-tool-implementation-complexity]] — Outlines the implementation plan that would use this classification
- [[004-sequential-execution-rationale]] — Explains why mutating tools must remain sequential
