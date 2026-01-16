# Discovery: Should `ToolDefinition` be extended to include an `is_mutating` field so the LLM can be informed of tool characteristics?

> Category: Follow-up (from 020-tool-trait-is-mutating-method)
> Discovered: 2026-01-16
> Confidence: Medium

## Answer

**Yes, but with caveats.** Extending `ToolDefinition` with an `is_mutating` field is technically straightforward and would provide the LLM with explicit information about tool side effects. However, neither Anthropic nor OpenAI's tool/function APIs have a standardized field for this metadata. The information would need to be conveyed through the tool's `description` field rather than a dedicated schema property.

**Recommended approach**: Add `is_mutating` to `ToolDefinition` for internal use (parallel execution decisions, auto-commit logic), and optionally append mutation status to the tool description for LLM awareness. This provides both machine-readable classification for the runtime and human-readable context for the LLM.

## Evidence

### Current ToolDefinition Structure

```rust
// crates/loom-common-core/src/tool.rs:9-13
pub struct ToolDefinition {
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,
}
```

### LLM Provider Tool Formats

All three providers (Anthropic, OpenAI, Vertex) use the same core fields:

**Anthropic** (`AnthropicTool`):
```rust
pub struct AnthropicTool {
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,
}
```

**OpenAI** (`OpenAIFunction`):
```rust
pub struct OpenAIFunction {
    pub name: String,
    pub description: String,
    pub parameters: serde_json::Value,
}
```

**Vertex AI** (`VertexFunctionDeclaration`):
```rust
pub struct VertexFunctionDeclaration {
    pub name: String,
    pub description: String,
    pub parameters: serde_json::Value,
}
```

None of these formats include a dedicated `is_mutating` or `side_effects` field. The only way to communicate this to the LLM is through the `description` text.

### Industry Standards (2025/2026)

Research into LLM tool definition standards reveals:

1. **No Universal Boolean Field**: Neither Anthropic nor OpenAI have a standardized `is_read_only` or `side_effects` field in their base APIs.

2. **Model Context Protocol (MCP)**: The emerging industry standard for tool interoperability supports an `annotations` object where developers can define custom properties like `is_read_only` or `requires_confirmation`. This is the closest to a structured approach.

3. **Best Practice - Natural Language**: Developers are encouraged to explicitly state side effects in the `description`:
   - *Example:* "Reads the content of a file. This tool is read-only and does not modify the filesystem."
   - For mutating tools: `[SIDE EFFECT] This tool will permanently delete data.`

4. **Provider-Specific Features**:
   - **OpenAI `strict`**: Guarantees output matches schema exactly (Structured Outputs)
   - **Anthropic `cache_control`**: Allows marking tool definitions for Prompt Caching

5. **Future Consideration**: If Loom adopts MCP compatibility, an `annotations` map would be more extensible than a single boolean field.

### Proposed Implementation

```rust
// crates/loom-common-core/src/tool.rs
#[derive(Clone, Debug, Serialize, Deserialize)]
pub struct ToolDefinition {
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,
    /// Whether this tool modifies the workspace or system state.
    /// Used for parallel execution decisions and auto-commit logic.
    #[serde(default)]
    pub is_mutating: bool,
}

impl ToolDefinition {
    pub fn new(
        name: impl Into<String>,
        description: impl Into<String>,
        input_schema: serde_json::Value,
    ) -> Self {
        Self {
            name: name.into(),
            description: description.into(),
            input_schema,
            is_mutating: false,  // Default: read-only
        }
    }

    pub fn with_mutating(mut self, is_mutating: bool) -> Self {
        self.is_mutating = is_mutating;
        self
    }
}
```

### Tool Trait Update

```rust
// crates/loom-cli-tools/src/registry.rs
fn to_definition(&self) -> ToolDefinition {
    ToolDefinition {
        name: self.name().to_string(),
        description: self.description().to_string(),
        input_schema: self.input_schema(),
        is_mutating: self.is_mutating(),  // New: pull from trait method
    }
}
```

### Provider Conversion (Optional LLM Awareness)

If we want the LLM to be aware of mutation status, we can append it to the description during conversion:

```rust
// crates/loom-server-llm-openai/src/types.rs
impl From<&ToolDefinition> for OpenAITool {
    fn from(tool: &ToolDefinition) -> Self {
        let description = if tool.is_mutating {
            format!("{} [MUTATING: This tool modifies files or system state]", tool.description)
        } else {
            tool.description.clone()
        };
        
        Self {
            tool_type: "function".to_string(),
            function: OpenAIFunction {
                name: tool.name.clone(),
                description,
                parameters: tool.input_schema.clone(),
            },
        }
    }
}
```

**Key Files:**
- `crates/loom-common-core/src/tool.rs` — ToolDefinition struct (needs modification)
- `crates/loom-cli-tools/src/registry.rs` — Tool trait and to_definition() (needs modification)
- `crates/loom-server-llm-openai/src/types.rs` — OpenAI conversion (optional modification)
- `crates/loom-server-llm-anthropic/src/types.rs` — Anthropic conversion (optional modification)
- `crates/loom-server-llm-vertex/src/types.rs` — Vertex conversion (optional modification)

## Implications

1. **Backward compatible**: Adding `#[serde(default)]` ensures existing serialized `ToolDefinition` objects deserialize correctly with `is_mutating: false`.

2. **Enables runtime optimizations**: With `is_mutating` in `ToolDefinition`, the execution loop can partition tools without needing access to the `ToolRegistry`:
   ```rust
   let (read_only, mutating): (Vec<_>, Vec<_>) = tool_calls
       .iter()
       .partition(|tc| !definitions.get(&tc.tool_name).map(|d| d.is_mutating).unwrap_or(true));
   ```

3. **LLM awareness is optional**: The field can exist purely for internal use. Appending to descriptions is a choice, not a requirement.

4. **Consistency with discovery-020**: This change complements the `is_mutating()` method on the `Tool` trait. The trait method is the source of truth; `ToolDefinition.is_mutating` is the serialized representation.

5. **No API changes required**: Since LLM providers don't have a dedicated field, we're not constrained by external schemas.

6. **Migration path**:
   - Phase 1: Add `is_mutating` field to `ToolDefinition` with `#[serde(default)]`
   - Phase 2: Update `to_definition()` to populate from `Tool::is_mutating()`
   - Phase 3: (Optional) Update provider conversions to append to description
   - Phase 4: Update `has_mutating_tools()` in agent.rs to use the field

## Follow-up Questions

- [ ] Should there be additional classification methods like `is_network_bound()` to enable smarter execution strategies?
- [ ] Should the `error` code field be included in the displayed message for certain error types (e.g., "conflict: Version mismatch...")?

## Related Discoveries

- [[020-tool-trait-is-mutating-method]] — Recommends adding `is_mutating()` to the Tool trait (prerequisite)
- [[011-read-only-tools-parallelization]] — Identifies which tools are safe to parallelize
- [[019-parallel-read-only-tool-implementation-complexity]] — Outlines parallel execution implementation
- [[034-parallel-execution-shared-crate]] — Proposes shared ToolExecutor crate that would use this field
