---
name: spec
description: Turn a feature proposal into a complete, declarative use-case specification in spec/spec.md, including use-case relationships, scenarios, guarantees, and observable outcomes. Use before technical rules or implementation.
---

# Use-Case Specification Skill

Create the behavioral contract for a feature as a set of related use cases. The use cases in `spec/spec.md` are the
units that `execute` implements and `converge` verifies. Do not create a separate behavior list, acceptance-criteria
file, task plan, artifact map, or implementation checklist.

Pipeline position: proposal -> **spec** -> rules -> execute <-> converge (once per use case)

## Role

Clarify what actors need the system to accomplish. Keep the result declarative and implementation-agnostic. A use case
states actor intent, observable system responses, alternative paths, guarantees, and postconditions. It does not name
classes, packages, source files, frameworks, database tables, controller methods, or implementation phases.

## Input

- Feature request: `spec/proposal.md`
- Existing application behavior and product vocabulary from the codebase

If `spec/proposal.md` does not exist, ask for the proposal and save it there.

## Grounding and Interview

Read enough of the existing application to avoid asking questions it already answers and to find affected behavior the
proposal omits. Inspect existing user journeys, routes and pages, roles and access rules, domain lifecycles, normative
data, tests as evidence of intent, and user-visible conventions.

Interview one question at a time. Use concrete, mutually exclusive options, recommend one, and explain briefly why the
answer changes the specification. Ask only when the answer changes behavior, scope, an unsafe assumption, a use-case
relationship, or an observable outcome. Resolve smaller choices yourself and record the decision with its rationale.

Before writing, explicitly resolve any applicable blind spots:

- Authentication or new roles: access to every existing user-facing route, landing behavior, identity display, and
  logout.
- A lifecycle: every state, allowed transition, refused transition, and failure side effect.
- A multi-step journey: the ordered main scenario and what happens when each step cannot complete.
- Seed, reference, or configuration data: whether it is exact or illustrative and whether extra values are allowed.
- New screens: layout and navigation behavior, per-role visibility, and localization.
- Data produced then stored or shown: field fidelity and unmappable values.
- Loaded words such as *full*, *complete*, *coherent*, *all*, or *minimal*: the concrete observable meaning.
- Verification expectations: isolation from runtime data, the boundary that must be exercised, and any required human
  walkthrough.

A finished spec has no open product or behavioral questions. If a decision genuinely depends on an unavailable
stakeholder, record it under `External dependencies` with the blocker and the explicit default in force.

## Use-Case Model

Write one use case per actor goal. Do not split by screen, endpoint, class, or anticipated code change. A use case is
large enough to deliver a meaningful actor outcome and small enough to implement and converge as one vertical slice.
There is no target count.

Use stable IDs in document order: `UC-1`, `UC-2`, and so on. Mark exactly one use case `(primary)` unless the proposal
truly contains several independent features and should be split.

### Relationship semantics

Use only these relations:

- `Requires UC-n`: this use case can start only after the required use case's successful postcondition exists. This is
  an execution dependency.
- `Includes UC-n at step k`: the included actor goal is always performed at that step. Includes also implies
  `Requires UC-n` for execution ordering.
- `Extends UC-n at extension point ka`: this use case conditionally adds an independently meaningful actor goal to the
  base use case. The extending use case is implemented after the base use case.

Keep ordinary error and alternative paths as extensions inside their use case. Promote a branch to a separate use case
only when it is an independently meaningful actor goal. Every referenced ID must exist. `Requires` and implied
execution dependencies must be acyclic. Do not add sequence relations merely to prescribe an implementation order.

Include a `Use-case map` table near the start of the document with each ID, actor goal, primary actor, and relations.
The map summarizes the behavioral graph; it is not a task plan.

### Use-case contract

Use this format:

```markdown
## UC-1 - <actor goal> (primary)

- Goal: <outcome the actor wants>
- Primary actor: <role or external system>
- Supporting actors: <roles/systems, or none>
- Trigger: <observable event that starts the use case>
- Preconditions: <facts that must already hold>
- Relations:
  - Requires: <UC ids with reason, or none>
  - Includes: <UC id at step k, or none>
  - Extends: <UC id at extension point ka, or none>

### Main success scenario

1. <Actor intention or external event.>
2. <Observable system response.>
3. <Actor intention.>
4. <Observable system response.>

### Extensions

- 2a. If <condition>, the system <observable response>; resume at step 1 | continue with UC-n | end.
- 3a. If <condition>, <observable response>; end.

### Guarantees

- G1. <Invariant, access boundary, fidelity rule, presentation outcome, or quality condition that holds across this UC.>
- G2. <For a refused action, what is not disclosed, persisted, emitted, or otherwise changed.>

### Postconditions

- Success: <observable state and what the actor can now do or see>.
- Minimal guarantee: <what remains true when the use case cannot complete>.
```

Scenario steps stay at intention level: `the owner requests an appointment`, not `the owner clicks Submit` or
`POST /requests`. Every system step must be observable at the boundary named by the use case. Every extension must end,
resume at a numbered step, or continue with another use case.

`Guarantees` carry behavior that applies across multiple paths of the use case and would be awkward to repeat in every
step. They are part of the use case, not a separate criteria layer. Refer to them as `UC-1 G1`; refer to scenario
elements as `UC-1 main step 2`, `UC-1 extension 2a`, and `UC-1 success postcondition`.

## Completeness Rules

- Every requested runtime behavior belongs to a scenario step, extension, guarantee, or postcondition of at least one
  use case. Do not create a detached `Behaviors to verify` section.
- Every alternative or failure that changes the observable outcome is an extension.
- Every guarded action states both the response and the absence of prohibited disclosure or mutation.
- Every lifecycle transition appears in a scenario or extension. The state model lists all allowed transitions and
  states that all others are refused without side effects.
- Every produced-then-stored-or-shown structure has a guarantee naming the fields preserved and the handling of values
  that cannot be mapped.
- Presentation and navigation outcomes belong to the use cases in which actors encounter them.
- Normative tables are copied into `spec.md` with `exactly these rows - no more, no fewer`; use cases reference the
  local table by heading when they consume it.
- Non-functional behavior is concrete and attached as a guarantee to every use case it affects. Do not write vague
  claims such as `fast`, `secure`, or `accessible`.
- Out-of-scope items do not appear as scenario steps or implied prerequisites.

## Completion Gate

Before writing, verify all of the following:

- All material ambiguities and applicable blind spots have explicit decisions.
- The use-case map and detailed definitions contain the same IDs and relations.
- Exactly one use case is primary; all relationship targets exist; dependency relations are acyclic.
- Each use case has a trigger, preconditions, main scenario, extensions or an explicit `none`, guarantees,
  success postcondition, and minimal guarantee.
- All scenarios are declarative, observable, and free of implementation design.
- All extensions are closed and all lifecycle transitions are covered.
- Access failures name what is denied and what remains undisclosed and unchanged.
- Normative data is self-contained and exact.
- Every product behavior is inside a use case; there are no B-N or AC-N identifiers and no generated task plan.
- No question remains that the rules or execute stages would have to answer as a product decision.

Do not write a partial specification.

## Output

Write `spec/spec.md` with:

1. Feature summary
2. Scope and resolved decisions
3. Actors and domain terms
4. Use-case map
5. State models, when applicable
6. Detailed use cases, using the contract above
7. Normative data, when applicable
8. Out of scope
9. External dependencies, when applicable
