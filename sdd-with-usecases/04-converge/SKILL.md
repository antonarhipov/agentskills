---
name: converge
description: Independently verify one implemented use case, automatically perform its UI walkthrough with Playwright after technical convergence, and record approval or findings before execution may continue. Use whenever a UC is ready for convergence or the user asks to validate an implemented use case.
---

# Converge Use Case Skill

Verify one implemented use case immediately after execution submits it. Grade evidence you obtain yourself, record the
verdict in `<feature-dir>/convergence/UC-n.md`, and update `<feature-dir>/status.md`. Do not wait for a phase, batch several use cases,
or depend on criteria, review, plan, task, or artifact-map files.

Pipeline position: proposal -> spec -> [rules, optional] -> execute/checkpoint -> **converge UC-n -> Playwright walkthrough/approval** -> next UC ...

## Feature Directory

Resolve the same `<feature-dir>` used by the executor before reading evidence or writing a verdict. Use the directory
explicitly named by the user or recorded in the submitted checkpoint. When the target could belong to more than one
feature, stop and ask which feature is in scope. Never verify against a sibling feature's specification or rules.

## Role

You are the verifier, not the builder. Do not fix production code, tests, migrations, templates, or build files. Start
from the use-case contract, reproduce the claimed behavior, inspect the assertions, and decide whether the UC is
actually complete. A green suite is supporting evidence, not a verdict.

Only `APPROVED` or `APPROVED WITH NOTES` lets execute select another UC. A UI use case also needs a passing
agent-run Playwright walkthrough after technical convergence. This satisfies the walkthrough gate by default;
user confirmation is required only when explicitly requested or required by an applicable contract. Preserve such
existing requirements unless the user authorizes changing them; never claim automated evidence is human confirmation.

## Inputs

- Target use case and all related use cases: `<feature-dir>/spec.md`
- Applicable constraints: project guidance and `<feature-dir>/rules.md`, when present
- Executor ledger: `<feature-dir>/status.md`
- Executor submission: `<feature-dir>/checkpoints/UC-n.md`
- Earlier convergence reports: `<feature-dir>/convergence/UC-*.md`
- Git history and diff, code, tests, configuration, migrations, templates, messages, and runtime data

A missing rules file is not a protocol finding or a blocker. Use the specification, project guidance, and this
skill's verification requirements directly. Read and enforce every applicable existing rule; if a submission cites a
missing constraint source, investigate that discrepancy rather than silently ignoring it.

Read the detailed UC, not only the use-case map. The contract consists of every main-scenario step, extension,
guarantee, success and minimal postcondition, state transition, normative-data dependency, and relationship.

## Per-UC Protocol Gate

Run this gate before grading behavior:

1. Exactly one target UC is named and its status is `READY_FOR_CONVERGENCE`. For a resumed
   `PENDING_WALKTHROUGH`, verify the prior passing technical convergence and unchanged submitted implementation
   instead; resume its missing walkthrough without resetting or re-running unchanged evidence.
2. `<feature-dir>/checkpoints/UC-n.md` exists and is committed with the submitted implementation. If commits were explicitly
   prohibited, it instead names the base commit and exact current diff, and no later work has begun.
3. Every `Requires` and `Includes` dependency is `APPROVED`; the base of an `Extends` relation is `APPROVED`.
4. No other UC is `IN_PROGRESS` or `READY_FOR_CONVERGENCE`.
5. The checkpoint contains evidence rows for the main scenario, every extension and guarantee, both postconditions,
   every applicable rule, relationship behavior, test commands, changed files, and regression results for approved UCs.
6. The implementation diff is attributable to this UC or necessary shared enabling infrastructure. It does not contain
   unrelated user changes or completed behavior belonging only to a later UC.

A missing report, batched UCs, unmet relationship dependency, or missing immutable submission boundary is a
`PROTOCOL` finding and blocks approval. Missing evidence rows do not stop the audit; grade the missing elements
`ABSENT`.

## Grounding

1. Record `git status --short` before running anything and identify pre-existing changes from the checkpoint.
2. Resolve the submitted commit and diff from the checkpoint's base. Inspect every changed file; do not treat the file
   list as a declaration that the change is correct.
3. For a new technical audit, run the focused tests named in the checkpoint and the project's full relevant suite
   yourself. Record run, failure, error, and skipped counts. When only resuming the walkthrough/confirmation for an
   unchanged passing submission, use its recorded audit evidence; do not repeat the suite without a new reason.
4. Record `git status --short` afterward. Any new tracked runtime-data or generated-file change is a finding.
5. Locate every claimed assertion at its cited `file:line` and read enough production code to establish that the test
   drives a real path.
6. When the UC has a runtime boundary, reproduce the main scenario and boundary-relevant extensions through that
   boundary as the named actor. Record observed responses and state, not just process exit.
7. Re-run regression evidence for all approved UCs that share a component, route, state, or relation with the target.

Do not substitute executor-reported output for these checks.

## Evidence Ledger

Create one row for every contract element:

| Contract element | Executor claim | Evidence obtained | Strength | Verified |
|---|---|---|---|---|
| UC-1 main step 2 | ... | file:line / runtime result | STRONG | yes |
| UC-1 extension 2a | ... | ... | ABSENT | no |
| UC-1 G1 | ... | ... | WEAK | no |
| UC-1 success postcondition | ... | ... | STRONG | yes |
| UC-1 Requires UC-2 | ... | ... | MISPLACED | no |

Use these grades:

- `STRONG`: pins the observable outcome at the boundary where the contract lives, including prohibited side effects.
- `WEAK`: can pass for a wrong implementation, such as count-only data checks, non-null assertions, status-only
  security checks, or assertions on a mock's own stub.
- `IMPOSSIBLE`: relies on a double, exception, value, or route that production cannot produce.
- `MISPLACED`: tests below the boundary the actor or rule uses, such as a service call presented as an HTTP journey.
- `ABSENT`: no evidence exercises the contract element.

Only `STRONG` is verified. Every other grade produces a blocking gap.

## Verification Categories

Run every applicable category for the target UC.

### 1. Main scenario and postconditions

Walk the main scenario in order as the primary actor. Confirm every observable system response, consequential state
change, success postcondition, and minimal guarantee. A test that skips a step, calls an internal service in place of
the actor boundary, or checks only the final response is insufficient.

### 2. Extensions and negative behavior

Exercise every extension at its branch point and confirm its declared continuation. Negative paths must prove both the
response and absence of prohibited disclosure, mutation, persistence, emission, or collaborator invocation. Confirm
failures preserve the minimal guarantee. For external-process interruption, timeout, transport, or rejected-output
paths, require production actor-boundary evidence of the durable terminal state; adapter-only coverage is `MISPLACED`.

### 3. Relationships and regression

- `Requires`: prove the UC consumes the required UC's actual approved postcondition.
- `Includes`: prove the scenario invokes the included UC's production path rather than a duplicate approximation.
- `Extends`: prove the extension occurs only at its declared point and the base UC remains green unchanged.

Run all approved related UCs. A regression is a critical finding against the target UC.

### 4. Lifecycle, data, and fidelity

For every relevant lifecycle, derive the action-by-state matrix from code and compare it with `spec.md`. Refusal must
be enforced below the UI with the declared result and no side effect. Query freshly migrated normative data and compare
every row and credential by value. For produced, stored, and rendered structures, compare every named field and verify
unmappable values follow the declared extension.

### 5. Security surface

Enumerate the routes the UC uses plus affected existing routes and compare their access with the specification,
project security guidance, and any applicable rules/access matrix. If no matrix is recorded, derive the route-to-access
mapping during the audit from those sources and actual configuration; an absent rules file is not a security failure.
Reproduce anonymous, wrong-role, and wrong-owner cases where applicable. Verify denial, no disclosure, and no mutation.
An unexplained access mapping, `permitAll` contrary to the declared policy, or test scoped narrower than the UC
guarantee is blocking.

### 6. Presentation and walkthrough

For UI use cases, verify layout, navigation, role visibility, identity/logout behavior, messages, and localization from
code and rendered output. Derive the walkthrough from the UC's main scenario, extensions and presentation guarantees.
Complete all other applicable verification categories first. Once all technical evidence is STRONG and there are no
blocking findings, record and commit the technical convergence
report with `PENDING_WALKTHROUGH`, then run the following workflow immediately in the same task. Do not hand the
browser work back to the user. The final approval follows the walkthrough; a green suite alone is insufficient.

#### Automatic Playwright walkthrough

1. Read and use the available `playwright` skill. Prefer its CLI workflow in a real browser against the submitted
   running application. Provision isolated disposable data and use the feature's actual entry/navigation path as its
   primary actor. Record the submission hash, application/build, URL, browser, data isolation and starting state.
2. Drive the declared main journey and browser-reachable extensions through rendered controls. Refresh snapshots
   before using element refs after navigation or rendering changes. Exercise applicable failed saves, unsaved
   navigation choices, concurrent/stale recovery, and reload through the UI; do not replace clicks with internal
   service calls or direct database writes. Verify protected access/roles when they are part of the contract.
3. Assert the response and consequential durable state at each significant action. Use read-only production API or
   isolated database queries to corroborate exact saved fields, versions, source/Current fidelity and absence of
   prohibited mutation. Refused operations must retain the required working values and saved state. Existing strong
   convergence evidence may support failures needing controlled infrastructure faults; explicitly cite that evidence
   rather than claiming the interactive walkthrough exercised them.
4. Exercise the specified desktop and narrow viewports, inspect screenshots, and check overflow, visibility, labels
   and controls. For editable forms, confirm resizing neither saves nor discards edits and widening restores working
   values where required. Observe saved restoration on reload and restart when the UC requires it; reuse fresh
   unchanged restart evidence from technical convergence rather than repeating expensive checks without cause.
5. Save step-by-step expected/observed PASS or FAIL results, assertion outcomes, screenshots, accessible snapshots and
   durable-state evidence under a UC-specific directory in `output/playwright/`. Link them from the convergence report
   and status entry. Use unique run paths for revisions; preserve prior evidence. Record expected refusal/network
   errors separately from unexpected browser failures. Close agent-owned browser sessions; record the disposition of
   disposable servers/data and keep unrelated operator state untouched.
6. If every required walkthrough outcome passes, finalize approval automatically and commit the final report, status
   and referenced evidence. Do not request another confirmation unless an explicit human gate applies. A behavioral
   defect or missing assertion is a convergence finding: reject and return to execute for a revised checkpoint, then
   reconverge the changed submission and repeat its walkthrough. Unavailable Playwright/runtime access is BLOCKED
   with concrete evidence; never silently skip the walkthrough, infer PASS, or relax a required boundary.

For an explicitly retained human gate, perform all available automation first, record its results, then leave
`PENDING_WALKTHROUGH` with only the exact confirmation still needed. Already received confirmation applies without a
repeat request. For a resumed unchanged submission, finalize from the recorded evidence/confirmation; repeat checks
only for changed code, stale evidence, failures or unresolved concerns.

### 7. Technical rules

For every existing rule applying to the UC or all use cases, quote the modal constraint and point to the code,
configuration, test, or runtime evidence that satisfies it. A rule listed in the checkpoint without evidence is not
satisfied. Also verify applicable project constraints. When no feature rules apply, record that conclusion without
inventing rule IDs; all other applicable verification categories still run.

### 8. Test and repository hygiene

Confirm isolated data, no new unjustified skips, production-conformant doubles, stable time/network behavior, and no
test-generated tracked changes. Check that architecture tests have not gained exemptions. Inspect the diff for later-UC
behavior and unrelated changes.

## Findings and Severity

Each finding includes an ID, title, exact UC/RULE reference quoted from the source, code and test evidence with
`file:line`, why it fails, and a concrete revision outcome.

- `C-n CRITICAL`: observable behavior, state, data, access, relationship, or approved UC is wrong.
- `G-n GAP`: behavior may be right but its evidence is WEAK, IMPOSSIBLE, MISPLACED, or ABSENT.
- `P-n PROTOCOL`: the per-UC protocol gate failed.
- `D-n DRIFT`: implementation differs only where the spec explicitly allowed variation; non-blocking candidate note.
- `K-n COSMETIC`: naming or documentation defect that does not change behavior or proof.

Critical, gap, and protocol findings block approval.

## Verdict and Status Update

- `APPROVE`: zero critical, gap, or protocol findings and a passing recorded Playwright walkthrough for UI UCs
  (plus explicit human confirmation only when required). Set UC to `APPROVED`.
- `APPROVE WITH NOTES`: same, with non-blocking drift/cosmetic notes. Set UC to `APPROVED`.
- `PENDING WALKTHROUGH`: technical convergence passes and the automatic UI walkthrough is next; set
  `PENDING_WALKTHROUGH` and immediately run it. Leave this as a final verdict only when an explicit human
  confirmation gate remains unsatisfied; missing runtime/tool access is BLOCKED.
- `REJECT`: any critical, gap, or protocol finding that implementation can resolve. Set UC to `NEEDS_REVISION` and
  emit `REVISE UC-n: ...`.
- `BLOCKED`: resolving the finding requires a product decision, conflicting rules, or unavailable authority. Set UC
  to `BLOCKED`.

Update only the target row, `Current`, `Next eligible`, and convergence reference in `<feature-dir>/status.md`. Preserve the
executor's evidence. On rejection, list finding IDs under the UC entry.

After writing the report and status update, commit only those convergence artifacts and their referenced walkthrough
evidence as `UC-n: convergence <verdict>`. For UI slices, commit the passing technical report before the walkthrough
and the final verdict/evidence afterward. If commits were explicitly prohibited, leave them as clearly identified verifier
changes and do not let a later implementation submission absorb them.

## Specification Changes

Ordinary convergence does not rewrite `spec.md` or an existing `rules.md` to match the implementation. Routine
implementation choices within the contract and established architecture do not require a new rule or user decision.
When code reveals an unresolved product decision or a consequential deviation from an applicable constraint, report
it as blocked. Change the affected contract only after the user explicitly chooses it. Then mark every approved UC
affected by that change `NEEDS_REVISION` and reconverge it; do not accept as-built drift silently.

## Completion Gate

Before writing the report, verify:

- The protocol gate and grounding steps ran for exactly one UC.
- The ledger has a row for every scenario step, extension, guarantee, postcondition, relationship, and applicable rule.
- Every non-STRONG row has a blocking finding.
- Every applicable verification category records evidence or a finding.
- Related approved UCs were regression-tested.
- The verdict matches the finding counts and walkthrough state.
- `<feature-dir>/status.md` reflects the verdict and names the convergence report.
- The report ends with exactly one response line for execute.

Do not write a partial report.

## Output

Write or replace `<feature-dir>/convergence/UC-n.md`:

```markdown
# Convergence: UC-n - <actor goal>

## Summary

- Submission: <feature-dir>/checkpoints/UC-n.md at <commit or exact diff>
- Verdict: <APPROVE | APPROVE WITH NOTES | PENDING WALKTHROUGH | REJECT | BLOCKED>
- Findings: <counts by severity>
- Suite: <run/failed/errors/skipped>
- Working tree impact from verification: <none or files>

## Protocol Gate

<each item with evidence>

## Runtime Reproduction

| Actor | Step or extension | Executor reported | Converge observed |
|---|---|---|---|

## Evidence Ledger

| Contract element | Executor claim | Evidence obtained | Strength | Verified |
|---|---|---|---|---|

## Rule Conformance

<If no feature rules apply, state that explicitly; omit the table. Project constraints still require verification.>

| Rule | Constraint | Evidence | Result |
|---|---|---|---|

## Related-UC Regression

| Use case | Relationship/shared surface | Evidence | Result |
|---|---|---|---|

## Findings

<grouped by severity; empty groups may be omitted>

## Walkthrough

<UC-derived Playwright steps, expected/observed results, assertion outcomes, screenshots/snapshots and durable-state
evidence links, environment/submission and cleanup; include actual user confirmation only if required/received.
Omit for non-UI use cases.>

## Status Update

<old status -> new status; next eligible UCs>

## Response to execute

<exactly one line: APPROVED | APPROVED WITH NOTES: ... | PENDING WALKTHROUGH: ... | REVISE UC-n: ... | BLOCKED: ...>
```
