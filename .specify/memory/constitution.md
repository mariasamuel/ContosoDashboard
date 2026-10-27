<!--
Sync Impact Report
Version change: unratified template -> 1.0.0 (initial adoption)
Modified principles: placeholders -> I. Training Focus; II. User Isolation and Authorization;
	III. Layered Design; IV. Verifiable Changes; V. Deliberate Scope
Added sections: Additional Constraints; Development Workflow
Removed sections: none
Templates: plan-template.md reviewed, no change; spec-template.md reviewed, no change;
	tasks-template.md updated; constitution-template.md reviewed, no change;
	checklist-template.md reviewed, no change
Workflow guidance: speckit.tasks.agent.md updated; speckit.implement.agent.md updated
Command templates: .specify/templates/commands/ does not exist; command guidance is in .github/agents/
Follow-up TODOs: none
-->
# ContosoDashboard Constitution

## Core Principles

### I. Training Focus
This repository MUST remain a fictional training application. Features MUST serve the
training purpose and MUST NOT describe mock behavior as production-ready. Cloud services,
real identity providers, and production deployment are out of scope unless a feature
specification explicitly includes them.

### II. User Isolation and Authorization
Every new protected page MUST require authentication. Services that read or mutate
user-owned data MUST enforce the requesting user's access at the service boundary;
page-level authorization alone is insufficient. Access-control changes MUST cover
anonymous access and unauthorized cross-user or cross-project access in verification.

### III. Layered Design
UI behavior belongs in Blazor pages and shared components, business rules and
authorization belong in services, persistence belongs in the data layer, and domain
entities belong in models. New code MUST follow these boundaries and use dependency
injection for service dependencies. A feature MUST NOT add an external infrastructure
dependency when an existing local implementation meets its stated requirements.

### IV. Verifiable Changes
Behavior changes MUST include focused automated tests when a suitable test harness exists.
When no suitable harness exists, the plan or task list MUST specify reproducible manual
checks and disclose the automated-coverage gap. Changes to authorization MUST include
checks for permitted and denied access. Application changes MUST pass the applicable
build and test commands before completion.

### V. Deliberate Scope
Specifications MUST state user value, testable functional requirements, acceptance
scenarios, and relevant edge cases. Implementation MUST stay within the approved scope;
unrelated cleanup and new abstractions require separate justification. Success criteria
MUST be objectively verifiable rather than relying on unmeasured terms such as "fast" or
"secure."

## Additional Constraints
The application targets .NET 8, ASP.NET Core, Blazor Server, and Entity Framework Core
with SQL Server LocalDB for local training. The default workflow MUST remain usable
without cloud accounts or external service dependencies. Authentication is a mock,
password-free user-selection flow; it MUST NOT be treated as an identity security
solution. Secrets and real credentials MUST NOT be committed to the repository.

## Development Workflow
Feature work MUST begin with a Spec Kit specification under `specs/` before implementation.
The specification MUST be followed by a technical plan and dependency-ordered tasks when
the change spans user behavior, data, services, authorization, or multiple pages. A
localized bug fix may use a smaller workflow when its scope and verification are clear.
Plans MUST evaluate the constitution, tasks MUST include the required verification, and
implementation MUST update task status and relevant documentation. Reviewers MUST check
the change against its specification and this constitution.

## Governance
This constitution governs project-specific engineering decisions. Amendments require
explicit project-owner approval, a documented rationale, a semantic version update, and
synchronization of affected templates and `.github/agents/` guidance in the same change.
Versioning uses MAJOR for incompatible principle or governance changes, MINOR for new or
materially expanded principles, and PATCH for clarifications that do not change meaning.
Every feature plan MUST record constitution compliance; any deviation MUST be justified
in the plan and accepted during review. The training-only and non-production boundary
must be retained in user-facing project guidance.

**Version**: 1.0.0 | **Ratified**: 2026-10-26 | **Last Amended**: 2026-10-26
