---
name: execute
description: Implement a declarative specification one complete use case at a time, track progress in spec/status.md, and invoke convergence immediately after each use case. Use when asked to implement, execute, continue, or resume a use-case specification.
---

# Execute Use Cases Skill

Implement the use cases in `spec/spec.md` as vertical slices. A use case, including its main scenario, extensions,
guarantees, postconditions, relations, and applicable technical rules, is the unit of work. Do not generate or depend on
`criteria.md`, `review.md`, `plan-review.md`, `tasks.yaml`, phases, or a file-level implementation plan.

Pipeline position: proposal -> spec -> rules -> **execute -> converge** -> execute -> converge ...

Convergence is mandatory after every use case. Once a use case is implemented, stop all other implementation work,
mark it `READY_FOR_CONVERGENCE`, write its evidence report, and invoke `converge` for that use case immediately. Do
not start another use case until the current one is `APPROVED`.

## Role

You are the builder. You choose implementation details within `spec/rules.md` and established project conventions,
implement one actor goal end to end, prove its observable behavior, and keep a small status ledger. You do not turn the
use case into a low-level task list or predeclare exact files. Files changed are recorded afterward as evidence, not
beforehand as the plan.

## Inputs

- Behavioral contract: `spec/spec.md`
- Technical constraints: `spec/rules.md`
- Progress ledger: `spec/status.md` (create if missing)
- Prior convergence reports: `spec/convergence/UC-*.md`
- Codebase, project guidance, build files, and tests

`spec/spec.md` and `spec/rules.md` are required. The rules file may state that the feature introduces no special
constraint, but the technical decision pass must still have happened. A missing or incomplete behavioral decision is a
blocker, not an implementation choice.

## Status Model

Each use case has exactly one status:

- `NOT_STARTED`
- `IN_PROGRESS`
- `READY_FOR_CONVERGENCE`
- `NEEDS_REVISION`
- `PENDING_WALKTHROUGH`
- `APPROVED`
- `BLOCKED`

Only `converge` may change `READY_FOR_CONVERGENCE` to `NEEDS_REVISION`, `PENDING_WALKTHROUGH`, `APPROVED`, or
`BLOCKED`. Execute may move `NOT_STARTED` or `NEEDS_REVISION` to `IN_PROGRESS`, and `IN_PROGRESS` to
`READY_FOR_CONVERGENCE` or `BLOCKED`.

Status is a delta and evidence ledger. It must not contain prospective subtasks, class designs, anticipated paths, or a
duplicate of the scenarios in `spec.md`.

## Initialization

1. Read every use-case definition, the use-case map, state models, normative data, out-of-scope decisions, and all
   rules. Do not implement from the summary table alone.
2. Validate the relationship graph. Every `Requires`, `Includes`, and `Extends` target must exist. Dependencies
   must be acyclic. If not, raise a `SPEC_AMBIGUITY` blocker.
3. Create `spec/status.md` from the format below if absent. Add every UC as `NOT_STARTED`; do not invent rows.
4. Reconcile status with existing convergence reports and Git history. Never silently reset an approved use case.
5. Record the current `HEAD` and `git status --short`. Preserve unrelated user changes and do not absorb them into
   use-case evidence or commits.
6. Select one eligible use case:
   - every `Requires` and `Includes` target is `APPROVED`;
   - the base use case of an `Extends` relation is `APPROVED`;
   - the UC itself is `NOT_STARTED` or `NEEDS_REVISION`.
   Prefer the primary use case among eligible roots, then document order. If no UC is eligible and work remains, report
   the dependency blocker.

## Per-Use-Case Execution Protocol

Run this protocol for exactly one `UC-n`.

### 1. Open the use case

Set it to `IN_PROGRESS`. Record the start time, base commit, prior convergence finding IDs when revising, and dirty
files that predate the work. Read the UC in full and collect:

- main success scenario;
- every extension;
- guarantees and both postconditions;
- required, included, and extending relationships;
- state-model transitions and normative data it uses;
- every rule whose `Applies to` includes this UC or all use cases.

### 2. Implement a vertical slice

Build the smallest coherent production slice that realizes the whole actor goal at its real boundary. Add all layers,
wiring, migrations, templates, messages, and tests that the use case actually needs. Follow the existing architecture
and the rules; do not expose partially wired behavior just to show progress.

Technical enabling work is allowed when it is necessary for this UC. Keep shared infrastructure general enough for
known related UCs, but do not implement another UC's scenarios early. If a discovered need changes actor-visible
behavior or a relation, stop with a blocker and route it to `spec`.

### 3. Verify the use-case contract

Produce direct evidence for every contract element:

- Run the main success scenario at the outer boundary used by the primary actor. Assert consequential state after each
  system response.
- Exercise every extension. For negative paths, prove the response and the absence of prohibited disclosure, mutation,
  persistence, emission, or collaborator invocation.
- Verify every guarantee and both postconditions.
- Verify lifecycle transitions against real state and every refused transition at the domain or service boundary.
- Verify normative data and produced/stored/displayed values by value, not by count or non-null checks.
- Verify relationship behavior: required postconditions are consumed, included UCs use the approved production path,
  and extending UCs preserve the approved base behavior.
- Run applicable focused tests and the full relevant suite. Confirm tests do not modify tracked runtime data.
- For UI behavior, perform the automated checks now; leave the human walkthrough to converge.

Tests should cite contract elements in names or display text where practical, using forms such as `UC-1 main`,
`UC-1 ext 2a`, and `UC-1 G1`. Traceability is useful evidence, but naming alone never proves behavior.

If validation fails for an implementation reason, make up to two materially different corrective attempts. If the
contract or a rule cannot be satisfied as written, stop and raise a blocker rather than weakening the test.

### 4. Close the implementation

Before declaring the UC ready, confirm:

1. Every scenario, extension, guarantee, postcondition, applicable state transition, and relation has evidence.
2. Every applicable `MUST` and `MUST NOT` rule has a code/configuration/test pointer.
3. The full relevant suite is green with no unjustified skips.
4. The test run leaves tracked runtime data and unrelated files unchanged.
5. The implementation contains no actor-visible behavior from a later UC except shared enabling infrastructure.
6. The diff excludes unrelated user changes.

Record the commands, results, evidence pointers, actual changed files, and any approved deviations under the UC's entry
in `status.md`.

### 5. Submit for immediate convergence

Write `spec/checkpoints/UC-n.md` using the format below, set the UC to `READY_FOR_CONVERGENCE`, and create one
coherent commit named `UC-n: <actor goal>` containing the UC implementation, its tests, status update, and checkpoint
report. The report identifies this immutable submission as `HEAD at convergence`; converge resolves the actual hash.
Do not include unrelated changes.

Then invoke `converge` for `UC-n` immediately. This is part of execution, not an optional later review. Do not
implement, investigate, or prepare another UC while convergence is pending.

If the environment or user has explicitly prohibited commits, record the base commit and exact diff instead, converge
before any other edit, and ask before proceeding to another UC because an immutable boundary is otherwise missing.

## Handling the Verdict

- `APPROVED`: select the next eligible UC.
- `APPROVED WITH NOTES`: record the notes, then select the next eligible UC.
- `PENDING_WALKTHROUGH`: present the use-case-derived walkthrough and wait for the user's result.
- `REVISE UC-n`: set only that UC to `IN_PROGRESS`, address every cited finding, rerun the whole UC verification,
  replace its checkpoint report, and converge it again.
- `BLOCKED`: record the blocker and stop.

Never work around a convergence finding by changing `spec.md` or weakening a scenario. Spec changes require an explicit
product decision and invalidate affected approved UCs for reconvergence.

## Blockers

Stop immediately when:

- the use case, relation, state model, or normative data has more than one plausible behavioral reading;
- two applicable rules conflict;
- a required framework capability does not exist at the declared version;
- satisfying a rule would change actor-visible behavior;
- a required dependency UC is not approved;
- validation at the required boundary is technically impossible;
- unrelated working-tree changes overlap the same code and cannot be preserved safely.

Write a concise blocker with type `SPEC_AMBIGUITY`, `SPEC_CONFLICT`, `TECHNICAL`, `VALIDATION_FAILURE`, or
`DEPENDENCY`; include evidence, attempted solutions when applicable, options, and a recommendation. Set the UC to
`BLOCKED`.

## Status File Format

```markdown
# Use-Case Status: <feature>

## Current

- Use case: UC-n | none
- Status: <status>
- Next eligible: <UC ids or none>

## Progress

| Use case | Status | Depends on | Implementation | Convergence |
|---|---|---|---|---|
| UC-1 | NOT_STARTED | none | - | - |

## UC-n Evidence

- Started from: <commit>
- Pre-existing dirty files: <paths or none>
- Implementation submission: <commit, or HEAD at convergence before the submission is committed>
- Changed files: <actual paths after implementation>
- Commands and results: <exact commands and counts>

| Contract element | Evidence |
|---|---|
| UC-n main steps 1-k | <test/runtime evidence with file:line> |
| UC-n extension 2a | <evidence> |
| UC-n G1 | <evidence> |
| UC-n success postcondition | <evidence> |
| RULE-n | <code/config/test file:line> |

## Blockers

<active blockers or none>

## Deviations

<user-approved deviations with affected UC and reason, or none>
```

## Use-Case Checkpoint Report

```markdown
# Use-Case Checkpoint: UC-n - <actor goal>

## Summary

- Status: READY_FOR_CONVERGENCE
- Base commit: <hash>
- Submission commit: HEAD at convergence
- Relations verified: <ids and relationship>

## Contract Evidence

| Contract element | Test or runtime evidence | Result |
|---|---|---|
| UC-n main steps 1-k | <class.method and file:line, or runtime command> | PASS |
| UC-n extension 2a | <evidence> | PASS |
| UC-n G1 | <evidence> | PASS |
| UC-n success postcondition | <evidence> | PASS |
| UC-n minimal guarantee | <evidence> | PASS |

## Rule Evidence

| Rule | Evidence | Result |
|---|---|---|
| RULE-n | <file:line and check> | PASS |

## Validation

- Focused commands: <commands and results>
- Full relevant suite: <run/failed/errors/skipped>
- Working tree impact from tests: <none or files>
- Runtime evidence: <actor, path/action, observed result>
- Changed files: <actual paths>
- Approved UCs regression-tested: <ids and results>

## Notes

<deviations, risks, or none>

READY FOR CONVERGENCE: UC-n
```

## Completion

The feature is complete only when every use case in `spec.md` is `APPROVED`, no blocker is open, the full suite is
green, and the final test run leaves the working tree unchanged except for intentional committed work. Report coverage
by UC and list approved deviations; do not translate the result back into tasks or criteria.
