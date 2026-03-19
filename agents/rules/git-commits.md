# Git Commit Conventions

## Standard

All commits in this repository **must** follow the [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) specification. This enables automated changelog generation, semantic versioning, and clear communication of intent.

## Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Required components

| Component | Rules |
|-----------|-------|
| **type** | Lowercase. One of the allowed types listed below. |
| **description** | Present tense. Imperative mood. No capital first letter. No period at the end. Max 100 characters. |

### Optional components

| Component | Rules |
|-----------|-------|
| **scope** | Lowercase. Wrapped in parentheses. Describes the area of the codebase affected. |
| **body** | Separated from description by a blank line. Explains *what* and *why*, not *how*. Wrap at 72 characters. |
| **footer** | `BREAKING CHANGE: <description>`, `Closes #<issue>`, `Reviewed-by: <name>` |

## Allowed Types

| Type | When to use |
|------|-------------|
| `feat` | A new feature visible to the end user or a new public API |
| `fix` | A bug fix visible to the end user |
| `docs` | Documentation-only changes (README, comments, JSDoc, XML docs) |
| `style` | Code style changes that do not affect meaning (formatting, whitespace, semicolons) |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Code change that improves performance |
| `test` | Adding or updating tests; no production code change |
| `chore` | Changes to the build process, tooling, or auxiliary tasks |
| `ci` | Changes to CI/CD configuration (GitHub Actions, pipelines) |
| `build` | Changes that affect the build system or external dependencies |
| `revert` | Reverts a previous commit (include `This reverts commit <hash>` in the body) |

## Breaking Changes

Append `!` after the type/scope to signal a breaking change:

```
feat(api)!: remove deprecated /v1/users endpoint
```

Or use the `BREAKING CHANGE:` footer (or both):

```
feat(auth)!: replace JWT with session tokens

BREAKING CHANGE: The Authorization header format has changed from
`Bearer <jwt>` to `Session <token>`. All existing tokens are invalid.
Clients must re-authenticate to obtain a session token.

Migration guide: docs/migration/v2-auth.md
```

A commit with a `BREAKING CHANGE` footer triggers a major version bump in automated versioning.

## Scope Guidelines

Scopes should match the top-level areas of the codebase. Agree on scopes as a team and keep the list small.

Common scopes:

- `auth` — Authentication / authorization
- `api` — Public API surface
- `db` / `data` — Database layer
- `ui` — User interface components
- `config` — Configuration handling
- `deps` — Dependency updates
- `ci` — CI/CD pipeline changes
- `docs` — Documentation system
- `tests` — Test infrastructure

## Examples

### Simple feature

```
feat(auth): add OAuth2 support for GitHub and Google
```

### Bug fix with issue reference

```
fix(api): return 404 instead of 500 when resource not found

The GET /items/:id endpoint was returning HTTP 500 when the item
did not exist in the database, causing false positive alerts.

Closes #142
```

### Dependency update

```
chore(deps): bump typescript from 5.3.3 to 5.4.5
```

### Refactor with scope

```
refactor(auth): extract token validation into dedicated service

Move JWT validation logic out of the middleware and into
AuthTokenService to improve testability and separation of concerns.
No behavioral change.
```

### CI change

```
ci: add coverage reporting to pull request checks
```

### Revert

```
revert: feat(auth): add OAuth2 support for GitHub and Google

This reverts commit a1b2c3d4 because the OAuth callback URL
misconfiguration caused authentication failures in production.
```

## What NOT to do

```
# Too vague
fix: bug fix
chore: changes
update: stuff

# Wrong tense / capitalization
feat: Added user authentication
feat: Adds user authentication.

# Description too long (>100 chars)
feat(auth): add a very detailed and comprehensive OAuth2 implementation supporting multiple providers

# Missing type
add OAuth2 support

# Using the wrong type
feat: fix null reference exception   ← should be `fix:`
chore: add new search feature        ← should be `feat:`
```

## Enforced automatically

The `commit-msg` git hook (installed by `apply.sh`) validates every commit message against the Conventional Commits pattern before it is recorded. If your message does not conform, the commit is rejected with a helpful error showing correct examples.
