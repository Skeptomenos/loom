# Discovery: What is the primary purpose of Loom?

> Category: Architecture
> Discovered: 2026-01-16
> Confidence: High

## Answer

Loom is an **AI-powered coding agent** built in Rust that enables developers to interact with LLM-powered assistants capable of executing tools to perform file system operations, code analysis, shell commands, and other development tasks. The system is designed around a **server-side LLM proxy architecture** where API keys never leave the server, providing security and centralized credential management.

The primary workflows Loom supports are:

1. **Interactive REPL** — A conversational interface where users provide natural language prompts, and the agent reads files, edits code, runs commands, and auto-commits changes.
2. **Remote Execution (Weavers)** — Ephemeral Kubernetes pods that provide isolated environments for running agent code, accessible via WireGuard tunnels.
3. **Editor Integration (ACP)** — The Agent Client Protocol allows IDEs like VS Code to use Loom as a backend, with the editor providing UI while the CLI handles agent logic.

Loom is fundamentally a **tool orchestration platform** that bridges LLM reasoning with concrete filesystem and shell operations, enforcing security boundaries (workspace sandboxing) while enabling powerful autonomous coding capabilities.

## Evidence

**Architecture Spec** (`specs/architecture.md`):
```
Loom is an AI-powered coding assistant built in Rust. It provides a REPL interface for interacting
with LLM-powered agents that can execute tools to perform file system operations and other tasks.

The system is designed around three core principles:
1. Modularity - Clean separation between core abstractions, LLM providers, and tools
2. Extensibility - Easy addition of new LLM providers and tools via trait implementations
3. Reliability - Robust error handling with retry mechanisms and structured logging
```

**Server-Side Proxy Design** (`specs/architecture.md`):
```
┌─────────────┐      HTTP       ┌─────────────┐     Provider API    ┌─────────────┐
│  loom-cli   │ ───────────────▶│ loom-server │ ──────────────────▶ │  Anthropic  │
│             │ /proxy/{provider}│             │                     │   OpenAI    │
│ ProxyLlm-   │  /complete      │  LlmService │                     │    etc.     │
│ Client      │  /stream        │             │                     │             │
└─────────────┘ ◀─────────────  └─────────────┘ ◀────────────────── └─────────────┘
                  SSE stream                        SSE stream
```

**State Machine** (`specs/state-machine.md`):
- `WaitingForUserInput` → `CallingLlm` → `ProcessingLlmResponse` → `ExecutingTools` → `PostToolsHook` → loop
- Explicit event-driven state machine with `AgentEvent` inputs and `AgentAction` outputs

**Key Files:**
- `crates/loom-cli/src/main.rs` — CLI entry point with REPL, weaver, and ACP commands
- `specs/architecture.md` — Core architecture documentation
- `specs/state-machine.md` — Agent state machine specification
- `specs/tool-system.md` — Tool registry and execution framework
- `crates/loom-cli-tools/` — Tool implementations (read_file, edit_file, bash, oracle, web_search)

## Implications

- **Security-first design**: API keys are server-side only; all LLM requests go through the proxy
- **Tool-centric architecture**: The agent's power comes from its tools; adding capabilities means implementing the `Tool` trait
- **State machine clarity**: All agent behavior is explicit and testable via the state machine pattern
- **Multi-environment support**: Same agent logic works locally (REPL), remotely (Weaver), and in editors (ACP)

When working in this codebase:
- Understand that `loom-cli` is the "hands" executing tools, while `loom-server` is the "brain" routing LLM requests
- New tools go in `loom-cli-tools` and are registered in `main.rs`
- The state machine in `loom-core` is the source of truth for agent behavior

## Follow-up Questions

- [ ] How does the Agent state machine handle concurrent tool executions and what happens when multiple tools are called in parallel?
- [ ] What is the complete request flow from user input through the LLM proxy to tool execution and back?

## Related Discoveries

<!-- Links to related discovery files -->
- (none yet - this is the first discovery)
