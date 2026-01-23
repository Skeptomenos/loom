# Coding Principles

A collection of generalized coding best practices, design patterns, and guidelines for building robust software. These principles are language-agnostic where possible, with specific examples in Rust and TypeScript/Svelte.

## Documents

| Document | Description |
|----------|-------------|
| [conventions.md](conventions.md) | Code style, naming conventions, import organization, and formatting rules |
| [patterns.md](patterns.md) | Catalog of design patterns with examples and rationale |
| [testing.md](testing.md) | Testing strategy emphasizing property-based testing |
| [error-handling.md](error-handling.md) | Error type design, propagation, and recovery strategies |
| [retry-strategy.md](retry-strategy.md) | HTTP retry patterns with exponential backoff and jitter |
| [gotchas.md](gotchas.md) | Common pitfalls and mistakes to avoid |

## Quick Reference

### Core Principles

1. **Property-Based Testing First** - Prefer tests that verify invariants over example-based tests
2. **Explicit Error Types** - Use typed errors in library code, not generic error types
3. **Never Log Secrets** - Wrap sensitive values in types that auto-redact in logs
4. **Centralized HTTP Clients** - Use factory functions for consistent configuration
5. **Structured Logging** - Use structured fields, not string interpolation
6. **Exponential Backoff with Jitter** - Prevent thundering herd on retries

### Technology-Specific

#### Rust
- Use `thiserror` for error enums, `anyhow` for propagation
- Use `async-trait` for async trait methods
- Instrument functions with `#[instrument(skip(...))]`
- Prefer `proptest` for property-based testing

#### Svelte 5
- Use runes: `$state`, `$derived`, `$effect`, `$props`
- Never use Svelte 4 patterns (`export let`, `$:`, `on:click`)

## How to Use These Guidelines

1. **For new projects**: Copy these files and customize for your project
2. **For existing projects**: Use as a checklist to identify improvement areas
3. **For code reviews**: Reference specific sections when providing feedback
4. **For onboarding**: Share with new team members to establish baseline expectations

## Customization

These guidelines are meant to be adapted. When using them:

1. Replace `{project-name}` placeholders with your project name
2. Adjust crate/package naming conventions to match your organization
3. Modify formatting rules to match your team preferences
4. Add project-specific gotchas as you discover them

## Contributing

When adding new principles:

1. Ensure they are generalizable (not project-specific)
2. Include rationale ("Why") not just rules ("What")
3. Provide code examples where helpful
4. Link related documents for cross-referencing
