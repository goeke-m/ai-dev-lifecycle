# C# Coding Conventions

## Naming

| Construct | Convention | Example |
|-----------|------------|---------|
| Classes, structs, records | PascalCase | `OrderProcessor`, `UserDto` |
| Interfaces | PascalCase with `I` prefix | `IUserRepository`, `IOrderService` |
| Methods | PascalCase | `GetUserAsync`, `ProcessOrder` |
| Properties | PascalCase | `FirstName`, `IsActive` |
| Events | PascalCase | `OrderPlaced`, `UserCreated` |
| Constants | PascalCase | `MaxRetryAttempts`, `DefaultTimeout` |
| Enums and enum members | PascalCase | `OrderStatus.Pending` |
| Private fields | `_camelCase` | `_repository`, `_logger` |
| Local variables | camelCase | `orderTotal`, `userId` |
| Parameters | camelCase | `userId`, `cancellationToken` |
| Type parameters | `T` or descriptive PascalCase | `TEntity`, `TResult` |
| Async methods | Suffix with `Async` | `GetUserAsync`, `SaveOrderAsync` |

## Types and Declarations

### Use record types for DTOs and value objects

Records are immutable by default, provide value equality, and produce concise code:

```csharp
// DTOs (request/response shapes)
public sealed record CreateUserRequest(string Name, string Email);
public sealed record UserResponse(Guid Id, string Name, string Email, DateTimeOffset CreatedAt);

// Value objects with behaviour
public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} to {other.Currency}.");
        return this with { Amount = Amount + other.Amount };
    }
}
```

### Prefer `sealed` where inheritance is not intended

Sealing classes enables compiler optimisations and communicates design intent:

```csharp
// Prefer
public sealed class OrderService : IOrderService { }

// Unless designed for inheritance
public abstract class RepositoryBase<TEntity> { }
```

### Use file-scoped namespace declarations

```csharp
// Prefer (C# 10+)
namespace MyProject.Services;

public sealed class OrderService { }

// Avoid
namespace MyProject.Services
{
    public sealed class OrderService { }
}
```

### Use `var` when the type is apparent

```csharp
// Good — type is obvious from the right side
var user = new User(id, name);
var users = new List<User>();
var count = users.Count;

// Avoid when the type is not apparent
var result = GetData(); // What type does GetData() return?

// Prefer explicit type when clarity matters
OrderStatus status = GetOrderStatus(orderId);
```

## Async / Await

### Always await asynchronous operations

```csharp
// Good
public async Task<User?> GetUserAsync(Guid id, CancellationToken ct = default)
{
    return await _repository.GetByIdAsync(id, ct);
}

// Bad — fire-and-forget without intention
public void SaveUser(User user)
{
    _repository.SaveAsync(user); // Not awaited — exception will be swallowed
}
```

### Always pass CancellationToken through

```csharp
// Good
public async Task<IReadOnlyList<Order>> GetOrdersAsync(
    Guid userId,
    CancellationToken cancellationToken = default)
{
    return await _repository.GetByUserIdAsync(userId, cancellationToken);
}

// Pass the token to every awaitable call
var user = await _userRepository.GetByIdAsync(userId, cancellationToken);
var orders = await _orderRepository.GetByUserIdAsync(userId, cancellationToken);
```

### Use ConfigureAwait thoughtfully

- In library code: use `ConfigureAwait(false)` to avoid deadlocks
- In application code (ASP.NET Core): `ConfigureAwait(false)` is generally not required (no `SynchronizationContext`)
- Be consistent within a project

### Avoid async void

```csharp
// Good
public async Task OnButtonClickAsync() { ... }

// Bad — exceptions are unhandled; cannot be awaited
public async void OnButtonClick() { ... }
```

Exception: event handlers where the signature requires `void`. Even then, wrap the body in try/catch.

## Null Handling

With `<Nullable>enable</Nullable>` in `Directory.Build.props`, the compiler enforces null safety:

### Use nullable reference types correctly

```csharp
// Non-nullable — the compiler ensures this is never null
public string Name { get; init; }

// Nullable — explicitly acknowledges null is a valid value
public string? MiddleName { get; init; }

// Nullable return type — communicate clearly
public async Task<User?> FindByEmailAsync(string email, CancellationToken ct)
{
    return await _db.Users.FirstOrDefaultAsync(u => u.Email == email, ct);
}
```

### Prefer null-conditional and null-coalescing operators

```csharp
// Good
var displayName = user?.DisplayName ?? user?.Email ?? "Anonymous";

// Prefer null-coalescing assignment
_cache ??= new Dictionary<string, string>();

// Check for null with pattern matching
if (result is null)
    return NotFound();

if (result is { IsActive: true } activeUser)
    Process(activeUser);
```

### Avoid null for domain concepts — use Option/Result patterns or exceptions

```csharp
// Prefer explicit not-found handling over returning null from domain methods
public async Task<User> GetUserAsync(Guid id, CancellationToken ct)
{
    var user = await _repository.GetByIdAsync(id, ct);
    return user ?? throw new NotFoundException($"User {id} not found.");
}
```

## Exception Handling

### Throw exceptions for exceptional situations only

Exceptions are not control flow. Do not use them to signal expected domain states (e.g., "user not found" in a search feature).

```csharp
// Good — NotFoundException for a missing required resource
public async Task<User> GetUserByIdAsync(Guid id, CancellationToken ct)
{
    return await _repository.GetByIdAsync(id, ct)
        ?? throw new NotFoundException(nameof(User), id);
}

// Good — return null/Result for optional lookups
public async Task<User?> FindUserByEmailAsync(string email, CancellationToken ct)
{
    return await _repository.FindByEmailAsync(email, ct);
}
```

### Use custom exception types

```csharp
public sealed class NotFoundException : Exception
{
    public NotFoundException(string entityType, object id)
        : base($"{entityType} with id '{id}' was not found.") { }
}

public sealed class ValidationException : Exception
{
    public IReadOnlyDictionary<string, string[]> Errors { get; }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("One or more validation errors occurred.")
    {
        Errors = errors.AsReadOnly();
    }
}
```

### Do not catch and swallow exceptions

```csharp
// Bad — the exception is lost
try
{
    await ProcessOrderAsync(order);
}
catch (Exception)
{
    // silent swallow
}

// Good — log and rethrow, or handle meaningfully
try
{
    await ProcessOrderAsync(order);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Failed to process order {OrderId}", order.Id);
    throw;
}
```

## LINQ

### Prefer LINQ for collection transformations

```csharp
// Good
var activeUserEmails = users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .Select(u => u.Email)
    .ToList();

// Avoid imperative loops for simple transformations
var emails = new List<string>();
foreach (var user in users)
{
    if (user.IsActive)
        emails.Add(user.Email);
}
```

### Use method syntax; avoid query syntax in most cases

```csharp
// Prefer method syntax (more composable)
var result = orders
    .Where(o => o.Status == OrderStatus.Pending)
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Count = g.Count() });

// Query syntax is acceptable for complex joins
var joined = from order in orders
             join customer in customers on order.CustomerId equals customer.Id
             select new { order.Id, customer.Name };
```

### Call `.ToList()` / `.ToArray()` deliberately

Be explicit about when you materialise a query to avoid multiple enumerations:

```csharp
// Good — materialise once, use multiple times
var activeUsers = users.Where(u => u.IsActive).ToList();
var count = activeUsers.Count;
var first = activeUsers.FirstOrDefault();

// Bad — filters the collection twice
var count = users.Where(u => u.IsActive).Count();
var first = users.Where(u => u.IsActive).FirstOrDefault();
```

## Dependency Injection

### Constructor injection only

```csharp
public sealed class OrderService : IOrderService
{
    private readonly IOrderRepository _orderRepository;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository orderRepository, ILogger<OrderService> logger)
    {
        _orderRepository = orderRepository;
        _logger = logger;
    }
}
```

### Register services with the correct lifetime

| Lifetime | When to use |
|----------|-------------|
| `Singleton` | Stateless services shared across all requests (e.g., `IHttpClientFactory`) |
| `Scoped` | Services with per-request state (e.g., `DbContext`, unit-of-work) |
| `Transient` | Lightweight, stateless services created per-resolution |

Avoid `Singleton` services that depend on `Scoped` services — this causes captive dependency issues.

## Logging

### Use structured logging with `ILogger<T>`

```csharp
// Good — structured properties
_logger.LogInformation("Processing order {OrderId} for customer {CustomerId}", order.Id, order.CustomerId);

// Bad — string interpolation defeats structured logging
_logger.LogInformation($"Processing order {order.Id}");
```

### Use LoggerMessage source generators for hot paths

```csharp
// High-performance logging for frequently called paths
public static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Processing order {OrderId}")]
    public static partial void ProcessingOrder(ILogger logger, Guid orderId);
}
```
