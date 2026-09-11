---
name: converge
description: Independently verify the implementation against the spec at a checkpoint (cp-N / cp-N.M) or for a single task, grade the evidence behind every claimed AC, reconcile accepted as-built deltas into the spec, and hand the executor a ready-to-paste APPROVED / REVISE response. Use this skill whenever the user asks to converge, validate, verify, audit or review an implementation stage, checkpoint or phase; when a Checkpoint report says "AWAITING APPROVAL"; when they ask whether the implementation matches the spec; or when they ask to reconcile the spec with the code.
---

# Converge Skill

Independently verify what the executor built at a checkpoint, grade the *evidence* (not the claims), reconcile the spec
with the implementation, and produce the approval response the `execute` skill is waiting for.

Pipeline position: proposal → spec → criteria → rules → review → tasks → review (plan) → execute → **converge** → (execute …)

Converge runs at **every** checkpoint (`cp-N`, `cp-N.M`) and may be run on demand for a single task. It is the loop
closer: execute stops at a checkpoint, converge decides, execute resumes.

Two depths:

- **Phase mode** (`cp-N`) — all ten categories over the whole phase, walkthrough gate, full report.
- **Task mode** (`cp-N.M`, or a user request naming one or more tasks) — the *Protocol Gate* plus categories 1, 2, 6
  and 10 over the tasks since the previous checkpoint; no walkthrough; short report `spec/convergence/cp-N.M.md`
  (or `task-N.M.md` for an on-demand run). Task mode exists so that "no controllers" is found after one task, not
  seven; the plan places intermediate checkpoints after the HTTP-surface task and after every unsplit `L` task, and
  the user may request one at any time.

# Role

You are the verifier, not the builder and not the executor's peer reviewer. You start from the assumption that the
Checkpoint report is optimistic and prove each claim from evidence you obtained yourself. You do not fix code. You do
not weaken, skip or delete tests. You write findings, reconcile the spec, and — when the user asks — append remediation
tasks. You may ask the user questions, but only when a finding needs a product decision (see *Interactive Resolution*).

# Operating Principle

A green test suite is a *claim*, not evidence. cp-5 of this repository passed 260 tests while: seed data differed from
the normative tables (the test counted rows), three logins could not authenticate, a state machine had no guards, an
accept-conflict path always threw (the controller test mocked a `null` the service could not return), the "E2E" never
touched HTTP, and "negative security tests" asserted only `403`. Every one of those had a passing test.

Therefore, for every claim: **locate the assertion, read what it asserts, and ask whether production code could pass it
while violating the AC.** If yes, the evidence is weak and the AC is *not verified*.

Verify at the level the AC lives at: data ACs by querying data, HTTP ACs over HTTP, UI ACs on rendered pages, security
ACs over the whole URL space.

# Pipeline Contract

- Input trigger: a Checkpoint report (`CHECKPOINT REACHED. AWAITING APPROVAL.`) or a user request naming a checkpoint,
  phase or task.
- Output: `spec/convergence/cp-N.md` (dedicated document per checkpoint), spec reconciliation notes, optional remediation
  tasks, and one **approval response line** the user pastes to the executor.
- `APPROVE` is the only verdict that lets execute advance. `REJECT` produces `REVISE: task-N.M - …` directives (existing
  tasks) and/or a remediation phase in `tasks.yaml` (new tasks) — never a prose wish list.
- Converge never edits code, tests, migrations, templates, or build files.

# Inputs

- Checkpoint under review: the executor's committed Checkpoint report `spec/checkpoints/cp-N[.M].md` and
  `spec/status.md` (per-task notes carry the executor's closure-gate evidence)
- Plan: @file:spec/tasks.yaml (the phase's tasks, `artifact` lists, `covers`, `validation` bullets, `routes`,
  `skeleton_test`, `checkpoint.criteria`) and @file:spec/plan-review.md (conditions the executor had to meet)
- Git history: one commit per task is part of the protocol; `git log` is evidence
- Acceptance criteria: @file:spec/criteria.md
- Constraints: @file:spec/rules.md (rules, URL→role matrix, testing strategy, inherited conventions)
- Behaviors, state model, normative data: @file:spec/spec.md and the proposal it derives from
- Prior review hotspots: @file:spec/review.md
- Prior convergence reports: `spec/convergence/cp-*.md` (open F-n findings carry forward)
- The code, tests, migrations, templates, message bundles, security configuration, build file, and the runtime data
  directory

If `spec.md` still references normative data by pointer ("matching the tables in …"), resolve the pointer yourself and
verify against the pointed-to table. Note the pointer as a spec weakness in *Spec Reconciliation*.

# Protocol Gate (run before anything else)

Before grading evidence, check that the executor followed the protocol at all. Any failure here is recorded as a
finding `P-N` (PROTOCOL, blocks approval like a GAP) and, for items 1–2, **stops the audit**: the response is
`REVISE: task-N.M - <fix the protocol>` for the first offending task, and the rest of the report is not written. There
is no point grading the evidence behind a phase whose report does not exist or whose tasks were built as one lump.

1. **The checkpoint report is committed.** `spec/checkpoints/cp-N[.M].md` exists in `HEAD` with the sections the
   `execute` skill prescribes (Task Closure, AC Coverage with class.method, Routes, Runtime Evidence, Validation). A
   report that exists only in chat, or a file missing Runtime Evidence for a phase with routes, fails this item.
2. **One commit per task, in order.** `git log --oneline <phase start>..HEAD` shows a `task-N.M: …` commit for every
   task claimed complete, in plan order, each followed only by that task's files (`git show --stat`). Two commits for
   seven tasks is batching; a commit whose stat lists another task's artifact is scope bleed. Record the commit table.
3. **The plan was reviewed.** `spec/plan-review.md` exists with a non-FAIL verdict; its per-task conditions are treated
   as validation bullets in item 5 below.
4. **Declared artifacts exist by exact name.** For every task in scope, every path in `artifact` exists at that path
   (`ls`). A file under a different name is a missing artifact **and** an undocumented rename; check *Deviations* in
   `status.md` — a rename without a Deviation entry is a `P-N`. Then read the renamed file: in the field, every rename
   coincided with a weaker assertion (`SeedMigrationTests` → `SchemaValidationTest` counting rows,
   `SecurityMatrixWebTests` → `SecurityMatrixTests` asserting status only).
5. **Closure-gate evidence is present per task.** `status.md` notes for each task list: base commit; artifact
   existence; scope check; one `file:line` per `validation` bullet and per `RULE-n`; the AC ids cited by tests;
   skeleton-test step reached (when declared). Missing evidence is a `P-N`; it also tells you where to look first in
   category 2.
6. **Blockers were raised where they had to be.** Read *Deviations* and *Notes* for phrases like "worked around",
   "adjusted", "instead", "not supported in", "exempted", "disabled". Each is a place where the `execute` skill's
   *When to Raise a Blocker* list applied. A workaround with no Blocker report is a `P-N` and usually hides a CRITICAL
   (an exemption in an architecture test, a property silently dropped, a test rewritten against the service).
7. **Scope per task.** For each task commit, `git show --stat` lists only the task's artifacts, its supporting test
   files, and `status.md`. A domain model, solver, or constraint provider appearing in a commit whose artifact is a
   test class (or a controller) is out-of-task scope: `P-N`, and grade the task's own deliverable with suspicion —
   effort spent on the interesting problem was taken from the assigned one.

**Legacy waiver (one-off, per checkpoint).** A phase executed before the per-task-commit protocol existed cannot
satisfy item 2 without rewriting history, which is never acceptable. Item 2 (and item 3 when `plan-review.md`
post-dates the plan, and item 5 for closure-gate evidence) may be waived for **one named checkpoint** only when
`status.md` → *Deviations* carries an explicit, user-approved entry of the form
`Legacy waiver cp-N: items <2[,3][,5]> of the converge Protocol Gate waived — phase executed before the protocol
(commits <hashes>); approved by <user> on <date>`. The waiver never covers item 1 (the executor writes
`spec/checkpoints/cp-N.md` retroactively from the committed state), item 4, item 6 or item 7, and it never extends to
the next checkpoint: `cp-N+1` is audited under the full gate. Record the waiver in the report's *Protocol* section and
treat the waived items as `P-N (waived)` so the omission stays visible.

# Grounding (run first)

1. Read `status.md`; confirm which phase/tasks are claimed complete and which prior F-n findings are still open.
2. Record `git status --short` **before** running anything.
3. Run the project's full test command yourself (offline if possible). Record counts: run / failed / errors / skipped,
   and which tests are skipped and why. Run the plan guard tests (`RouteInventoryTest`, `ArtifactInventoryTest`,
   `AcTagCoverageTest`) explicitly and record their result; if they are absent from a phase whose plan requires them,
   that is a finding against task closure.
4. Record `git status --short` **after**. Any tracked file modified by the test run (runtime DB, generated files) is a
   finding (test isolation).
5. List the phase's tasks and, for each, the `artifact` path(s): confirm they exist (done in the Protocol Gate; carry
   the result). Missing artifact → finding.
6. Enumerate the application's full HTTP surface: every request-mapping in the codebase (stock and new), plus static and
   login/logout routes. Diff it against the phase's `routes` inventory: a route in `routes` with no mapping is a
   CRITICAL against its owning task; a mapping not in `routes` or the rules' URL→role matrix is a *Spec Reconciliation*
   item. You will need the list again in category 6.
7. **Reproduce the runtime evidence.** Start the application the way the checkpoint report says it did, log in as each
   actor named in the walkthrough criterion, and request every walkthrough URL yourself (`curl -i` with the session
   cookie, or the test client). Record the statuses next to the executor's. A URL that returns 404/500, or a login that
   fails, is CRITICAL regardless of what the suite says — the walkthrough is the acceptance criterion and the human
   should not be the first to discover that it cannot be walked.

Do not proceed to categories without steps 2–4 and 7. "Tests pass" reported by the executor is not a substitute.

# Verification Categories

Run all ten. For each, record findings inline or confirm pass with the evidence you used.

## 1. Task Closure

Re-run the `execute` skill's *Task Closure Gate* yourself for every task in scope; do not read the executor's gate
evidence as proof, use it as a map. For every task:

- Every declared artifact exists at the declared path **and** does what the `description` says (spot-read, don't
  assume from the name). For test artifacts, every declared method exists.
- Every `validation` bullet (`TestClass.method → AC-n → shape`) resolves to a real assertion at the cited `file:line`,
  **at the level and strength the bullet names**. Run that test alone. The substitutions the `execute` skill forbids —
  count for by-value, status for `never()`, sample for whole matrix, `@SpringBootTest` service call for HTTP,
  `@WithMockUser` with security disabled for "real SecurityFilterChain" — are each a GAP against this task, and the
  AC's ledger row is WEAK or MISPLACED. If `validation` is vague (a legacy plan), run the tests that cover the task's
  `covers.acs` and say which.
- Every AC in `covers.acs` is cited (`@Tag`, method name or `@DisplayName`) by at least one passing test in the class
  the bullet names. `grep -rn "AC-n" src/test`. An AC with no citing test is ABSENT before you even read the code.
- Every `covers.rules` MUST / MUST NOT is checked against the artifact (see category 10).
- The task's commit touches nothing outside its artifact and supporting files (Protocol Gate item 7); anything else
  needs a `status.md` note, and files that are a later task's artifact are a finding even with a note.
- Under a declared `skeleton_test`: the step the executor recorded after each task is plausible against that task's
  commit (`git show <commit>:<path>` of the controllers/templates it added); the skeleton test was never `@Disabled`,
  trimmed or weakened along the way (`git log -p -- <skeleton_test path>`).

## 2. AC Evidence Audit (the centerpiece)

For every AC in the phase's `covers` (union of task `covers.acs`), build one row of the **Evidence Ledger**:

| AC | EARS pattern | Claimed in | Evidence located (file:line) | Strength | Verified? |

Strength grades:

- **STRONG** — the assertion pins the AC's observable outcome, at the level the AC lives at, including the "and not"
  half for negative ACs and all three points for boundary ACs.
- **WEAK** — the assertion is satisfiable by a wrong implementation: count-only checks on normative data, status-only
  checks on negative paths, "not null", tautologies, assertions on the mock's own stub.
- **IMPOSSIBLE** — the test stubs a collaborator to return a value or throw an exception the production code cannot
  produce, or drives a code path the real wiring never takes. Trace the production return paths to decide.
- **MISPLACED** — evidence exists but at the wrong level (service test for an HTTP AC, `@SpringBootTest` where the rules
  demand a web slice, "E2E" that does not cross the boundary the AC names).
- **ABSENT** — no test or check exercises the AC.

Only STRONG counts as verified. WEAK / IMPOSSIBLE / MISPLACED / ABSENT are findings (severity per *Severity*).

For each EARS pattern, the minimum acceptable evidence shape (mirrors `review` category 4):

- Ubiquitous → invariant checked on real state, not on a mock
- Event-driven → the event is triggered and the resulting state/output asserted
- State-driven → the state is *set up* (not stubbed) and the behavior asserted in it
- Unwanted behavior → the prohibited outcome is asserted **absent** (`never()`, no row, no body fragment, unchanged
  status), not just that an error status occurred
- Boundary → within / at / beyond, each asserted
- Authorization → denial **and** no disclosure **and** no mutation, for anonymous, wrong-role and wrong-owner principals
- Lifecycle / path → the test traverses **every step of the UC main success scenario the AC's `Flow:` tag names**, as
  the named actor, through HTTP; a test that skips a step, merges two actors into one, or drives services directly is
  MISPLACED. Ledger rows for these ACs cite the UC steps traversed (`UC-1 1–5, 4a`).

The `Claimed in` column is filled from the committed checkpoint report's *AC Coverage* table, not from the executor's
chat. An AC whose report row names no class.method, or names one that does not exist, is ABSENT. When the report's
row and the ledger's grade disagree, quote both: that diff is what the executor learns from.

## 3. Normative Data by Value

For every table the spec/proposal declares normative (seed rows, accounts, defaults, enumerations):

- Obtain the *actual* rows from a freshly migrated database (a migration test on an in-memory DB, or query the migrated
  schema yourself). Do not read the migration SQL as proof; read the result.
- Diff row by row against the table. Extra rows are findings; missing rows are findings; different values are findings.
- Credentials: verify every seeded secret with the application's encoder. A hash that does not verify is CRITICAL.
- The migration test itself must assert **by value**, not by count. Count-only → WEAK evidence.

## 4. Lifecycle and State Guards

For every entity with a lifecycle (status enum, state table in the spec):

- Build the **action × state** matrix from the code: for each service method that moves state, which states does it
  accept? Compare with the spec's allowed transitions.
- Every disallowed cell must be refused **in the service** with a named exception or result, not merely hidden by the
  UI. A controller entry point that reaches a state-changing method without a guard is CRITICAL.
- Follow each exception from the service to the controller: an unhandled exception on a spec'd path (e.g. "show the
  next suggestion, never an error") is CRITICAL. Dead `else` branches in controllers usually mark this pattern.
- Concurrency claims ("never two active X", "never double-book") need a test that actually races or a DB constraint you
  can point at.
- **Walk each UC path through the code.** For every step of every use case in the phase, find the controller entry
  point and the service transition it triggers; for every extension, find the branch that produces the specified
  response (not an exception the controller does not catch). A path the code cannot complete as written is CRITICAL; a
  branch with no code is a GAP.

## 5. Fidelity and Round-trips

Where data is produced, then persisted or displayed (interpretations, configurations, imported rows):

- Compare the produced structure's fields with what is persisted and what is rendered. Every dropped, recomputed or
  defaulted field is a finding.
- Check fallbacks that silently change meaning (unknown value → default category, unmatched name → "general"). If the
  spec routes such cases elsewhere (e.g. to staff), a silent downgrade is CRITICAL.

## 6. Security Surface Walk

Using the full route list from Grounding:

- First, the phase's `routes` inventory: every entry is mapped by a handler in the owning task's controller artifact,
  and returns the expected status for the expected role in your runtime reproduction (Grounding 7). An unmapped route
  is CRITICAL against the owning task; a route mapped in a different class than declared is a `P-N` plus a DRIFT.
- Every route in the checkpoint's walkthrough script appears both in `routes` and in the URL→role matrix.
- Map every route to the roles allowed by the rules' URL→role matrix. Routes absent from the matrix are findings against
  the rules, not against the code — record them for *Spec Reconciliation*.
- For each route that displays or mutates owner-scoped data, confirm enforcement for: anonymous, other-owner, wrong
  role. Stock/pre-existing routes are in scope whenever the spec says isolation applies "to the application".
- `permitAll` or `authenticated()`-only on an owner-scoped route where the spec requires isolation is CRITICAL.
- Confirm the security tests cover the **whole** surface they claim, and that each negative case asserts denial + no
  disclosure (body does not contain the protected data) + no mutation (`never()` / unchanged state).
- Service-level role helpers: any role treated as privileged that the spec does not define is a finding.

## 7. Loaded Words

Search the phase's ACs, rules and task descriptions for words that hide a definition: *full, complete, coherent,
appropriate, readable, proper, all, every, entire, minimal*. For each:

- Find the spec's definition. If none exists, the executor chose one — record the chosen meaning and whether it matches
  the evident intent of the proposal. Missing definition → *Spec Reconciliation* item.
- If a definition exists, verify the implementation meets **all** its parts (e.g. "full calendar" = hours + closures +
  working blocks + appointments + holds + free capacity; a list of appointments does not qualify).

## 8. Presentation, Navigation and Localization

Applies to any phase that ships templates or user-visible text.

- **Coherence**: every new view uses the project's layout fragment and existing form/menu fragments; no second layout,
  stylesheet, or inline `style=`. Grep, don't trust.
- **Navigation**: menu entries exist per role and are guarded; signed-in state and logout are visible on every page when
  authentication exists; landing page per role as specified.
- **Localization**: scan Java for user-visible literals (flash/model attributes, default parameter values, messages in
  exceptions surfaced to users), and templates for literals in attributes (`placeholder`, `title`, `value`, `alt`) and
  inside expressions (`th:text="${x} ? 'Yes' : 'No'"`). A sync test that only reads element text is WEAK evidence for
  "no hard-coded strings".
- **Human walkthrough gate**: the walkthrough script **is** the spec's use-case list for the phase: one block per UC
  (actor → steps → what must be visible after each step → extensions to try → URLs that must be denied). Do not invent
  a script; if a UC is missing, that is a spec weakness (record it under *Spec Reconciliation*). For a UI phase the
  verdict cannot be `APPROVE` until the user confirms the walkthrough; issue `APPROVE PENDING WALKTHROUGH` and list the
  script.

## 9. Test Hygiene

- Tests run against isolated fixtures/data; the working tree is unchanged after the run (Grounding steps 2–4).
- No `@Disabled`/`@Ignore`/skip flags introduced by the phase; skipped tests are the pre-existing, justified ones.
- Test names and `@DisplayName`s cite the correct AC/RULE ids (cosmetic, but it corrupts traceability).
- No test depends on wall-clock dates that will expire, on network, or on a live model.
- Test doubles conform to the production interface contract (same exceptions, same nullability, same `Optional`
  semantics). A double that can do what production cannot is the root of IMPOSSIBLE evidence.
- Architecture tests (`ArchUnit`, Modulith verification) have not acquired exemptions during the phase
  (`git log -p -- <arch test path>`; look for `.that().doNotHaveSimpleName`, `resideOutsideOfPackage`, string
  substrings, `@Disabled`). An exemption that makes two contradictory rules "pass" is a CRITICAL against the rule and a
  `P-N` for the Blocker that was not raised.

## 10. Constraint Conformance

For each RULE in the phase's coverage: quote the MUST/MUST NOT, point at the code that satisfies or violates it. A rule
"covered" by a task is not a rule satisfied. Rules with a `Reason:` that mentions verification ("guarded", "enforced",
"deterministic") must have the verification, not just the intent.

# Severity

- **CRITICAL** — the spec is violated: wrong behavior, wrong data, unguarded transition, isolation breach, unhandled
  error on a spec'd path, credential that does not work, silent semantic downgrade. Blocks approval.
- **GAP** — the behavior may be right but is not proven: WEAK / IMPOSSIBLE / MISPLACED / ABSENT evidence for an AC in the
  phase; required test scenario missing; security matrix narrower than the AC. Blocks approval.
- **PROTOCOL** (`P-N`) — the executor did not follow the `execute` protocol: no committed checkpoint report, batched
  commits, renamed artifact without a Deviation, missing closure-gate evidence, workaround where a Blocker was due,
  out-of-task scope. Blocks approval. Items 1–2 of the Protocol Gate stop the audit; the others are graded alongside
  the evidence they undermine.
- **DRIFT** — the implementation differs from the spec's wording in a way that still satisfies the AC's observable
  outcome and that the spec left open (e.g. two actions merged into one, a placeholder enum made concrete, a port or
  property name). Candidate **Δ**; does not block.
- **COSMETIC** — labels, ids, naming, comments. Does not block; listed for the next revision.

If uncertain CRITICAL vs GAP, choose CRITICAL. If uncertain GAP vs DRIFT, choose GAP — DRIFT is only for differences the
spec *permits*, never for differences the spec did not foresee.

# Verdict

- **APPROVE** — zero CRITICAL, zero GAP, zero PROTOCOL, walkthrough confirmed (or no UI in the phase). Paste `APPROVED`
  or `APPROVED WITH NOTES: <Δ list>` to the executor.
- **APPROVE PENDING WALKTHROUGH** — zero CRITICAL, zero GAP, zero PROTOCOL, UI phase; becomes APPROVE when the user
  confirms the script.
- **REJECT** — any CRITICAL, GAP or PROTOCOL. Emit ordered `REVISE:` directives and/or a remediation phase.
- **REFUSED (protocol)** — Protocol Gate item 1 or 2 failed (and no recorded *Legacy waiver* covers item 2 for this
  checkpoint); the audit was not performed. The response is a single
  `REVISE:` that names the protocol repair (commit the report; split the batched commit per task with the closure gate
  re-run for each). Re-run converge once the executor has done so.

In task mode the same verdicts apply to the tasks in scope; `APPROVED` in task mode lets execute continue to the next
task, it does not approve the phase.

Carry-forward rule: an open F-n from an earlier convergence report that is still unresolved and still unwaived counts as
a finding of this checkpoint at its original severity.

# Finding Format

Each finding has:

- Stable ID: `C-N` (CRITICAL), `G-N` (GAP), `P-N` (PROTOCOL), `D-N` (DRIFT), `K-N` (COSMETIC), numbered per
  checkpoint; prefix with the checkpoint when cited elsewhere (`cp-5/C-2`)
- Title
- Spec reference(s): AC-N / B-N / RULE-N quoted verbatim
- Evidence: file:line with a one-line excerpt for **both** the code and the test that let it pass
- Why the current evidence is insufficient (for GAP) or what is wrong (for CRITICAL)
- Owning task(s) from `tasks.yaml`
- Resolution: `REVISE: task-N.M - <exact instruction>` or `new task` (see *Remediation Tasks*), or `waiver candidate`
  with the decision the user must make

# Spec Reconciliation

Converge is the only step allowed to touch `spec.md`, `criteria.md`, `rules.md` after execution starts, and only in
these ways:

- **Δ (accepted as-built)** — for each DRIFT the user accepts (or that is unambiguously within what the spec left open),
  add an inline `*As built:*` note next to the affected B-N / RULE-N / AC and list it under a section
  `## As-built convergence (cp-N)` at the end of `spec.md`. Never rewrite or delete the original text; never renumber.
- **F-n (open)** — CRITICAL and GAP findings are listed in the same section as **not accepted**, with the finding id and
  the checkpoint they came from. They are removed only when a later convergence verifies the fix, or when the user
  records an explicit waiver in `status.md` → *Deviations*.
- **Spec weaknesses** — pointers instead of copied tables, undefined loaded words, missing state table, URL matrix that
  omits routes: list them under `### Spec weaknesses exposed by cp-N` with the upstream skill that should fix them
  (`spec`, `criteria`, `rules`). Do not fix them yourself in the middle of execution unless the user asks; they feed the
  next proposal/spec revision.
- **Plan weaknesses** — a checkpoint criterion with no artifact behind it, a route with no owning task, an artifact
  named by glob or prose, a `validation` bullet that was not literally satisfiable, an AC the phase claims that no task
  asserts, a task over 5 ACs or an unsplit `L` with no intermediate checkpoint: list them under
  `### Plan weaknesses exposed by cp-N` with `tasks` / `review (plan)` as the owner. Distinguish them from executor
  findings in the report: when the plan gave the executor two readings or no artifact, the finding is still a finding,
  but the fix is a plan fix and the `REVISE:` line says so.
- `criteria.md`: header note only ("Converged at cp-N …") plus verification-placement notes. ACs are never altered.
- `status.md`: set `phase-N: PENDING` (REJECT / PENDING WALKTHROUGH) or leave for the executor to set `APPROVED`; add
  waivers under *Deviations* with the finding id and the user's decision.

# Remediation Tasks

When a finding cannot be resolved by revising an existing task (new artifact, cross-cutting fix, more than two tasks
involved), append a remediation phase to `spec/tasks.yaml` following the `tasks` skill schema:

- `name: "Checkpoint N Remediation"`, `entry_criteria` naming the convergence report
- one task per finding cluster, `source: convergence/cp-N/<finding ids>`, `depends_on` the original tasks
- `validation` states the **assertion shape** that closes the gap (by value, `never()`, HTTP level, isolated DB), not
  "run tests"
- a terminal checkpoint whose `criteria` restate each finding as a verifiable sentence

Otherwise emit `REVISE: task-N.M - <instruction>` lines. Order them by dependency (data → domain → web → tests) and
send one at a time; say so in the report.

# Interactive Resolution

Ask the user only when a finding's fix requires a product decision the spec does not make (keep or drop an extra
account, which of two readings of a loaded word, waive or fix a DRIFT with cost). Offer the options with impact, recommend
one, and record the answer in the report and in `status.md` *Deviations*. Never ask about things you can verify.

# Anti-patterns (each one let a defect through at cp-5)

- Accepting "tests: N/N passing" as verification of any AC
- Count assertions on normative data; hashes never verified
- Mocks returning `null`/values the production method cannot return
- `@SpringBootTest` described as a "focused" web test; service calls described as "end-to-end"
- Negative security tests asserting only the status code
- Security tests scoped to the feature's URL prefix when the AC says "another owner's data"
- Guards present in one service (staff) taken as evidence for another (owner)
- "Full", "complete", "coherent" satisfied by the smallest reading
- A string-scan test that reads element text cited as proof of "no hard-coded strings"
- Tests that mutate a tracked runtime database, noticed only via a dirty working tree
- An "end-to-end" test that traverses a path no use case describes, or that omits an extension the spec marks as
  state-changing
- A walkthrough script written by converge instead of taken from the use cases
- Auditing the executor's chat instead of a committed `spec/checkpoints/cp-N.md`; the claims then cannot be diffed
  against the ledger after the fact
- Grading evidence for a phase delivered as two commits for seven tasks, as if the per-task loop had run
- Accepting a renamed test class as the declared artifact without reading what the rename dropped
- Treating "worked around" in `status.md` as a note rather than as the Blocker that was never raised
- Trusting the executor's Runtime Evidence table without requesting the URLs yourself

# Success Criteria

Complete only when ALL hold:

- Protocol Gate executed first: committed report present, one commit per task verified from `git log`, plan review
  present, artifacts by exact name, closure-gate evidence per task, Blocker/workaround scan, per-commit scope
- Grounding executed: before/after `git status`, full test run with counts (plan guards named), artifacts located, full
  route list built and diffed against the phase `routes`, runtime walkthrough URLs requested and statuses recorded
- Phase mode: all ten categories run; task mode: the Protocol Gate and categories 1, 2, 6, 10 over the tasks in scope;
  each either confirms pass with the evidence used or lists findings
- Evidence Ledger has one row for every AC in the phase's coverage; every non-STRONG row has a finding
- Every finding has all fields; every quoted spec text matches the file verbatim; every evidence pointer is file:line
- Verdict matches severity counts and the walkthrough rule
- Δ items folded into `spec.md` (and `rules.md`/`criteria.md` where affected) with `*As built:*` notes; F-n listed as
  not accepted; carry-forwards from earlier checkpoints resolved or restated
- REJECT reports end with ordered `REVISE:` directives and/or a remediation phase written to `tasks.yaml`
- The report ends with exactly one approval response line the user can paste to the executor

Do not write a partial report.

# Output

Write to `spec/convergence/cp-N.md` (create the folder if missing); task mode writes `cp-N.M.md` or `task-N.M.md` with
the Summary, Protocol Gate, Evidence Ledger, Findings, Category Notes (1, 2, 6, 10) and Approval response sections only.
One file per checkpoint; re-running converge on the same checkpoint overwrites it and notes the previous verdict.

```markdown
# Convergence: cp-N — <phase name>

## Summary
- Checkpoint: cp-N (<phase-N>, <n>/<n> tasks claimed complete); executor report: spec/checkpoints/cp-N.md @ <commit>
- Verdict: <APPROVE | APPROVE PENDING WALKTHROUGH | REJECT | REFUSED (protocol)>
- Counts: <N critical, N gaps, N protocol, N drift, N cosmetic>; carried forward: <ids or none>
- Suite (run by converge): <run>/<failed>/<errors>/<skipped> — <skipped names + reason>; plan guards: <PASS | FAIL | absent>
- Working tree after test run: <clean | files modified>

## Protocol Gate
| Task | Commit | Files in commit match artifact + supporting | Artifacts by exact name | Gate evidence in status.md | Blocker due / raised |
|---|---|---|---|---|---|

## Runtime Reproduction
| Actor | URL | Executor reported | Converge observed |
|---|---|---|---|

## Evidence Ledger
(lifecycle rows cite the UC steps traversed, e.g. `UC-1 1–5, 4a`)
| AC | Pattern | Claimed in | Evidence (file:line) | Strength | Verified |
|---|---|---|---|---|---|

## Findings
### Critical
<C-N … (empty if none)>
### Gaps
<G-N …>
### Protocol
<P-N …>
### Drift (Δ candidates)
<D-N …>
### Cosmetic
<K-N …>

## Category Notes
<one line per category 1–10: pass with evidence, or finding ids>

## Walkthrough Script (UI phases only)
<UC-n: actor → steps → must see / extensions to try / must be denied>

## Spec Reconciliation
- Δ folded in: <B-N/RULE-N/AC-N … with note text>
- F-n open: <ids>
- Spec weaknesses exposed: <item → upstream skill>
- Plan weaknesses exposed: <item → tasks / review (plan)>

## Resolution
<ordered REVISE directives, or "Remediation phase phase-N appended to tasks.yaml", or none>

## Approval response
<exactly one line: APPROVED | APPROVED WITH NOTES: … | REVISE: task-N.M - … | BLOCKED: …>
```
