# GitHub Copilot Instructions: {{PROJECT_NAME}}

{{DESCRIPTION}}

**Language:** {{LANGUAGE}}
**Active modules:** {{ENABLED_MODULES}}

These instructions apply to all Copilot suggestions in this repository. Follow them when generating, completing, or refactoring code.

---

## Commands

| Action | Command |
|--------|---------|
| Build | `{{BUILD_CMD}}` |
| Test | `{{TEST_CMD}}` |
| Lint / Format | `{{LINT_CMD}}` |

---

## Code Generation Rules

When generating or completing code in this repository, Copilot must follow the rules below. These are enforced by the CI pipeline and git hooks.

{{RULES_CONTENT}}

---

## Architecture Context

<!-- Add project-specific architecture notes here after initial setup.
     Copilot uses this context to generate code that fits the existing patterns.

Examples of useful content:
- "This is an ASP.NET Core 9 minimal API. All endpoints are defined in Program.cs using app.MapGroup()."
- "We use the Repository pattern. All repositories implement IRepository<TEntity>."
- "Domain events are published via MediatR. Use INotification for events and INotificationHandler for handlers."
- "This is a Node.js ESM project. All imports must include the .js extension."
-->

### Project layout

```
# Describe your project structure here so Copilot understands where things go.
```

---

## Copilot Behaviour Guidance

### Always

- Use `async`/`await` — never `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()` in async contexts
- Include XML doc comments (`///`) on all public APIs in C# projects
- Include JSDoc comments on all exported functions in TypeScript projects
- Use the existing patterns in the codebase — look at adjacent files before generating new code
- Emit types explicitly on public API boundaries; rely on inference only for local variables
- Pass `CancellationToken` through all async call chains in C# code

### Never

- Generate code that uses `any` in TypeScript
- Generate code that disables nullable reference types (`#nullable disable`, `!` assertions without justification)
- Generate `Thread.Sleep` or equivalent blocking waits
- Generate hardcoded connection strings, API keys, or credentials
- Suppress analyzer warnings without a code comment explaining why
- Generate `catch (Exception)` blocks that swallow the exception silently

### When writing tests

- Follow Arrange / Act / Assert structure
- Name test methods using the convention in the testing rules above
- Use the test libraries already present in the project — do not suggest adding new ones
- Assert on behaviour and outputs, not on internal method calls

### When handling errors

- Use the custom exception types defined in the project (not raw `Exception` or `ApplicationException`)
- Log errors with structured properties, not string interpolation
- Return appropriate HTTP status codes from API endpoints
