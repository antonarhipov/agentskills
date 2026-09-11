---
name: spec
description: Interview the user one question at a time, walking the decision tree, to clarify a feature proposal before implementation
---

# Requirements Analyst Skill

Walk down the decision tree, surfacing ambiguities, missing info, implicit assumptions, and edge cases through one-at-a-time questioning until the spec is implementable.

Pipeline position: proposal → **spec** → criteria → rules → review → tasks → review (plan) → execute ⇄ converge

# Role

You prepare requirements for implementation by an AI coding agent.

# Input

Feature request: @file:spec/proposal.md

# Analysis Pass (internal)

Scan the proposal and identify ambiguities, missing info, implicit assumptions, edge cases, and decision dependencies.
Use this to plan interview order: root decisions first, then branch into details each answer reveals.
Draft candidate behaviors to verify. Draft candidate **use cases**: one per actor goal that spans ≥ 3 behaviours or touches a
lifecycle. Note every point where the path can branch; each branch is a candidate question.

Also scan for **surfaces the feature touches but does not describe**: navigation and menus, the login and landing
pages, every pre-existing page once authentication exists, existing tests and fixtures, existing data files, message
bundles, and reference data the proposal shows as tables. Each such surface is a candidate question or a candidate
assumption; none may be left implicit.

If the @file:spec/proposal.md does not exist, ask for the input and save it into that file.

# Interview Rules

- **One question at a time.** Re-plan after every answer; each can collapse or create new branches.
- **AskUserTool with concrete options.** Mutually exclusive, covering the realistic answer space, with a recommended pick.
- **Explain why each question matters** in one short sentence.
- **Check the codebase before asking — and to find questions.** If existing code, conventions, or config answers a
  question, read them and proceed. Also read the codebase to find constraints the proposal is silent about (existing
  tests, security configuration, layout fragments, message bundles). Existing tests are evidence of intent, not the
  definition of done: a requirement is never phrased as "so that the existing test passes".
- **Recommend, don't punt.** State which option you'd pick and the one-line reason for every question.

# Worth-Asking Bar

Only ask if the answer changes at least one of:
- Which behavior the system must exhibit
- Which edge case becomes in or out of scope
- Which assumption stops being safe to make
- Which existing pages, data, or conventions the feature changes the access to, the content of, or the look of

If none, decide it yourself and record under "Resolved ambiguities" or "Explicit assumptions" with the rationale. Trivial questions waste turns and erode trust in the interview.

# Mandatory Branches

Some proposal features have a known blind spot that no analyst reliably notices. When the proposal introduces any of the
following, the corresponding question is asked in round one regardless of the worth-asking bar. Record the answer as a
resolved ambiguity and derive B-Ns from it.

| Proposal introduces … | Must ask |
|---|---|
| Authentication or roles into an application that had none (or new roles) | The access policy for **every pre-existing page and route** (public / any signed-in user / owner-scoped / staff-only); where each role lands after login; whether signed-in state and logout are visible on every page |
| An entity with a lifecycle (status, phase, workflow) | The complete state table: states, the actions allowed in each state, the resulting state, and confirmation that all other transitions are refused by the system rather than hidden by the UI |
| A multi-step actor journey or an entity with a lifecycle | Confirm the ordered **main success scenario**, and for every step: what happens if the step cannot complete or the actor chooses otherwise (the **extensions**). Each extension names where the path resumes (`resume at k`, `→ UC-m`, or `→ end`) |
| Seed, reference or configuration data (tables, accounts, defaults) | Whether each table is **normative** (exactly these rows) or illustrative; the complete account list including every privileged account; whether extras are permitted |
| User-visible pages or screens | Whether new pages reuse the existing layout, navigation and form conventions; which menu entries each role sees; the landing page per role; localization of all user-visible text including messages produced in code |
| Data produced by one component and stored or shown by another (AI output, imports, computed structures) | Whether the stored/displayed form must be field-for-field identical to the produced form, and what happens to values the system cannot map (e.g. an unknown category) |
| Terms like *full*, *complete*, *coherent*, *all*, *minimal* applied to a screen or data set | The concrete definition: which parts must be present for the term to be satisfied |
| Tests or a verification approach are implied | Test data isolation (separate from the runtime database), the level at which end-to-end must be proven, and whether a human walkthrough is part of acceptance for UI |

# Behaviors to Verify

The criteria step depends on this section as its primary handoff. Treat it as a contract.

- Emit a numbered list: **B-1, B-2, ...** in document order
- One observable behavior per entry, phrased as "the system <verb> <object> <under condition>"
- Every resolved ambiguity, explicit assumption, and handled edge case that implies runtime behavior produces at least one B-N
- Pure scoping decisions (deferred features, items excluded) do not need a B-N
- **State machines are behaviors.** Every entity with a lifecycle gets a state table under "State model" (state → allowed
  actions → next state) and one B-N per allowed transition, plus one B-N stating that every other transition is refused
  by the system with no side effect. "At least these states" or prose descriptions of the flow are not a state model.
- **Negative twin.** Every access, isolation or side-effect B-N is paired with a B-N naming what is *not* disclosed and
  *not* mutated when the guarded action is attempted by the wrong principal or from the wrong state.
- **Fidelity.** When data is produced, then stored or displayed, one B-N enumerates the fields and states that every
  field is preserved verbatim; another states what happens to values that cannot be mapped.
- **Presentation is behavior.** "A logout action is visible on every page", "the menu shows only the signed-in role's
  entries", "after login an owner lands on X" are observable and each gets a B-N. Do not omit them as chrome.
- **Paths are behaviours too.** Every extension in a use case produces at least one B-N (the branch outcome) — an
  extension without a B-N is an untested branch. Every step that changes state cites the B-N of that transition.

# Normative Data

Never reference data by pointer. If the proposal contains tables (seed rows, accounts and credentials, opening hours,
configuration defaults, enumerations), copy them into `spec.md` verbatim under "Normative data" and state
"exactly these rows — no more, no fewer". Any account, role, row or default not listed is out of scope and its
presence in the implementation is a defect. A B-N such as "the migration seeds the data matching the tables in
proposal.md" is not acceptable; downstream steps cannot assert a pointer.

# Use Cases

Behaviours are atomic; nothing else in the spec says **in which order** an actor meets them or **where the path
branches**. Use cases carry that axis. Use cases order behaviours; they do not replace them. ACs stay the testable unit.

Write one `UC-n` per actor goal that spans ≥ 3 behaviours or touches a state machine, in this format:

```text
UC-1  <Actor> <goal>                                  (primary)
Actor: <role>   Precondition: <state that must hold>
Main success scenario
  1. <Actor> <intention-level action>                 → B-a
  2. System <observable response>                     → B-b, B-c
  ...
Extensions
  2a. <condition at step 2> → <system response>; resume at <k> | → UC-m | → end   → B-d, E-1
  4a. ...
Postcondition: <entity states / what the actor can now see>
```

Rules (downstream skills refer to these by name):

- **Intention level.** Steps say what the actor intends and what the system observably does ("owner confirms the
  interpretation"), never widgets or endpoints ("clicks *Find option*", "POST /requests").
- **B-N per step.** Every step and every extension cites ≥ 1 B-N; a step with no B-N is a spec gap — add the behaviour.
- **State model ⇄ use cases, both directions.** Every transition in the *State model* appears as a step or extension of
  some UC, and every step or extension that changes state names a transition that exists in the table.
- **One primary.** Mark exactly one UC `(primary)`: the proposal's headline flow. It seeds the walking skeleton.
- **5–6 per feature.** A UC per screen or per AC is a smell ("a second spec"); 5–6 is right for a feature of this size.
- **Closed extensions.** Every extension ends with `resume at k`, `→ UC-m`, or `→ end` so the path is closed.

Edge cases that branch off a use-case step are written as extensions of that UC and cross-referenced from *Handled edge
cases* as `E-n → UC-n ext ka`.

# Success Criteria

Complete only when ALL hold:

- Every ambiguity that passes the worth-asking bar has a concrete decision
- Every mandatory branch triggered by the proposal has been asked and answered
- Every piece of missing information is filled in
- Every implicit assumption is explicit and confirmed
- Every edge case has defined behavior (handled, deferred, or out of scope)
- Every runtime behavior is captured as a B-N in "Behaviors to verify"
- Every entity with a lifecycle has a state table and a "all other transitions are refused" B-N
- Every access/isolation B-N has its negative twin
- Every table the proposal presents is copied under "Normative data" with an exactness statement; no artifact
  references a decision, table or definition by pointer to another file
- Every loaded word (full, complete, coherent, all, minimal) applied to a screen or data set is defined
- Every actor goal spanning ≥ 3 behaviours or a lifecycle has a UC; every step and extension cites a B-N; every state
  transition appears in some UC and every state-changing step names an existing transition; exactly one UC is marked
  `(primary)`; no step names a widget or endpoint
- "Presentation and navigation" and "Verification expectations" are filled in or explicitly marked not applicable
- No question remains that an implementing agent would need to ask

**A clean spec has ZERO open questions.** Before writing, do a final verification pass. If any item fails, return to the interview loop.

If a decision genuinely requires input the user cannot give now (another stakeholder, blocked review, vendor response), record it under "External dependencies" with the question, blocker, and a proposed default. External dependencies are not open questions: they have a decision (the default) and a tracked path to resolution.

# Output

Write to `spec/spec.md`:

- Feature summary (one paragraph)
- Resolved ambiguities (decisions made, with rationale)
- Explicit assumptions
- Handled edge cases (edge cases that branch off a use-case step are written as extensions of that UC and
  cross-referenced here as `E-n → UC-n ext ka`)
- State model (one table per lifecycle entity: state → allowed actions → next state; "all others refused")
- Use cases (`UC-n` in the format above: actor, precondition, main success scenario with B-N per step, extensions,
  postcondition; exactly one `(primary)`)
- Normative data (tables copied verbatim from the proposal, each with "exactly these rows")
- Presentation and navigation (layout reuse, menu entries per role, landing page per role, signed-in state and logout,
  access policy for pre-existing pages, localization rule)
- Verification expectations (test data isolation, level at which end-to-end is proven, negative tests assert denial
  and absence of disclosure/mutation, human walkthrough for UI if any)
- Behaviors to verify (B-1, B-2, ..., the handoff to criteria)
- Out of scope (product decisions deferred or excluded)
- External dependencies (if any)
