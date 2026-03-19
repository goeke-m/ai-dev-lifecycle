# ADR-{{NUMBER}}: {{TITLE}}

- **Date:** {{DATE}}
- **Status:** {{STATUS}}
- **Deciders:** {{DECIDERS}}
- **Consulted:** {{CONSULTED}}
- **Informed:** {{INFORMED}}

<!-- Status options: Proposed | Accepted | Deprecated | Superseded by [ADR-XXX](ADR-XXX.md) -->

---

## Context

<!--
Describe the situation and the forces at play. What is the issue that motivates
this decision? What constraints are you working within (technical, business,
team, time)?

Write in present tense. Describe the problem, not the solution.

Example:
We are building a REST API that will be consumed by both a web client and a
mobile client. We need to decide how to handle authentication. The API must
support third-party integrations, and the team has limited experience with
cryptographic libraries.
-->

## Decision

<!--
Describe the decision that was made. State it clearly and actively.
Use present tense: "We will use..." not "We have decided to use..."

Example:
We will use JSON Web Tokens (JWT) with RS256 signing for authentication.
Tokens will be issued by our own auth service and validated by all other services.
The public key will be distributed via a JWKS endpoint.
-->

## Alternatives Considered

<!--
List the other options that were evaluated. For each option, briefly explain
what it is and why it was rejected.

Example:

### Option A: Session-based authentication
Store session data server-side; client holds a session cookie.
**Rejected because:** Does not scale well across multiple service instances
without sticky sessions or a shared session store. Adds infrastructure complexity.

### Option B: OAuth2 with an external provider (Auth0, Cognito)
Delegate authentication entirely to a third-party service.
**Rejected because:** Introduces an external dependency and cost for an internal
API. Overly complex for the current user base size.
-->

## Consequences

<!--
Describe the resulting context after applying the decision. All consequences
should be listed — both positive and negative.

### Positive
- ...
- ...

### Negative
- ...
- ...

### Risks
- ...
-->

### Positive

-

### Negative

-

### Risks

-

## Implementation Notes

<!--
Optional. Include any specific implementation guidance, gotchas, or links to
code that implements this decision.
-->

## References

<!--
Link to relevant documents, issues, RFCs, library documentation, or prior art
that informed this decision.
-->

-
