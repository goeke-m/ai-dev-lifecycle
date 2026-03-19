# Pull Request Standards

## Philosophy

A pull request is a unit of review, not a unit of work. Keep PRs small, focused, and easy to reason about. Large PRs are harder to review thoroughly, more likely to introduce bugs, and slower to merge.

## PR Size

### Guidelines

| Lines changed | Assessment |
|---------------|------------|
| < 200 | Ideal. Reviewable in a single focused session. |
| 200–400 | Acceptable. May require more time; consider splitting if possible. |
| 400–800 | Large. Justify in the PR description why it cannot be split. |
| > 800 | Exceptional only. Requires prior discussion (e.g., large refactor or migration). |

### What counts

- Lines changed = insertions + deletions in production code
- Generated files (migrations, protobuf output, lock files), test fixtures, and large JSON/YAML datasets are excluded from the mental calculation — but still flag them in the PR description

### How to keep PRs small

- **Stacked PRs**: Build feature branches on top of each other; merge the base first
- **Feature flags**: Merge incomplete features behind a flag so they can ship without being active
- **Separate refactoring from behaviour changes**: Refactoring PRs are easier to review than mixed PRs
- **One concern per PR**: If your branch touches auth, logging, and a new endpoint, consider three PRs

## Title

PR titles must follow [Conventional Commits](https://www.conventionalcommits.org/) format:

```
feat(auth): add OAuth2 login flow
fix(api): return 404 for missing resources instead of 500
```

This allows automated tools to categorise PRs in changelogs and release notes.

## Description Requirements

A PR description must include:

1. **Summary** — One to three sentences: what changed and why
2. **Motivation** — Link to the issue or explain the business/technical reason
3. **Testing** — How you verified the change works (unit tests added, manual test steps, etc.)
4. **Checklist** — Confirm coding standards, tests, and docs are addressed

Use the repository's pull request template. Do not submit a PR with an empty description.

## Coding Standards Compliance

Before opening a PR:

- Run the linter and fix all warnings: `dotnet format --verify-no-changes` or `npm run lint`
- Ensure all tests pass: `dotnet test` or `npm test`
- Ensure coverage thresholds are met
- Check that your commit messages are Conventional Commits compliant

## Review Expectations

### For authors

- Assign relevant reviewers promptly after opening the PR
- Respond to review comments within one business day
- Do not merge without at least one approving review (unless the PR is trivial, e.g., a typo fix)
- Mark your own comments as resolved once addressed
- Leave a comment if you disagree with a review suggestion — do not silently ignore it

### For reviewers

- Review within one business day of being assigned (or communicate if you cannot)
- Distinguish between blocking issues and non-blocking suggestions:
  - **Blocking**: Must be fixed before merge (correctness, security, breaking API contract)
  - **Non-blocking / nit**: Optional improvement, clearly marked as such: `nit: consider renaming this variable for clarity`
- Be specific and constructive — explain *why*, not just *what*
- Approve the PR once all blocking issues are resolved; do not require a second review cycle for nits that were addressed
- Review the diff, not the author — keep feedback impersonal and technical

### Automated checks

All PRs must pass:

- CI build (no compilation errors)
- All tests (no regressions)
- Coverage threshold enforcement
- PR title format check (Conventional Commits)
- Linting / formatting checks

A PR may not be merged while any required check is failing, even with an approval.

## Merge Strategy

**Squash merge is preferred** for feature branches.

- Produces a clean, linear history on `main`
- The squash commit message must be a valid Conventional Commits message (usually matching the PR title)
- Individual commit messages within the branch may be informal ("wip", "address review comments") since they are collapsed

**Merge commit** is acceptable for long-lived integration branches where preserving individual commits matters (e.g., merging `release` into `main`).

**Rebase merge** should be avoided unless the branch has a clean, well-structured commit history that is worth preserving verbatim.

## Branch Lifecycle

- Delete the source branch after merging — keep the branch list clean
- Do not reuse branches (e.g., do not push new work to a merged branch)
- Name branches with the format: `feat/short-description`, `fix/issue-42-login-crash`, `chore/upgrade-dotnet-9`

## Draft PRs

Use draft PRs to:

- Share work-in-progress for early feedback
- Block CI from running on every commit to a feature branch (set to draft until ready)
- Signal that the PR is not yet ready for a merge decision

Convert to "Ready for Review" only when all checklist items are satisfied.
