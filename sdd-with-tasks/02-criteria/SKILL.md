---
name: criteria
description: Translate the spec into a complete, traceable set of acceptance criteria in EARS form
---

# Acceptance Criteria Skill

Translate the resolved spec into acceptance criteria using EARS templates.

Pipeline position: proposal → spec → **criteria** → rules → review → tasks → review (plan) → execute ⇄ converge

# Role

You translate the spec into independently testable, implementation-agnostic acceptance criteria. You do not invent
product decisions. If a behavior is too ambiguous for a testable criterion, route the gap back to the spec step.

# Inputs

- Resolved spec: @file:spec/spec.md (primary source; "Behaviors to verify" is the contract; "State model",
  "Use cases", "Normative data", "Presentation and navigation" and "Verification expectations" feed the patterns below)
- Proposal: @file:spec/proposal.md (reference)

Spec takes precedence over proposal where they disagree. If the spec references a table or definition by pointer
("matching the tables in …") instead of containing it, that is a route-back: criteria cannot paste what the spec did
not copy. Likewise if a lifecycle exists but no use case covers it: criteria cannot tag paths the spec never wrote.

# EARS Templates

Use the smallest template that fits the requirement. Do not invent new templates.

**Ubiquitous** (always-true behavior):
`The <system> shall <response>.`

**Event-driven** (triggered by an event):
`When <trigger>, the <system> shall <response>.`

**State-driven** (active during a state):
`While <state>, the <system> shall <response>.`

**Optional feature** (applies when a feature is included):
`Where <feature is included>, the <system> shall <response>.`

**Unwanted behavior** (explicit handling of an undesired trigger):
`If <trigger>, then the <system> shall <response>.`

**Combined** (state plus event):
`While <state>, when <trigger>, the <system> shall <response>.`

Example:

```
If a CSV row is missing a required field, then the importer shall skip the row, log a warning with row number and field name, and continue processing.
```

# Patterns

**Boundary** (any bounded value): write three criteria, one within bounds, one at the boundary, one beyond.

**Error**: use the Unwanted behavior template. Specify what the system shall do (not just what it won't) and atomicity (
all-or-nothing vs partial with reported failures).

**State transition**: one criterion per transition, **plus one refusal criterion per action**. The allowed transitions
alone do not constrain the implementation; the complement does.
`While in state A, when <event> occurs, the <system> shall transition to state B and emit event X.`
`If <action> is attempted while the <entity> is not in <allowed states>, then the <system> shall refuse it and shall
not change the <entity>'s state or produce <side effect>.`
The refusal criterion is what a UI-only guard fails; it must be satisfiable only by the system itself.
A transition's AC and the UC step that performs it must agree on state names; the `Flow:` tag makes the pairing explicit.

**Negative** (authz, side-effect paths): use the Unwanted behavior template and state the prohibited outcome explicitly.
`If <unauthorized trigger>, then the <system> shall respond with 403 and not produce, log, or stage X.` The "and not"
half is what makes it negative.

**Universality of authorization criteria**: negative authz criteria are stated over the *entire application surface*,
not over the feature's own objects. Write `If an owner requests any page or action that displays or modifies another
owner's data — including pre-existing pages — then … and shall not disclose that data`, never `… another owner's
scheduling request`. A test scoped to the feature's URL prefix cannot satisfy the former; that is intended.

**Data exactness**: when a behavior refers to enumerated data (seed rows, accounts, defaults, hours), write one criterion
per table stating `shall contain exactly the following rows` and **paste the rows** into the criterion, plus one
criterion for the absence of extras (`shall contain no other rows / accounts / roles`). For credentials add
`each listed secret shall verify against the stored value`. Never write `matching the tables in <file>`.

**Fidelity** (data produced, then stored or displayed): enumerate the fields and require identity.
`When <producer> yields <structure>, the <system> shall persist and present it such that <field 1>, <field 2>, … are
identical to the produced values.` Pair it with an Unwanted-behavior criterion for values that cannot be mapped
(`If <field> names a value the system does not know, then the <system> shall <route/reject> and shall not substitute a
default`).

**Presentation** (observable UI outcomes, not widgets): `The <system> shall show the signed-in user and a logout action
on every page`, `While signed in as <role>, the <system> shall show only <role>'s navigation entries`, `When <role>
signs in, the <system> shall land on <page>`, `The <system> shall render every new page inside the existing layout`.

**Non-functional** (performance, security, observability, accessibility, compatibility, **test environment**,
**presentation coherence**, **localization**): same templates with thresholds and load context.
`When 100 concurrent requests are sustained for 60s, the <system> shall return p95 < 200ms with zero 5xx.`
`The automated tests shall run against an isolated database and shall not modify the runtime data files.`
`The <system> shall resolve every user-visible string, including messages produced in code and element attributes,
through a message key present in every locale bundle.`

# Anti-patterns

**Bad:** `SHALL store data in PostgreSQL using JDBC` → **Better:** `SHALL persist such that data survives application restart`. Don't prescribe implementation.

**Bad:** `... SHALL load config AND init cache AND connect DB` → **Better:** three separate criteria. Compound criteria aren't independently testable.

**Bad:** `SHALL be fast / intuitive / work correctly` → **Better:** `SHALL return p95 < 200ms`. Subjective adjectives aren't testable.

**Bad:** `WHEN user clicks the blue button on the bottom-right` → **Better:** `WHEN user submits the registration form`. Don't prescribe *widgets or positions* — but **do** state observable UI outcomes (a logout action is available on every page, the menu shows only the role's entries). "Don't prescribe UI" is not "don't mention UI".

**Bad:** `SHALL seed the clinic hours matching the tables in proposal.md` → **Better:** the rows pasted into the criterion with `exactly`. A pointer is not testable without the pointed-to file and lets the implementation drift unnoticed.

**Bad:** one criterion for a whole state machine (`SHALL support states A, B, C`) → **Better:** one transition criterion per allowed transition and one refusal criterion per action. Bundled state criteria are satisfied by an unguarded enum.

**Bad:** `If an owner accesses another owner's request, then SHALL return 403` → **Better:** `… any page or action that displays another owner's data … shall deny it and shall not disclose that data`. A status code alone is satisfied by a page that leaks the data elsewhere.

**Bad:** `Flow: UC-1` (the whole use case) → **Better:** `Flow: UC-1 step 4`. A tag on the whole UC is coverage by bundle; no one can tell which step is untested.

**Bad:** grouping ACs under `As an owner I want …` headings → **Better:** keep the flat list; the `Flow:` tag is the grouping. A story layer re-bundles ACs and strands cross-cutting ones in "misc".

# Granularity

One observable outcome per criterion. If the SHALL clause splits naturally or contains "and" between distinct outcomes, split.

# Traceability

Every criterion has:
- Stable ID: `AC-1`, `AC-2`, ... in document order
- `Covers:` line listing the behavior(s) it addresses, using B-N IDs from the spec
- `Flow:` line placing it on a path, with exactly this grammar: `UC-n step k` · `UC-n ext ka` · `cross-cutting`. One AC
  may list several (`UC-1 step 4, UC-3 step 2`). The actor is implicit in the UC; do not add a story layer.

If a criterion covers spec content outside "Behaviors to verify" (e.g., an explicit assumption that doesn't appear as a
B-N), reference by section heading and a distinguishing phrase, e.g.
`Covers: Resolved ambiguities, "currency conversion at order time"`.

# Coverage Rule

Every B-N in the spec must be covered by at least one AC. If a spec item outside "Behaviors to verify" implies runtime
behavior and no AC covers it, flag it as a spec gap. Do not silently invent the behavior.

Coverage is not a count. A B-N that bundles many facts (a seed table, a seven-field structure, a state machine) is
**not** covered by one AC that re-bundles them; apply the Data exactness, Fidelity and State transition patterns so
that each fact is independently assertable. Ask of every AC: "which single wrong implementation does this rule out?"
If the answer is "several at once" or "none in particular", split or sharpen it.

Every table under the spec's "Normative data" has its exactness ACs; every state table under "State model" has its
transition and refusal ACs; every item under "Presentation and navigation" and "Verification expectations" has an AC
or a listed exclusion.

**Paths are covered per step.** Every main-scenario step and every extension of every UC has ≥ 1 AC whose `Flow:`
names it. Extensions are usually Unwanted-behaviour or Combined ACs (`While <state>, when <condition at step k>, the
<system> shall <response> and shall not <error / silent fallback>`). Cross-cutting ACs (authz, localization, data
exactness, boundaries) are tagged `cross-cutting` and are never forced under a UC.

# Route-Back Threshold

Route back to the spec step when any of:

- At least one behavior cannot be phrased as a testable trigger/response without inventing product decisions
- At least two behaviors share the same gap (suggests systemic ambiguity)
- A boundary value, error path, or state transition is implied but not specified
- The spec references normative data by pointer, describes a lifecycle in prose without a state table, or applies a
  loaded word (full, complete, coherent, all) to a screen or data set without defining it
- A use-case extension has no B-N, a lifecycle entity has no use case, or a UC step names a widget or endpoint

A single missing detail you can resolve by reading the codebase is not a route-back; resolve it and note the source.
Don't pretend to understand to avoid the loop.

# Success Criteria

Complete only when ALL hold:

- Every B-N is covered by at least one AC
- Every bounded value has the three-criterion boundary pattern
- Every error path uses the Unwanted behavior template with behavior, atomicity, and observable response
- Every state transition has a transition AC, and every state-machine action has its refusal AC
- Every authorization-relevant path has a negative AC stated over the whole application surface, with the "and not"
  half naming what is not disclosed and not mutated
- Every normative table has exactness ACs with the rows pasted in and an absence-of-extras AC; no AC references a
  table, decision or definition by pointer to another file
- Every produced-then-stored/displayed structure has a fidelity AC enumerating its fields and an unmappable-value AC
- Non-functional categories considered, including test environment, presentation coherence and localization (covered
  or recorded under "Coverage exclusions")
- Every AC uses an EARS template, has one SHALL outcome, avoids implementation prescription, and contains no rationale
- Every AC has an ID and `Covers:` reference
- Every AC has a `Flow:` line; every UC step and extension has ≥ 1 AC naming it; cross-cutting ACs are tagged as such

Run a verification pass before writing. Do not write a partial file.

# Output

Write to `spec/criteria.md`:

```
# Acceptance Criteria: <Feature>

## Functional

### AC-1: <title>
**Covers:** B-N[, B-M ...]
**Flow:** UC-1 step 4

<EARS statement>

### AC-2: ...

## Non-functional

### AC-N: ...

## Coverage exclusions

- <non-functional category>: <reason for not covering>
```
