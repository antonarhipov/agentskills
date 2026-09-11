---
name: rules
description: Capture feature-level design decisions and the technical constraints that follow, traceable to acceptance criteria
---

# Technical Design and Constraints Skill

Translate spec and criteria into the design shape for this feature and the constraints that follow. Constraints specify HOW the system should be built, complementing acceptance criteria which specify WHAT.

Pipeline position: proposal → spec → criteria → **rules** → review → tasks → review (plan) → execute ⇄ converge

# Role

You make the feature-level design decisions an implementing agent would otherwise invent during coding, and record them as validatable constraints. You do not invent product decisions. If a rule would require resolving a product ambiguity, route the gap back to the spec step.

# Inputs

- Resolved spec: @file:spec/spec.md
- Acceptance criteria: @file:spec/criteria.md
- Proposal: @file:spec/proposal.md (tech stack and intent reference)
- Project conventions:
  - `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` at the project root, if present
  - Top-level build files (Gradle, Maven, package.json) for stack and versions
  - Existing source layout, ADRs, `docs/architecture` if present

Spec and criteria take precedence over the proposal.

# Codebase Grounding (run first)

Before writing any rule, read agent guidance files and note established package layout, frameworks and versions, and existing patterns for persistence, error handling, logging, testing, **security configuration** (the existing route→access mapping), and **presentation** (layout fragment, menu fragment, form-field fragments, stylesheet, message bundles). Rules align with existing conventions unless there is a documented reason to deviate.

This step is non-negotiable. The output of rules is the diff against conventions; you cannot write a diff without reading what you're diffing against. A convention you did not read cannot be inherited; the executor will invent its own.

# Analysis Pass (internal)

Identify the actual decisions this feature requires. Walk the coverage checklist below as prompts, not as a forced output structure. For each item, ask: "Does this feature need a non-trivial decision here, or does it inherit from AGENTS.md, skills, and existing patterns?"

Coverage checklist:

1. Project Structure: packages, modules, layering, boundaries
2. Component Design: classes, interfaces, responsibilities
3. Technology Decisions: specific libraries, versions, configurations
4. Code Style: naming, formatting, file organization
5. Design Patterns: which to apply, which to avoid
6. Error Handling: result types vs exceptions, boundaries, atomicity; **for every lifecycle enum, the allowed transitions and the named exception/result with which the *service* (not the UI) refuses all others**
7. Testing Strategy: pyramid composition, mocking policy, fixtures, frameworks — written with the *Testing rule template* below (test data isolation, test level per AC pattern, assertion shape for negative and data ACs, end-to-end tests derived from the spec's use cases)
8. Security: the **full URL→role matrix for the whole application** (pre-existing routes, static resources, login/logout included), stated as a table in the rule with every `permitAll` as an explicit row; input validation, secrets, PII
9. Observability: what to log (and not), structured format, metrics, traces
10. Concurrency: thread safety, async patterns, blocking call rules
11. Data Persistence: transactions, migrations, query patterns, schema evolution; **seed provenance** — the single normative source of seed/reference data (the copied table in `spec.md`) and the requirement that the migration test asserts every row by value and every credential with the encoder
12. API Contracts: versioning, backward compatibility, deprecation
13. Performance Budgets: resource targets tied to non-functional ACs
14. Dependency Policy: new deps to add, deps to avoid, version pinning
15. Presentation: layout and fragment reuse, navigation entries per role, localization of all user-visible text (templates, attributes, messages produced in code) — required whenever the feature ships templates

If an item is fully covered by AGENTS.md, skills, or existing patterns, do not produce a rule for it. Inheritance is the default; rules capture only the diff. Items 6 (state guards), 7, 8, 11 (seed provenance) and 15 are exceptions: whenever the feature has a lifecycle enum, tests, routes, seed data or templates, a rule is **required** even if it restates a convention, because these are the categories the executor most often narrows silently.

# Testing Rule Template

The testing strategy rule(s) MUST fix all four of the following. "Unit tests with Mockito, integration tests with
`@SpringBootTest`" is not a strategy.

1. **Test data isolation** — which datasource/fixtures tests use; that the runtime database or data files are never
   read or written by tests; that the working tree is unchanged after the suite.
2. **Test level per AC pattern** — a table: negative authz ACs → web-slice test with the real security configuration
   over the **whole** URL space; state-refusal ACs → service-level test that sets up the state (not stubs it) and
   asserts refusal plus `never()` on collaborators; lifecycle/end-to-end ACs → **one HTTP-level test per use-case main
   success scenario**, executed as each actor the scenario names, through the real endpoints on the isolated datasource,
   asserting state after every step; plus **one leg per extension that changes state** (an extension that only displays
   may be covered by the step's own AC test). The rule **copies the numbered step list** from `spec.md` §Use cases — it
   does not point at it; data-exactness ACs → migration test on a fresh database; fidelity ACs → round-trip test
   comparing every field; boundary ACs → three points.
3. **Assertion shape for negative ACs** — status **and** absence of disclosure (response body does not contain the
   protected data) **and** absence of mutation (`never()` / unchanged state). Status-only assertions do not satisfy a
   negative AC.
4. **Test double contract** — doubles conform to the production interface (same exceptions, nullability, `Optional`
   semantics); a test may not stub a collaborator to return a value production cannot produce.

A rule covering more than five ACs must contain a per-AC-pattern validation table or be split; "Covers: AC-1 … AC-13"
with one sentence is coverage by pointer, not a constraint.

# Ecosystem Survey

For each checklist item that touches technology or architecture (3, 5, 6, 7, 9, 10, 11, 12, 15), apply the Ecosystem Survey lens before deciding:

- Is there a canonical framework for this category that the project doesn't use? (Spring Batch for batch jobs; Spring Integration for messaging; Liquibase/Flyway for migrations; Testcontainers for integration testing; Resilience4j for retries)
- Are there multiple equally-valid library options for a required capability that the project hasn't already chosen? (CSV parser, HTTP client, JSON library)
- Does the feature require an architectural pattern not present in the codebase? (async, streaming, event-driven, distributed transactions)
- Does the decision have long-term coupling beyond this feature?

If yes to any, the decision is a judgment call. It MUST go through Interactive Resolution. Do not silently default to "what the project already has." Staying lean is a valid choice, but it must be a recorded one.

The survey is not only about libraries. Also flag for Interactive Resolution any decision the spec or criteria leave
to a loaded word — *full*, *complete*, *coherent*, *appropriate*, *minimal*, *all* — applied to a screen, data set or
test suite ("full calendar", "complete interpretation", "coherent UI"). Either the rule defines the word concretely
(the parts that must be present) after asking, or the gap is routed back to spec. Never let the executor pick the
smallest reading.

# Interactive Resolution

Some decisions derive mechanically (naming matches convention, test framework matches the project). Others require judgment, including every decision flagged by the Ecosystem Survey.

For judgment calls, you **MUST** use AskUserTool. Do not silently default. Provide 2 to 4 concrete options that always include the leanest viable path (no new framework, no new dependency). Mark your pick "(recommended)" with a one-line reason and a one-line trade-off per option. **One question at a time.** Re-plan after every answer.

If a decision needs a stakeholder you cannot reach, record under "External dependencies" with the question, blocker, and default in use until resolution.

# Worth-Recording Bar

Only emit a rule if at least one holds:
- It captures a feature-specific decision not already in AGENTS.md, skills, or build config
- It documents a deliberate deviation from project conventions
- It binds a specific AC to a technical constraint that validates it
- It records a deliberate choice NOT to adopt an ecosystem option that was surfaced (e.g., "decided against Spring Batch; uses @Scheduled and JdbcTemplate")

Restating project-wide defaults is noise. If the rule would just say "use the framework the rest of the project uses," skip it. Negative decisions, on the other hand, are the most often-lost context and must be recorded.

# Constraint Language (RFC 2119)

- **MUST**: non-negotiable; violating breaks a requirement, AC, or invariant
- **MUST NOT**: known antipattern, security or correctness hazard
- **SHOULD / SHOULD NOT**: strong preference; deviation requires inline justification
- **MAY**: optional, no preference

If every rule in the output is at one level, you've lost signal. Use the scale.

# Rule Format

Each rule has:
- Stable ID: `RULE-1`, `RULE-2`, ... in document order
- Modal verb
- Concrete, validatable statement
- `Reason:` line on why it exists
- `Covers:` line listing AC(s), or `Covers: project-wide` for cross-cutting rules

Example:

### RULE-7
**Covers:** AC-3, AC-4
**MUST** place domain logic under `com.acme.invoice.domain` and depend only on Java stdlib and Kotlin stdlib.
**Reason:** Keeps domain free of framework coupling; testable without Spring context.

Example of a negative-decision rule:

### RULE-2
**Covers:** project-wide
**MUST NOT** introduce Spring Batch as a dependency.
**Reason:** Surveyed as the canonical batch framework; declined because the import is single-source, single-table, with acceptable manual restart logic via a checkpoint column. Reconsider if requirements grow to multi-source, partitioned, or long-running imports.

# Anti-patterns

**Bad:** `MUST be well-architected` → **Better:** `MUST place domain logic in com.acme.feature.domain; MUST NOT depend on Spring from this package.` "Well-architected" isn't validatable.

**Bad:** `MUST handle errors properly` → **Better:** `MUST return Result<T, DomainError>; MUST NOT throw across the application service boundary.` Concrete subjects and verbs.

**Bad:** `MUST follow best practices` → drop, or name the specific practice. Unfalsifiable rules are worse than no rule.

**Bad:** restating an AGENTS.md convention → drop. If the rule would say "use the framework the rest of the project uses," inheritance handles it.

**Bad:** rationale stuffed into the rule statement → keep the rule one concrete sentence; rationale belongs in `Reason:`.

**Bad:** `MUST require authentication for all scheduling endpoints` → **Better:** the full URL→role table including `/`, `/owners/**`, `/vets/**`, `/login`, static resources, each with its access level. A matrix that lists only the new prefixes leaves pre-existing routes to accident.

**Bad:** `Reason: guarded service transitions` on a rule whose statement says nothing about guards → **Better:** `MUST refuse <action> from any state other than <states> in the service with <named exception>; MUST NOT rely on the UI to hide the action.` Verification intent belongs in the statement, where it is checkable.

**Bad:** `MUST write negative-path tests for AC-1 … AC-13` → **Better:** the Testing Rule Template: level, fixture, and the exact assertion shape (`403` **and** body excludes the protected data **and** `never()` on the service). "Negative test" alone is satisfied by asserting a status code.

**Bad:** `MUST seed the data in proposal.md` → **Better:** `MUST seed exactly the rows in spec.md §Normative data; the migration test MUST assert each row by value and each credential with the encoder.` Pointers cannot be verified; counts prove nothing.

**Bad:** `MUST write an end-to-end lifecycle test` → **Better:** `MUST implement one HTTP-level test per UC main scenario: UC-1 steps 1–5 as owner, then UC-3 steps 1–4 as staff …` with the steps copied in. "Lifecycle" without the steps lets the executor pick which path counts.

**Bad:** defaulting to "stay with what's in build.gradle" for a category where a canonical ecosystem solution exists, without surfacing the choice → always run the Ecosystem Survey first.

# Route-Back Triggers

Route back to an earlier step when any of:
- A rule would require inventing a product decision (route to spec)
- An AC is too vague to derive a technical constraint from (route to criteria)
- A boundary, dependency, or pattern decision keeps oscillating between two equally valid options with no AC to disambiguate (route to criteria; the spec likely under-constrained the behavior)
- The spec references normative data by pointer, has no state table for a lifecycle entity, or does not state the access policy for pre-existing pages when the feature introduces authentication (route to spec)
- Negative authz ACs are scoped to the feature's objects rather than the whole application surface (route to criteria)
- The spec has a lifecycle entity but no use case, or a use-case extension has no AC (route to spec / criteria)

Don't paper over a missing decision with a `SHOULD` rule. That hides the gap.

# Success Criteria

Complete only when ALL hold:

- Every coverage checklist item considered during analysis (covered, inherited, or noted internally as not applicable)
- Every technology or architecture item that triggered the Ecosystem Survey went through Interactive Resolution
- Every rule passes the Worth-Recording Bar
- Every rule has ID, modal, concrete statement, `Reason:`, `Covers:`
- Every rule is validatable by reading code (no subjective adjectives)
- Every AC is either covered by at least one RULE or explicitly marked as needing none; any rule covering more than five ACs has a per-AC-pattern validation table
- The Testing Rule Template is satisfied: isolation, level per AC pattern, negative assertion shape, double contract
- If the spec has use cases, the testing rule names one HTTP-level test per main success scenario with the steps copied in, and one leg per state-changing extension
- If the application has routes, a rule contains the full URL→role matrix as a table covering every route in the codebase, pre-existing ones included
- If the feature has a lifecycle enum, a rule lists the allowed transitions and the service-level refusal of all others
- If the feature has seed/reference data, a rule names the normative source in `spec.md` and requires by-value assertion of every row and every credential
- If the feature ships templates, a rule fixes layout/fragment reuse, per-role navigation and localization of all user-visible text
- Every loaded word inherited from spec/criteria is defined in a rule or routed back
- No rule references a decision, table or definition by pointer to another file; the normative content is copied in
- Rules align with project conventions, or deviation is justified in `Reason:`
- The Design section communicates the feature's shape in under 200 words

Run a verification pass before writing. Do not write a partial file.

# Output

Write to `spec/rules.md`:

```
# Technical Design and Constraints: <Feature>

## Overview
Feature in one sentence. Tech stack with versions. Links to spec/spec.md and spec/criteria.md.

## Design
Components: <new components introduced by this feature, with one-line responsibility each>
Boundaries: <what this feature exposes and what it keeps internal>
Flow: <one paragraph or numbered steps describing how data moves through the components>
Key dependencies: <new libraries to add, with reason; existing libraries to use; ecosystem options considered and declined>

## Codebase Alignment
One paragraph: conventions inherited from AGENTS.md, skills, and existing patterns — including the presentation conventions (layout fragment, menu fragment, form fragments, stylesheet, message bundles) and the existing security configuration. Deviations with justification.

## Security Surface
| Route (pattern) | Pre-existing? | Access | Rule |
|---|---|---|---|
| `/login`, static resources | yes | public | RULE-n |
| `/owners/**` | yes | … | RULE-n |
| `/<feature>/**` | no | … | RULE-n |

(every route in the codebase; `permitAll` rows are explicit)

## Testing Strategy
| AC pattern | Test level | Fixture / datasource | Assertion shape |
|---|---|---|---|
| Negative authz | web slice + real security config, whole URL space | mocked services | status **and** no disclosure **and** `never()` |
| State refusal | service | state set up, not stubbed | named exception **and** `never()` save |
| Lifecycle end-to-end | HTTP, real endpoints, as each actor in the UC | isolated in-memory DB | one test per UC main scenario (steps copied below), one leg per state-changing extension; state asserted after each step |
| Data exactness | migration on fresh DB | — | every row by value; credentials via encoder |
| Fidelity | round-trip | — | every field equal |
| Boundary | unit | — | within / at / beyond |

### End-to-end scenarios
| UC | Test name | Steps (copied from spec.md §Use cases) | Actors |
|---|---|---|---|
| UC-1 | `<TestClass>.<method>` | 1. … 2. … 3. … (+ ext 4a, 5a) | owner, staff |
| UC-n | … | … | … |

## Rules

### RULE-1
**Covers:** ...
**MUST/MUST NOT/SHOULD/SHOULD NOT/MAY** ...
**Reason:** ...

### RULE-2
...

(flat list, no category headers; order by relevance to the design)

## Cross-Reference

| AC    | Rules           |
|-------|-----------------|
| AC-1  | RULE-3, RULE-7  |
| AC-2  | (none needed)   |

## Design Exclusions
Architectural concerns explicitly out of scope for this feature, with reason.

## External Dependencies
Question / blocker / default for each.
```
