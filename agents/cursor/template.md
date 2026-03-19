---
description: Development conventions and rules for {{PROJECT_NAME}}
globs:
  - "**/*.cs"
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
alwaysApply: true
---

# {{PROJECT_NAME}} — Cursor Rules

{{DESCRIPTION}}

**Language:** {{LANGUAGE}}
**Active modules:** {{ENABLED_MODULES}}

---

## Commands

```
Build  : {{BUILD_CMD}}
Test   : {{TEST_CMD}}
Lint   : {{LINT_CMD}}
```

---

## Conventions

Follow these rules in all code suggestions, completions, and edits. They are enforced by CI and git hooks.

{{RULES_CONTENT}}

---

## Architecture

<!-- Populate this section with project-specific context to help Cursor
     generate code that fits the existing structure.

Suggested content:
- What framework / runtime is used (ASP.NET Core, Fastify, Next.js, etc.)
- Primary design patterns (repository, mediator, CQRS, etc.)
- Folder/layer conventions
- Key third-party libraries and how they are used
-->

### Project structure

```
# Describe the folder layout here
```

### Patterns in use

<!-- Example:
- All handlers implement IRequestHandler<TRequest, TResponse> (MediatR)
- All repositories implement IRepository<TEntity, TId>
- Validation is done with FluentValidation in the command/query layer
-->

---

## Cursor-Specific Guidance

### Code completion

- Prefer completing based on patterns in adjacent files over guessing from scratch
- When generating a new class, look at an existing similar class first
- Always include the namespace / imports that match the file's location

### Refactoring

- When suggesting a rename, check all usages across the codebase
- When extracting a method, preserve the original behaviour exactly
- When converting to async, add `CancellationToken ct = default` to the signature

### What to avoid

- Do not suggest `TODO` comments in generated code — either implement fully or ask for clarification
- Do not generate commented-out code blocks
- Do not suggest `any` in TypeScript or `dynamic` in C#
- Do not suppress warnings without a comment explaining the suppression
