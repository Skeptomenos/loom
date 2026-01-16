# Discovery: What is the primary purpose of this codebase?

> Category: Architecture
> Discovered: 2026-01-16
> Confidence: High

## Answer

Loom is an AI-powered coding agent built in Rust that provides a REPL interface for interacting with LLM-powered agents capable of executing tools to perform file system operations, code analysis, and other development tasks.

The system is designed around three core principles: **Modularity** (clean separation between core abstractions, LLM providers, and tools), **Extensibility** (easy addition of new LLM providers and tools via trait implementations), and **Reliability** (robust error handling with retry mechanisms and structured logging).

## Evidence

**Key Files:**
- `README.md` — States: "Loom is an AI-powered coding agent built in Rust. It provides a REPL interface for interacting with LLM-powered agents that can execute tools to perform file system operations, code analysis, and other development tasks."

```markdown
The system is designed around three core principles:

1. **Modularity** - Clean separation between core abstractions, LLM providers, and tools
2. **Extensibility** - Easy addition of new LLM providers and tools via trait implementations
3. **Reliability** - Robust error handling with retry mechanisms and structured logging
```

**Architecture Overview:**
- 30+ Rust crates organized in a workspace
- Server-side LLM proxy architecture (API keys never leave the server)
- Kubernetes-based remote execution environments (Weavers)
- Svelte 5 web frontend
- SQLite persistence with FTS5 search

## Implications

- When adding new features, consider which of the three principles (modularity, extensibility, reliability) applies
- New LLM providers should be added via trait implementations in dedicated crates
- Tools are first-class citizens with their own registry and execution framework
- The proxy architecture means clients never handle API keys directly

## Follow-up Questions

- [ ] How does the state machine orchestrate conversation flow and tool execution?
- [ ] What is the relationship between loom-core, loom-server, and loom-cli?

## Related Discoveries

<!-- Links to related discovery files -->
