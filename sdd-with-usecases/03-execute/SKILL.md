---
name: execute
description: Implement use-case specifications in an autonomous loop, record technical evidence, run a real Playwright walkthrough, fix failures, and approve each passing use case before continuing.
---

# Execute Use Cases

Implement the user-authorized use cases in one feature directory under `spec/`. Read its complete `spec.md`, existing
`status.md`, applicable project guidance and optional `rules.md`. Preserve unrelated working-tree changes. Do not
introduce task plans, criteria files or new product decisions in place of the specification.

The normal loop is:

**implement -> relevant technical checks -> recorded Playwright walkthrough -> fix and repeat when needed -> approve -> next UC**

A separate convergence review is optional unless the user explicitly requests it or an applicable product contract
requires it. A passing real-browser walkthrough and its durable ledger own the UI approval gate. Do not add a second
human confirmation, mandatory independent audit, or procedural pause after the automated walkthrough passes.

## Scope and ordering

Resolve one feature directory and read the detailed use cases, lifecycle, normative data and exclusions. Reconcile
existing approvals against committed evidence; do not silently reset an approved UC. Record the base commit and
pre-existing dirty files. Validate that relationship targets exist and explicit dependencies are acyclic.

Select the next eligible UC, normally in document order. Explicit `Requires`, `Includes` and `Extends` relationships
remain dependencies: consume their actual production postconditions. A reference such as “continue with UC-n,” an
editor link, or an unavailable-setting explanation is a handoff to that destination goal, not an implicit dependency
that prevents the current UC from being implemented or approved. Check the current UC's explanation, handoff and
absence of prohibited mutation; implement and verify the destination journey when its UC runs. Record outstanding
handoffs in the ledger and close them when the destination is implemented. Never describe a destination as complete
merely because its entry/guidance exists.

Resolve routine sequencing and necessary shared or prerequisite work autonomously within the authorized scope.
Keep evidence attributable to each UC; enabling work does not itself approve another UC. Do not stop solely because
a future continuation is not yet implemented, a separate convergence report is missing, or the old procedural order
would prevent completing a dependency. Ask only for a missing product decision or authority that is actually needed.

## Implementation and technical checks

Set the current UC to `IN_PROGRESS` and record its start/base, applicable constraints and existing findings. Implement
its actor goal at the production boundary, including applicable main steps, extensions, guarantees, postconditions,
state transitions and normative fields. Preserve exact source/Current values, optional presence, identity and
provenance where required. Use explicit atomic saves and existing access/state/version guards.

Choose routine architecture and implementation details from the project and contract. Record material constraints
in the status evidence; use the rules skill only for consequential new constraints or deviations. Do not weaken the
specification, tests or walkthrough to fit the implementation.

Run checks appropriate to the change and every affected approved UC. Consolidate slow runtime checks at the completed
slice; reuse unchanged valid evidence rather than repeatedly running broad unrelated suites. Assert produced and
stored values by value, not only counts/status/non-null. Negative paths must prove their response and absence of
prohibited mutation. For production-process failures, verify the durable outcome through the real actor boundary.
Record commands, results/counts, actual changed files and technical evidence. Tests must leave operator data and
unrelated tracked files untouched.

Keep a reviewable checkpoint in `<feature-dir>/checkpoints/UC-n.md`: base/submission identity, implemented contract
coverage, test commands/results, changed files, source/Current fidelity, outstanding destination handoffs and known
limitations. Commit only intended work when commits are authorized and available. If committing is prohibited or
unavailable, identify the base and exact diff; this alone does not require another permission request or stop the loop.

## Playwright walkthrough and ledger

For a UI goal, set `PENDING_WALKTHROUGH` and immediately read/use the available Playwright skill against the actual
running application. Existing automated browser tests support this walkthrough but do not replace its recorded
administrator journey. Use isolated disposable school/workspace data, the real navigation and rendered controls.

Derive the journey from the current UC: main success scenario, browser-reachable negative paths, save/cancel/stale
recovery, reload and responsive widths where applicable. Corroborate consequential durable values through read-only
production API or isolated database queries. Required restart/process-fault evidence may come from fresh technical
checks; identify it explicitly rather than claiming to have clicked an infrastructure failure. Treat cross-UC
continuations according to the ordering rule above and keep their outstanding status visible.

Store every attempt under a unique UC-specific `output/playwright/<feature>/UC-n/<run>/` directory. The Playwright
skill must write a durable walkthrough ledger there with:

- feature/UC, submission or exact diff, date, application URL/build, browser and starting state/data isolation;
- each step's contract reference, action, expected result, observed result and PASS/FAIL;
- assertions and exact saved values/revisions/source/Current checks, including refused-operation non-mutation;
- links to screenshots and snapshots where applicable, inspected layout/overflow results, and relevant fault evidence;
- unexpected errors/findings, overall verdict, and browser/server/disposable-data cleanup or retained disposition.

Capture and inspect screenshots for presentation/responsive claims, significant saved/refused states and failures when
applicable. Explain when a step has no useful screenshot instead of manufacturing an image. Preserve failed attempts,
logs and screenshots; do not overwrite them with the later successful run. Link the ledger from feature status and
checkpoint/approval evidence.

## Fix, approve and continue

- **PASS:** when the applicable technical checks and recorded walkthrough outcomes pass, set the current UC to
  `APPROVED`, record/link its passing ledger, and commit the approval/evidence when available. Immediately select and
  implement the next eligible UC within the user-authorized scope. No additional approval or convergence ceremony.
- **FAIL:** set `NEEDS_REVISION`, record every observed finding and retained evidence, then execute the production fix.
  Run affected technical checks and repeat the relevant journey against the changed submission; repeat the full
  current-UC walkthrough before approval. Continue this fix/walkthrough loop until it passes. There is no arbitrary
  two-attempt limit for ordinary reversible implementation fixes.
- **Unavailable boundary or unresolved decision:** never infer PASS. Diagnose and try meaningful alternatives. Set
  `BLOCKED` only when required runtime evidence, authority, product decisions or contradictory product constraints
  genuinely prevent progress. Record the concrete condition and attempts. A separate procedural gate or unfinished
  destination handoff is not itself a blocker.

For non-UI goals, use the real actor boundary and store an equivalent execution ledger; mark approval only from
observed success and applicable technical evidence. Honor an explicit human gate only when the user or product
contract actually retains it. Automated evidence is never labelled human confirmation.

## Status and completion

Use the existing status values (`NOT_STARTED`, `IN_PROGRESS`, `READY_FOR_CONVERGENCE`, `NEEDS_REVISION`,
`PENDING_WALKTHROUGH`, `APPROVED`, `BLOCKED`). Execute may finalize the walkthrough verdict and approval directly.
`READY_FOR_CONVERGENCE` is used only for an explicitly requested separate review, not as a mandatory normal step.

Maintain a concise append-oriented evidence ledger in `<feature-dir>/status.md`: current UC/status, each UC's status,
explicit dependencies, implementation/checkpoint and walkthrough references, dated findings/fixes/approvals and any
outstanding destination handoffs. Approval of one UC never asserts completion of another.

Continue across all authorized UCs without per-UC questions. The feature is complete when every specified UC is
approved, all destination handoffs are implemented and verified, no blocker remains and applicable final regressions
pass without operator-data or unrelated-file changes. Report the actual coverage and limitations; do not claim the
whole feature complete from one passing slice.
