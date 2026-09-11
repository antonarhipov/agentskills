---
name: execute
description: Execute the implementation plan in spec/tasks.yaml task by task, validating each against its acceptance criteria and halting at checkpoints. Use this skill whenever the user asks to implement, execute, run, continue, or work on a plan; when a spec/ directory with tasks.yaml exists in the repo; when they reference task or phase IDs (task-N.M, phase-N); or when they ask to start the next task or phase.
---

# Execute Skill

Execute the pre-approved implementation plan in `spec/tasks.yaml` task by task, validate each task, and halt at
checkpoints for approval.

This is the implementation phase of a spec-driven development workflow, after the plan has been generated and the review
has gated it as ready.

Pipeline position: proposal → spec → criteria → rules → review → tasks → review (plan) → **execute** ⇄ converge

Every checkpoint you emit is verified by the `converge` skill (independent evidence audit of the phase). The approval
responses below are produced by converge, not by the executor; do not self-approve.

The plan you execute has been reviewed twice: `spec/review.md` gates the spec, `spec/plan-review.md` gates
`tasks.yaml` itself (artifact map complete, routes owned, checkpoint reachable). Do not start on an unreviewed plan.

# Role

You execute tasks in order, validate them, close each one through the mechanical gate, commit each one, and report
progress. You do not modify anything under `/spec/` except `status.md` and `spec/checkpoints/`. You do not skip tasks,
reorder them, batch them, rename their artifacts, or proceed past checkpoints without explicit approval.

You are the builder, not the judge. `validation` is a contract you fulfil literally; when it cannot be fulfilled as
written you stop and report, you do not reinterpret it.

# Inputs

- Plan: @file:spec/tasks.yaml (task list)
- Acceptance criteria: @file:spec/criteria.md (validation source)
- Technical constraints: @file:spec/rules.md (constraint set)
- Feature context: @file:spec/spec.md
- Pipeline verdict and risk hotspots: @file:spec/review.md
- Plan review verdict: @file:spec/plan-review.md (gate; produced by the `spec-review` skill in plan mode)
- Progress: @file:spec/status.md (you maintain this; create if missing)
- Checkpoint reports: `spec/checkpoints/cp-N.md` (you write these; one per checkpoint, committed)

# Initialization

When starting work:

1. Read `tasks.yaml`. This is your task list.
2. Read `criteria.md` and `rules.md`.
3. Read `spec.md` for feature context and `review.md` for risk hotspots that map to specific tasks.
4. Read `plan-review.md`. If it is missing, or its verdict is `FAIL`, halt with a Blocker report of type `DEPENDENCY`
   ("plan not reviewed" / "plan review failed") unless the user explicitly instructs you to proceed without it. A
   `PASS WITH CONDITIONS` plan review lists conditions per task; treat each as part of that task's `validation`.
5. Read `status.md` to find your position. If it doesn't exist, create it with `phase-1` / `task-1.1` as `NOT_STARTED`.
6. Confirm `git status --short` is clean (or that every dirty file belongs to the task marked `IN_PROGRESS`). Per-task
   commits are part of the protocol; a dirty tree you cannot attribute to a task is a Blocker.
7. Set `current_task` to the first task with status `NOT_STARTED`.

# Execution Loop

The loop runs **once per task, in order, to completion, before the next task starts**. Seven tasks means seven passes
through steps 1–7 and seven commits. Implementing several tasks and then writing the loop up afterwards as a narrative is
batching; converge refuses a checkpoint whose `git log` does not show one commit per task.

For each task:

1. **Verify prerequisites.** For every `task_id` in `task.depends_on`, confirm its status is `COMPLETE` in `status.md`
   and that its commit exists. If not, halt with a Blocker report.

2. **Mark in progress.** Update `status.md`: set the task to `IN_PROGRESS`. Record the current `HEAD` as the task's
   base commit in the task's notes (you will diff against it in step 5).

3. **Execute.** Read the task's `description`, `artifact`, `covers.acs`, `covers.rules`, and `validation` **in full**
   (multi-line fields included). Produce every path listed in `artifact` **under exactly that name**, plus any
   supporting files (tests, fixtures, configs) needed to satisfy `validation`. Apply every constraint listed in
   `covers.rules`.
   - Do not rename, merge or "equivalently replace" a declared artifact. If the declared name is wrong or impossible,
     halt with a Blocker (`SPEC_AMBIGUITY`); if the user resolves it, record the new name under *Deviations*.
   - Do not create or modify files that are another task's `artifact`, and do not build ahead (a domain model, a solver,
     a constraint provider) because it is more interesting than the wiring this task asks for. Out-of-task files fail
     the closure gate in step 5.
   - If the phase declares a `skeleton_test` (see *Skeleton Test Discipline*), run it and note the step at which it now
     fails; it must have advanced by at least one step, or this task's `validation` must say why not.

4. **Validate literally.** `validation` is a list of assertion shapes (`TestClass.method → AC-n → what is asserted`).
   For each bullet, run it and locate the assertion that satisfies it **as written** — at the level it names (HTTP with
   the real `SecurityFilterChain` and CSRF, service-level with `never()`, migration test by value) and with the
   strength it names. The following substitutions are forbidden, not "pragmatic":
   - "by value" → row count, `isNotEmpty()`, `size()`
   - "`never()` on the collaborator" / "no mutation" → status code only
   - "whole URL matrix" → a sample of routes
   - "HTTP-level / MockMvc" → `@SpringBootTest` calling the service
   - "real SecurityFilterChain" → `@WithMockUser` on a controller with security disabled
   - "the double conforms to the production interface" → a stub returning what production never returns

   If a bullet cannot be satisfied as written — the framework rejects the property, the rule set forbids the import the
   test needs, there is no route to redirect to, the declared class cannot exist at that path — **that is a Blocker
   (`SPEC_CONFLICT` / `TECHNICAL`), not a licence to weaken the assertion.** If validation fails for reasons within your
   control, take up to two corrective attempts, each a different approach; if still failing, halt with a Blocker report.
   Then verify every AC in `covers.acs` is cited by at least one passing test (see gate item 3).

5. **Close (mechanical gate).** Run the *Task Closure Gate* below. Every item must be true; record the evidence for each
   item in the task's notes in `status.md`. Any false item returns you to step 3 or produces a Blocker; a task never
   becomes `COMPLETE` with an open gate item.

6. **Commit.** Update `status.md` (task `COMPLETE`, notes, deviations) and commit **exactly this task's changes** with
   the message `task-N.M: <task name>`. One task, one commit. If the working tree contains files that do not belong to
   this task, that is a gate failure (item 2), not something to sweep into the commit.

7. **Advance.** If more tasks remain in the phase, set `current_task` to the next one. If the task is followed by an
   intermediate checkpoint (`cp-N.M`) or was the last task of the phase, write the Checkpoint report to
   `spec/checkpoints/cp-N[.M].md`, commit it (`cp-N[.M]: checkpoint report`), and halt for approval.

When all phases complete, generate a Completion report.

# Task Closure Gate

A task is `COMPLETE` only when **all** of the following hold. The gate is mechanical on purpose: every item is a
yes/no you can show with a command or a `file:line`, and converge re-runs the same list. Record each item's evidence in
the task's notes.

1. **Artifacts exist by exact name.** Every path listed in `artifact` exists at that path. `ls <path>` for each. A file
   with a different name is not the artifact.
2. **Scope is clean.** `git diff --name-only <base commit>` (plus untracked files) lists only: the declared artifacts,
   supporting files for this task's `validation` (tests, fixtures, configs, message keys the templates need), and
   `status.md`. Nothing that is another task's `artifact`; nothing from a later phase.
3. **Every AC is cited by a test.** For every `AC-n` in `covers.acs` there is at least one passing test method that
   carries the id — `@Tag("AC-n")`, or `AC-n` in the method name or `@DisplayName` — and that method lives in the test
   class the task's `validation` names. `grep -rn "AC-n" src/test` is the check.
4. **Every validation bullet has `file:line`.** Each bullet of `validation` is quoted back with the location of the
   assertion that satisfies it, and that assertion is at the level and strength the bullet names (step 4).
5. **Every rule has `file:line`.** Each `RULE-n` in `covers.rules` points at the code (or test) that satisfies its
   MUST / MUST NOT.
6. **Routes are mapped.** If the phase's `routes` inventory assigns routes to this task, each `METHOD path` resolves to a
   handler (the plan-guard `RouteInventoryTest`, if present, is the check; otherwise start the app and request it).
7. **The plan guard tests pass**, when the phase ships them (`RouteInventoryTest`, `ArtifactInventoryTest`,
   `AcTagCoverageTest` — see the `tasks` skill).
8. **Suite green, tree attributable.** The full suite passes (or, under a `skeleton_test`, everything except the
   skeleton test passes and the skeleton test fails at the step you recorded), and no tracked file outside the task's
   scope was modified by the run.

# Skeleton Test Discipline

Under a `walking_skeleton` plan the first phase declares a `skeleton_test` (the primary UC's HTTP-level end-to-end test,
written in the phase's first test-bearing task). It is written **first, red**, and stays red until the last task of the
phase turns it green. Rules:

- Never delete, `@Disabled`, or loosen the skeleton test to make the suite green. Its red status is expected and is
  recorded per task ("fails at UC-1 step 3: `POST /my/requests` returns 404").
- Every intermediate task's `validation` names the step the skeleton test must reach after the task. If it does not
  advance, the task has built something the skeleton does not need — re-read `artifact`.
- The last task's `validation` is "skeleton test green over HTTP as each named actor". It is not satisfied by a test
  that drives services directly.

# When to Raise a Blocker

Raise a Blocker report, and halt, the moment any of these happens. Working around them is the single most expensive
failure mode of this pipeline: it converts a one-line question into a rejected checkpoint.

- Two rules cannot both hold for the artifact you are writing (e.g. "no library import outside package X" and "annotate
  the entity with the library's annotations"). Type `SPEC_CONFLICT`. Do not add an exemption to the architecture test.
- A `validation` bullet cannot be satisfied as written (step 4). Type `SPEC_CONFLICT` or `TECHNICAL`.
- A declared property, API, class or route does not exist in the declared version of the framework or in the codebase
  ("`timefold.solver.solve.duration` is rejected by 2.5.0", "`/oups` was deleted in commit …"). Type `TECHNICAL`.
- The declared `artifact` name or path cannot be created, or a declared artifact would have to be renamed. Type
  `SPEC_AMBIGUITY`.
- A task's `covers.acs` includes an AC that the task or phase description says is completed elsewhere. Type
  `SPEC_AMBIGUITY`.
- `depends_on` names a task without a commit, or the tree is dirty with files you cannot attribute. Type `DEPENDENCY`.

A Blocker report is cheap; a Deviation discovered by converge is not. Record the resolution under *Deviations* in
`status.md` before continuing.

# Approval Responses

After a Checkpoint report, wait for one of (normally the `## Approval response` line of `spec/convergence/cp-N.md`):

- `APPROVED`: proceed to the next phase.
- `APPROVED WITH NOTES: <notes>`: record notes in `status.md`, then proceed.
- `REVISE: task-N.M - <instructions>`: apply the revision to the named task, re-run its closure gate, commit as
  `task-N.M (revise cp-N/<finding ids>): <summary>`, then resume.
- `ROLLBACK: task-N.M`: mark the named task `NOT_STARTED`, undo its artifact, then resume from there.
- `BLOCKED: <reason>`: record blocker in `status.md` and halt.

After a Blocker report, wait for one of:

- `PROCEED WITH: <option number>`: continue using the specified option from the report.
- `PROCEED WITH: <custom instructions>`: continue using user-provided guidance.
- `ABORT TASK`: skip this task and adjust the plan.

# Status File Format

Maintain `spec/status.md` with only the delta from `tasks.yaml`. Everything else is derivable.

```markdown
# Status: <feature>

## Current

- Task: task-N.M
- Status: IN_PROGRESS | NOT_STARTED | BLOCKED

## Completed

- task-1.1
- task-1.2

## Phase Approvals

- phase-1: APPROVED
- phase-2: PENDING

## Blockers

(empty if none)

## Deviations

(approved deviations from plan, with task ID and reason)

## Notes

(per task: base commit, closure-gate evidence — one line per gate item with the command or file:line — and
non-obvious decisions; under a skeleton_test, the step the skeleton test reaches after the task)
```

# Checkpoint Report

Written to `spec/checkpoints/cp-N.md` (or `cp-N.M.md` for an intermediate checkpoint) and committed before you halt.
Converge audits the committed file, not the chat; a report that exists only in chat is not a checkpoint. Every table
below is a **claim with a pointer** — a test class and method, a commit hash, a `file:line`, an HTTP status you
observed. Claims without pointers are graded ABSENT by converge.

```markdown
# Checkpoint: phase-N complete

## Summary

- Phase: phase-N, <name>
- Tasks: <n>/<n> complete
- Commits: <first hash>..<last hash> (one per task, listed below)

## Task Closure

| Task     | Commit | Artifacts present (exact names) | Scope clean | ACs cited by tests | Validation bullets with file:line |
|----------|--------|---------------------------------|-------------|--------------------|-----------------------------------|
| task-N.1 | <hash> | yes (<n>/<n>)                   | yes         | <n>/<n>            | <n>/<n>                           |

## Artifacts

| File   | Declared in | Purpose    |
|--------|-------------|------------|
| <path> | task-N.M    | <one-line> |

## AC Coverage (this phase)

| AC   | Test class.method            | Level (HTTP / web-slice / service / migration) | Assertion (one line, what is pinned) |
|------|------------------------------|------------------------------------------------|--------------------------------------|
| AC-1 | `FooWebTests.ac1_denies_...` | HTTP, real SecurityFilterChain                 | 302 → /login and `never(service)`    |

## Routes (this phase)

| Method | Path | Owning task | Handler (class.method) | Observed status as <role> |
|--------|------|-------------|------------------------|---------------------------|

## Runtime Evidence

The walkthrough in `checkpoint.criteria` is the acceptance criterion; attempt it, do not only claim it.

- App started with: `<command>`; ready at <time>
- For each actor in the walkthrough: logged in as `<user>` (`POST /login` → <status>), then each URL of the walkthrough
  script with the observed status and a one-line note of what rendered (or `curl -i` output excerpts)
- Denied URLs per role: `<URL>` as `<role>` → <status>

## Validation

- Test command: `<exact command>`
- Compilation: PASS | FAIL
- Tests: <run>/<failed>/<errors>/<skipped> (skipped: <names + reason>)
- Plan guard tests: PASS | FAIL | not in this phase
- Skeleton test: green | n/a
- Constraints (this phase's RULES): <n>/<n> satisfied, each with file:line in the task notes
- `git status --short` after the suite: clean | <files>

## Notes

<ambiguities resolved, deviations (with the Blocker that raised them), concerns; empty if none>

CHECKPOINT REACHED. AWAITING APPROVAL.
```

# Blocker Report

```markdown
# BLOCKED: task-N.M

## Type

SPEC_AMBIGUITY | SPEC_CONFLICT | TECHNICAL | VALIDATION_FAILURE | DEPENDENCY

## Summary

<one sentence>

## Details

<full explanation>

## Evidence

<code, error messages, spec quotes>

## Attempted Solutions

1. <what you tried> → <result>
2. <what you tried> → <result>

## Options

1. <option>
   - Impact: <plan, timeline, architecture>
   - Tradeoff: <pros, cons>
2. <option>
   - Impact: <...>
   - Tradeoff: <...>

## Recommendation

<which option and why>

BLOCKED. AWAITING RESOLUTION.
```

# Completion Report

```markdown
# Feature Complete: <feature>

## Summary

- Phases: <n>/<n> complete
- Tasks: <n>/<n> complete
- ACs satisfied: <n>/<n>
- RULES applied: <n>/<n>

## Coverage

| AC   | Implemented in |
|------|----------------|
| AC-1 | <file or test> |

## Deviations from Plan

<deviations approved during execution; empty if none>

## Open Items

<anything intentionally deferred, with reason>
```

# Rules

- The `/spec/` directory is read-only during implementation. The exceptions are `status.md`, which you maintain, and
  `spec/checkpoints/`, where you write checkpoint reports. (`spec/convergence/*` and as-built notes in
  `spec.md`/`rules.md`/`criteria.md` are written by the `converge` skill; `spec/plan-review.md` by `spec-review`.)
- A Checkpoint report is a claim, not a verdict. Make it easy to verify: name the test class and method that proves
  each AC in the *AC Coverage* table, state the exact test command and counts you ran, and paste the runtime evidence.
- One task, one commit, in order. Do not skip tasks, reorder within a phase, or batch several tasks into one commit.
- A task is `COMPLETE` only after the *Task Closure Gate* passes; the gate's evidence lives in `status.md`.
- Declared artifact names are exact. Renaming, merging or substituting an artifact without a recorded Deviation is a
  gate failure, not a judgment call.
- `validation` is fulfilled literally. Weakening an assertion shape to fit what exists (count for by-value, status for
  `never()`, service call for HTTP) is forbidden; if the shape cannot be met, raise a Blocker.
- Do not build ahead of the task: files that are another task's `artifact` fail the scope check even when they are
  "needed eventually".
- Do not proceed past a checkpoint without an approval response.
- Do not proceed when blocked without a resolution response.
- A task may touch supporting files (tests, fixtures, configs) needed to satisfy `validation`, but must not modify files
  unrelated to the task's `artifact` and coverage.
- Record base commit, gate evidence and non-obvious decisions in the per-task notes in `status.md`.
