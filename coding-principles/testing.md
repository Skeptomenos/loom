# Testing Strategy

This document outlines a testing philosophy centered on property-based testing, complemented by targeted unit and integration tests.

---

## Core Principles

1. **Property-Based Testing First**: Prefer tests that verify invariants over example-based tests that check specific cases
2. **Document Every Test**: Each test should explain why it's important and what invariant it verifies
3. **Fail Fast, Fail Clearly**: Tests should produce clear error messages that identify the root cause
4. **Test Behavior, Not Implementation**: Focus on what the code does, not how it does it

---

## Test Categories

### Unit Tests

Standard tests for synchronous, isolated logic. Located in test modules within source files.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_validation() {
        let result = validate_input("valid");
        assert!(result.is_ok());
    }
    
    #[tokio::test]
    async fn test_async_operation() {
        let result = fetch_data().await;
        assert!(result.is_ok());
    }
}
```

### Property-Based Tests

Generative tests that verify invariants hold across randomly generated inputs.

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn serialization_roundtrip(input in "[a-z]{1,100}") {
        let serialized = serialize(&input);
        let deserialized = deserialize(&serialized)?;
        prop_assert_eq!(input, deserialized);
    }
}
```

### Integration Tests

Async tests that exercise complete workflows including I/O operations.

```rust
#[tokio::test]
async fn test_complete_workflow() {
    let client = TestClient::new();
    let result = client.create_resource().await;
    assert!(result.is_ok());
    
    let fetched = client.get_resource(result.id).await;
    assert_eq!(fetched.name, "test");
}
```

---

## Property-Based Testing with proptest

### Why Property Tests Over Example-Based

| Example-Based Tests | Property-Based Tests |
|---------------------|---------------------|
| Test specific inputs | Test input *space* |
| May miss edge cases | Explores edge cases automatically |
| Documents behavior for one case | Documents invariants for all cases |
| Brittle to refactoring | Robust to implementation changes |

Property tests are preferred because they:

1. **Discover edge cases** you didn't think of (unicode, empty strings, boundary values)
2. **Verify invariants** that must hold for all valid inputs
3. **Shrink failures** to minimal reproducible examples
4. **Scale testing** to thousands of cases with one test

### Generators and Strategies

proptest provides strategies for generating test data:

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn example_property(
        // String matching regex pattern
        name in "[a-zA-Z][a-zA-Z0-9_]{0,30}",
        // Optional value
        max_value in proptest::option::of(1u32..10000),
        // Range of values
        temperature in 0.0f32..2.0,
        // Collection with size bounds
        items in prop::collection::vec("[a-z]{1,10}", 0..10),
        // Hash set (unique values)
        unique_names in prop::collection::hash_set("[a-z]{1,10}", 0..5),
    ) {
        prop_assert!(name.len() <= 31);
        prop_assert!(items.len() <= 10);
    }
}
```

**Common strategies**:
- `"[a-zA-Z0-9]{n,m}"` - Regex-based string generation
- `proptest::option::of(strategy)` - Optional values
- `prop::collection::vec(strategy, range)` - Vector generation
- `prop::collection::hash_set(strategy, range)` - Unique value sets
- `n..m` - Numeric ranges

### Preconditions with prop_assume!

Use `prop_assume!` to filter invalid test cases:

```rust
proptest! {
    #[test]
    fn division_test(
        numerator in 0i32..1000,
        denominator in 0i32..1000
    ) {
        // Skip division by zero
        prop_assume!(denominator != 0);
        
        let result = numerator / denominator;
        prop_assert!(result <= numerator);
    }
}
```

---

## Key Test Areas

### 1. State Machine Transitions

Verify correct transitions between states:

```rust
proptest! {
    #[test]
    fn state_transitions_are_valid(events in prop::collection::vec(any::<Event>(), 0..20)) {
        let mut machine = StateMachine::new();
        
        for event in events {
            let old_state = machine.state().clone();
            let action = machine.handle_event(event);
            
            // Verify transition was valid
            prop_assert!(is_valid_transition(&old_state, machine.state()));
        }
    }
}
```

**Key properties**:
- Machine always starts in initial state
- Specific inputs deterministically trigger expected state changes
- Bounded retry counts never exceed maximum
- Shutdown always succeeds from any state

### 2. Serialization Roundtrips

Verify data survives serialization:

```rust
proptest! {
    #[test]
    fn json_roundtrip_preserves_data(
        name in "[a-z]{1,20}",
        count in 0u32..10000,
    ) {
        let original = Data { name, count };
        let json = serde_json::to_string(&original)?;
        let restored: Data = serde_json::from_str(&json)?;
        
        prop_assert_eq!(original, restored);
    }
}
```

### 3. File Operations

Verify file operations maintain invariants:

```rust
proptest! {
    #[test]
    fn edit_is_reversible(
        original in "[a-z]{10,100}",
        replacement in "[A-Z]{5,50}"
    ) {
        let edited = original.replace("abc", &replacement);
        let restored = edited.replace(&replacement, "abc");
        
        // If original contained "abc", edit should be reversible
        if original.contains("abc") && !replacement.contains("abc") {
            prop_assert_eq!(original, restored);
        }
    }
}
```

**Key properties**:
- Reversibility: operation followed by inverse restores original
- Idempotency: same operation applied twice produces same result
- Byte count accuracy: reported sizes match actual sizes
- Unicode safety: multi-byte sequences handled correctly

### 4. Retry Behavior

Verify retry logic:

```rust
#[test]
fn non_retryable_errors_fail_immediately() {
    let error = ClientError::validation("invalid input");
    assert!(!error.is_retryable());
}

#[test]
fn retryable_errors_respect_max_attempts() {
    let config = RetryConfig { max_attempts: 3, .. };
    let mut attempts = 0;
    
    let result = retry(&config, || {
        attempts += 1;
        Err(TransientError::timeout())
    });
    
    assert!(result.is_err());
    assert_eq!(attempts, 3);
}
```

---

## Test Documentation Requirements

**Every test MUST document**:

1. **Purpose**: Why this test is important
2. **Invariant**: What property/behavior it verifies
3. **Context**: When this matters (failure scenarios, edge cases)

### Required Format

```rust
/// **Test Name**: Brief description of what's being tested
///
/// **Why this is important**: Explain the significance and potential
/// failure modes this test catches. Include real-world scenarios.
///
/// **Invariant**: Formal statement of the property being verified.
#[test]
fn test_example() {
    // ...
}
```

### Example

```rust
/// **Property test: Retry count never exceeds max_retries**
/// 
/// This property verifies the retry bound invariant:
/// - After max_retries errors, the system must stop retrying
/// - System should transition to error state at the limit
/// 
/// Prevents infinite retry loops that could exhaust resources.
#[test]
fn retry_count_bounded_by_max_retries() {
    // ...
}
```

---

## Mock Implementations

### Mock Clients for External Services

```rust
struct MockApiClient;

#[async_trait]
impl ApiClient for MockApiClient {
    async fn request(&self, _req: Request) -> Result<Response, Error> {
        Ok(Response {
            status: 200,
            body: json!({"result": "ok"}),
        })
    }
}
```

### Mock with Controllable Behavior

```rust
struct MockWithBehavior {
    should_fail: bool,
    call_count: AtomicU32,
}

#[async_trait]
impl Service for MockWithBehavior {
    async fn call(&self) -> Result<(), Error> {
        self.call_count.fetch_add(1, Ordering::SeqCst);
        if self.should_fail {
            Err(Error::simulated())
        } else {
            Ok(())
        }
    }
}
```

### Controllable Error for Retry Tests

```rust
#[derive(Debug)]
struct MockError {
    retryable: bool,
}

impl RetryableError for MockError {
    fn is_retryable(&self) -> bool {
        self.retryable
    }
}
```

---

## Async Testing

### tokio::test Attribute

For async tests, use the `#[tokio::test]` attribute:

```rust
#[tokio::test]
async fn test_async_operation() {
    let client = TestClient::new();
    let result = client.fetch().await.unwrap();
    assert_eq!(result.status, "ok");
}
```

### Runtime Creation for proptest

proptest doesn't natively support async. Create a runtime inside the test:

```rust
proptest! {
    #[test]
    fn async_property_test(input in "[a-z]{1,20}") {
        let rt = tokio::runtime::Runtime::new().unwrap();
        rt.block_on(async {
            let result = async_operation(&input).await;
            prop_assert!(result.is_ok());
            Ok(())
        }).unwrap();
    }
}
```

---

## Running Tests

### All Tests

```bash
cargo test --workspace
# or
npm test
```

### Specific Package/Module

```bash
cargo test -p project-core
cargo test -p project-server
```

### Test Filtering

```bash
# Run tests matching pattern
cargo test state_machine

# Run specific test
cargo test test_user_input_transitions

# Run with output displayed
cargo test -- --nocapture

# Run ignored tests
cargo test -- --ignored
```

### Proptest Configuration

Control proptest behavior with environment variables:

```bash
# Run more cases for thorough testing
PROPTEST_CASES=1000 cargo test

# Set seed for reproducibility
PROPTEST_SEED=12345 cargo test

# Verbose output for debugging failures
PROPTEST_VERBOSE=1 cargo test
```

---

## Test Helpers

### Temporary Resources

Use temporary directories/files for isolated tests:

```rust
fn setup_temp_dir() -> tempfile::TempDir {
    tempfile::tempdir().expect("failed to create temp dir")
}

#[test]
fn test_file_operation() {
    let dir = setup_temp_dir();
    let file_path = dir.path().join("test.txt");
    
    std::fs::write(&file_path, "content").unwrap();
    // Test operations...
}
```

### Test Fixtures

Helper functions for creating test data:

```rust
fn create_test_config() -> Config {
    Config {
        timeout: Duration::from_secs(30),
        max_retries: 3,
        ..Default::default()
    }
}

fn create_test_request(body: &str) -> Request {
    Request {
        method: Method::POST,
        body: body.to_string(),
        headers: HashMap::new(),
    }
}
```

---

## Test Dependencies

### Rust (Cargo.toml)

```toml
[dev-dependencies]
proptest = "1.4"
tokio-test = "0.4"
tempfile = "3"
mockall = "0.11"
```

### TypeScript (package.json)

```json
{
  "devDependencies": {
    "vitest": "^1.0.0",
    "fast-check": "^3.0.0",
    "@testing-library/svelte": "^4.0.0"
  }
}
```

---

## See Also

- [conventions.md](conventions.md) - Coding conventions
- [error-handling.md](error-handling.md) - Error handling patterns
- [patterns.md](patterns.md) - Design patterns
