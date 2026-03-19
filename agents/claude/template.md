# Project: {{PROJECT_NAME}}

{{DESCRIPTION}}

**Language:** {{LANGUAGE}}
**Active modules:** {{ENABLED_MODULES}}

---

## Commands

Use these commands when working on this project. Always run them from the repository root unless otherwise noted.

### Build

```
{{BUILD_CMD}}
```

### Test

```
{{TEST_CMD}}
```

### Lint / Format

```
{{LINT_CMD}}
```

---

## Architecture

<!-- This section is intentionally left for project-specific content.
     Update it after running apply.sh to describe the high-level architecture:
     - Main layers / projects
     - Key design patterns (CQRS, mediator, repository, etc.)
     - External dependencies (databases, message queues, external APIs)
     - Data flow overview
-->

### Project structure

```
# Add your project structure here, e.g.:
# src/
#   MyProject/          ← Application entry point
#   MyProject.Core/     ← Domain logic
#   MyProject.Data/     ← Data access
# tests/
#   MyProject.Tests/    ← Unit tests
```

### Key design decisions

<!-- Reference ADRs from docs/adr/ for major architectural choices. -->

---

## Conventions

The rules below are enforced automatically (by git hooks and CI) and should be followed in all code suggestions, completions, and generated code.

{{RULES_CONTENT}}

---

## Working with this project

### Adding new features

1. Create a branch: `git checkout -b feat/<short-description>`
2. Implement the feature, following the conventions above
3. Write tests covering the new behaviour
4. Run the full test suite to confirm no regressions
5. Open a PR using the repository's PR template

### Fixing bugs

1. Write a failing test that reproduces the bug
2. Fix the bug
3. Confirm the test now passes
4. Check for similar bugs in related code
5. Open a PR with `fix:` commit type

### What Claude should and should not do

- **Do** suggest code that conforms to the conventions in this file
- **Do** flag potential issues: null safety, missing error handling, missing tests
- **Do** offer refactoring suggestions that improve clarity without changing behaviour
- **Do not** suggest using `var` in TypeScript / `any` in TypeScript
- **Do not** generate code that bypasses the nullable reference type system
- **Do not** suggest committing generated files or secrets
- **Do not** use `Thread.Sleep` or raw `Task.Delay` in production code without a strong justification
