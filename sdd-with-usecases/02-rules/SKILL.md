---
name: rules
description: Confirm feature-level technical decisions and constraints for a use-case specification, recording validatable rules in spec/rules.md and tracing each rule directly to the use cases it governs.
---

# Technical Rules Skill

Confirm how the application must be built to realize the use cases in `spec/spec.md`. Use cases define behavior; rules
define technical constraints. Do not create acceptance criteria, an implementation task list, file map, or phase plan.

Pipeline position: proposal -> spec -> **rules** -> execute <-> converge (once per use case)

## Role

Resolve technical decisions that an executor should not invent during implementation. Ground each decision in the
actual codebase, confirm judgment calls with the user, and write only constraints that materially change or validate
the implementation. If a decision changes product behavior, route it back to `spec`.

## Inputs

- Use-case specification: `spec/spec.md` (authoritative for behavior)
- Proposal: `spec/proposal.md` (technical intent and background)
- Project guidance: root `AGENTS.md`, `CLAUDE.md`, or `GEMINI.md`, if present
- Build files, source layout, architecture documentation, and existing implementation patterns

## Codebase Grounding

Before proposing rules, inspect the relevant project conventions and versions, including:

- package/module boundaries and dependency direction;
- persistence, transactions, migrations, and concurrency controls;
- error handling and lifecycle guards;
- security configuration and the complete existing route surface;
- tests, fixtures, test datasource isolation, and available test levels;
- UI layout, navigation, form fragments, styles, and message bundles;
- external integrations, observability, and dependency policy.

Rules describe the feature-specific delta from those conventions. Do not restate a project default unless omission
would create a meaningful correctness, security, data, lifecycle, or verification risk.

## Technical Decision Pass

For each use case and its relations, decide whether implementation needs a non-trivial constraint in any of these
areas:

1. Component boundaries and responsibilities
2. Technology or dependency choices
3. Persistence, transactions, migrations, locking, and data fidelity
4. State-transition enforcement and failure representation
5. Security, ownership, input validation, secrets, and the full route-to-access matrix
6. External integration boundaries, timeouts, retries, and deterministic substitutes
7. Presentation integration and localization
8. Testing level, fixtures, negative assertion shape, and runtime-data isolation
9. Observability and prohibited logging
10. Performance, compatibility, deployment, and operational limits

For a relationship, also decide how it is realized without duplicating behavior: included use cases share one
production path; required use cases consume the required postcondition; extending use cases preserve the base use
case's already-approved behavior.

Apply an ecosystem survey where the feature introduces a capability with a canonical framework, multiple plausible
libraries, a new architectural pattern, or long-term coupling. Surface the leanest viable option as well as the
canonical options.

## Interactive Resolution

Ask one technical judgment question at a time. Offer two to four concrete options, including the leanest viable path,
mark the recommendation, and state the trade-off of each. Do not ask when the codebase or an existing project rule
already determines the answer.

If a decision needs an unavailable stakeholder, record the question, blocker, and default under `External
dependencies`. Do not hide an unresolved decision behind `SHOULD`.

## Required Rule Areas

The following are mandatory when applicable, even if they partially repeat a convention:

- Lifecycle: allowed transitions and service/domain-level refusal of every other transition, including the named
  exception or result and prohibited side effects.
- Security: a full route-to-access table for the whole affected application surface, including existing routes,
  login/logout, static content, and explicit public access.
- Normative data: one source of truth from `spec.md`, fresh-database verification by value, absence of extra values,
  and credential verification when secrets are seeded.
- Presentation: reuse of layout/navigation/form conventions and localization of every user-visible string.
- Verification: how each use case is proven at its real boundary, including its main scenario, extensions, guarantees,
  postconditions, and relations.

## Verification Rule

The testing strategy must establish these constraints without generating per-file tasks:

1. Tests use isolated fixtures or a dedicated datasource and never modify runtime data or tracked generated files.
2. Each use case has an end-to-end test of its main success scenario at the outer boundary the actor uses. State is
   asserted after every consequential step.
3. Each extension has a test at the lowest level that still proves its observable response. State-changing,
   authorization, routing, and integration extensions are tested at their real boundary.
4. Negative behavior proves denial plus absence of disclosure, persistence, emission, or collaborator invocation as
   applicable. Status-only evidence is insufficient.
5. Lifecycle refusal tests set up real state rather than stubbing it. Data exactness tests compare rows and fields by
   value. Boundary requirements exercise within, at, and beyond the boundary.
6. Relationship tests prove included behavior uses the same production path, required postconditions are consumed,
   and extending use cases do not regress the base use case.
7. Test doubles conform to production contracts: the same exceptions, nullability, option semantics, and field shape.
8. UI use cases require a human walkthrough after automated verification; the script comes directly from the use
   case's scenario and extensions.

The rule may strengthen or specialize these levels for the project. It must not weaken them to whatever tests already
exist.

## Rule Format

Use stable IDs in document order.

```markdown
### RULE-1 - <short title>

- Applies to: UC-1, UC-3 | all use cases
- Constraint: MUST | MUST NOT | SHOULD | SHOULD NOT | MAY <concrete, validatable statement>.
- Reason: <why this decision exists>.
- Verification: <observable code, test, configuration, or runtime evidence that can prove it>.
```

Use RFC 2119 force deliberately. `MUST` and `MUST NOT` are non-negotiable. A `SHOULD` allows deviation only with an
explicit rationale in `spec/status.md`. A `MAY` records a genuine option and must not masquerade as a requirement.

A rule passes the recording bar only when it is feature-specific, documents a deliberate deviation or negative
decision, binds one or more use cases to a technical invariant, or prevents a concrete implementation risk. Statements
such as `follow best practices`, `handle errors properly`, or `use the existing framework` are not rules.

Trace rules directly to use cases with `Applies to`. Use `all use cases` only for a truly cross-cutting constraint.
Never use AC identifiers.

## Route-Back Triggers

Return to `spec` when:

- a technical rule would have to choose actor-visible behavior;
- a use case or extension is ambiguous or untestable;
- a relation target is missing, circular, or does not state the postcondition it supplies;
- a lifecycle lacks a complete state model;
- normative data is referenced but not copied into `spec.md`;
- authentication is introduced without an access decision for existing routes;
- a loaded word has no observable definition.

## Completion Gate

Before writing, verify:

- Every use case and relationship was considered for technical risk.
- Every judgment call from the ecosystem survey was confirmed interactively.
- Every rule meets the recording bar and has ID, applicability, modal constraint, reason, and verification.
- Every referenced UC exists and every UC appears in the cross-reference table, even if it needs no special rule.
- Required lifecycle, security, normative-data, presentation, and verification rules exist when applicable.
- Rules are mutually consistent, compatible with the codebase, and do not change use-case behavior.
- No rule references criteria, reviews, tasks, phases, artifact maps, or files that a future executor is required to
  create by exact name.
- The design overview is under 200 words.

Do not write a partial rules file.

## Output

Write `spec/rules.md`:

```markdown
# Technical Rules: <feature>

## Design overview
<components, boundaries, flow, and dependencies in under 200 words>

## Codebase alignment
<inherited conventions and justified deviations>

## Security surface
<full route-to-access table when applicable>

## Verification strategy
<project-specific implementation of the Verification Rule>

## Rules
<flat RULE-n list>

## Use-case cross-reference
| Use case | Rules |
|---|---|
| UC-1 | RULE-1, RULE-3 |

## Design exclusions
<technical concerns explicitly out of scope>

## External dependencies
<question, blocker, and default; omit when empty>
```
