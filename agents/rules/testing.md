# Testing Standards

## Philosophy

Tests are executable documentation. A test suite should tell the story of how the system behaves, not just verify that lines of code were executed.

Write tests for **behaviour**, not for **implementation**. If you can refactor the internals without changing any tests, your tests are well-targeted. If every refactor requires test rewrites, your tests are too coupled to the implementation.

## Test Naming

### Arrange / Act / Assert structure

Structure every test with three clearly separated sections:

```csharp
// C#
[Fact]
public async Task GetUser_WhenUserExists_ReturnsUserDto()
{
    // Arrange
    var userId = Guid.NewGuid();
    var user = new User(userId, "Alice", "alice@example.com");
    _repository.GetByIdAsync(userId).Returns(user);

    // Act
    var result = await _sut.GetUserAsync(userId);

    // Assert
    result.Should().NotBeNull();
    result!.Id.Should().Be(userId);
    result.Name.Should().Be("Alice");
}
```

```typescript
// TypeScript (Vitest)
it('should return the user when the user exists', async () => {
  // Arrange
  const userId = 'user-123';
  const mockUser = { id: userId, name: 'Alice', email: 'alice@example.com' };
  vi.mocked(userRepository.findById).mockResolvedValue(mockUser);

  // Act
  const result = await userService.getUser(userId);

  // Assert
  expect(result).toEqual(mockUser);
});
```

### Naming conventions

**C# (xUnit):**

Method name follows `MethodUnderTest_Scenario_ExpectedBehaviour`:

```
GetUser_WhenUserExists_ReturnsUserDto
GetUser_WhenUserNotFound_ThrowsNotFoundException
CreateOrder_WhenInsufficientStock_ReturnsBadRequestResult
PlaceOrder_Always_PublishesOrderPlacedEvent
```

**TypeScript (Vitest):**

Use `describe` to group by unit under test; `it`/`test` to describe the scenario in plain English:

```typescript
describe('UserService', () => {
  describe('getUser', () => {
    it('returns the user when found', ...);
    it('throws NotFoundError when user does not exist', ...);
    it('rejects with an error when the repository throws', ...);
  });
});
```

Prefer `it('should ...')` for capability assertions and `it('throws ...')` for error paths.

## Coverage Expectations

### Thresholds

| Metric | Minimum | Target |
|--------|---------|--------|
| Line coverage | 80% | 90%+ |
| Branch coverage | 75% | 85%+ |
| Function coverage | 80% | 90%+ |

These thresholds are enforced by the CI pipeline. A build with coverage below the minimum fails.

### What coverage is and is not

Coverage tells you which code was *executed* by tests — not whether the tests are *meaningful*. 90% coverage with no assertions is nearly worthless. Strive for meaningful coverage:

- Every public method has at least one happy-path test
- Every branch (if/else, switch, null checks) has a test for each significant path
- Every error/exception path is tested

### What to exclude from coverage

- Generated code (EF migrations, protobuf, serialization scaffolding)
- Program entry points (`Program.cs`, `main()`)
- DI registration code (tested implicitly by integration tests)
- DTOs and record types with no behaviour

Mark with `[ExcludeFromCodeCoverage]` (C#) or add to coverage exclusions in `vitest.config.ts`.

## What to Test

### Do test

- **Business logic**: Every rule, calculation, and decision the system makes
- **Error handling**: What happens when dependencies fail, inputs are invalid, or state is unexpected
- **Boundary conditions**: Empty collections, null values, minimum/maximum values, off-by-one
- **Integration points**: The contract between your code and its dependencies (use integration tests, not unit tests with mocks)
- **Public API contracts**: Serialization format, HTTP status codes, event schemas

### Do not test

- **Framework code**: Do not test that ASP.NET routes work — test that your handler returns the right response
- **Third-party library behaviour**: Do not test that `List<T>.Add()` works
- **Trivial getters/setters**: Auto-properties with no logic do not need tests
- **Private methods**: If a private method needs a test, extract it into a testable service

## Mocking

### Use mocks for

- External systems: databases, HTTP APIs, message queues, file systems
- Expensive or slow operations that would make the test suite slow
- Non-deterministic dependencies: clocks, random number generators, GUIDs

### Avoid mocking

- Value objects and domain models — use real instances
- Internal collaborators that you own — test the integrated behaviour instead
- Multiple layers deep — if you need three nested mocks, your class has too many dependencies

### Prefer fakes over mocks where possible

A fake is a simplified but real implementation (e.g., an in-memory repository). Fakes catch more realistic bugs than mocks, which only verify pre-programmed expectations.

```csharp
// Prefer a fake in-memory implementation for repository tests
public class InMemoryUserRepository : IUserRepository
{
    private readonly Dictionary<Guid, User> _store = new();

    public Task<User?> GetByIdAsync(Guid id) =>
        Task.FromResult(_store.GetValueOrDefault(id));

    public Task SaveAsync(User user)
    {
        _store[user.Id] = user;
        return Task.CompletedTask;
    }
}
```

## Test Isolation

Each test must be fully independent:

- No shared mutable state between tests
- No dependency on test execution order
- No dependency on external services (use fakes, test containers, or in-memory implementations)
- Reset state in `BeforeEach` / constructor — do not rely on `AfterEach` teardown for correctness (teardown may not run if the test crashes)

## Test Categories

Structure tests in layers:

| Layer | Purpose | Speed | Isolation |
|-------|---------|-------|-----------|
| **Unit** | Test a single class/function in isolation | < 1ms | Full (all deps mocked/faked) |
| **Integration** | Test multiple components together (e.g., service + real DB) | 10ms–1s | Partial |
| **End-to-end** | Test the full system through its external interface | 1s–30s | None (real infra) |

Keep the unit test layer thick. Integration and E2E tests are more expensive to write and maintain; use them to validate integration contracts, not business logic.

## Test Data

- Use builders or factories to construct test data — avoid large object literals repeated across tests
- Use realistic data that resembles production (use libraries like Bogus/Faker for randomised data)
- Do not use magic numbers or strings without a comment explaining their significance
- Name test data to communicate intent: `var expiredToken = TokenBuilder.Expired();`

## Anti-Patterns to Avoid

| Anti-pattern | Problem | Fix |
|---|---|---|
| Testing implementation details | Breaks on refactoring | Test observable behaviour instead |
| One assertion per test (dogma) | Requires many tests for one scenario | Use multiple assertions for a single scenario, but each `it` block should test one concept |
| Giant test setup | Hard to understand what matters | Extract to helpers/builders; leave only what is relevant to the scenario |
| Asserting on the mock itself | Only verifies a call was made, not the result | Assert on the output and side effects |
| Skipped / ignored tests | Hides failures | Delete or fix skipped tests; never leave `[Fact(Skip = "TODO")]` in the codebase |
| `Thread.Sleep` / `setTimeout` in tests | Flaky and slow | Use async patterns, fake clocks, or proper awaiting |
