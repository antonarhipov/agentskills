---
name: tasks
description: Generate an implementation task list from validated spec artifacts
---

# Task List Generator Skill

Translate a validated spec into an ordered, atomic, AC-traceable execution list an implementing agent can run task by task.

Pipeline position: proposal → spec → criteria → rules → review → **tasks** → review (plan) → execute ⇄ converge

# Role

You translate a validated spec into a task list written to disk. You do not write code, run tests, or modify project files outside `spec/tasks.yaml`. You do not ask questions; document judgment calls in `decisions` for the user to review.

The plan you write is itself reviewed (`spec-review` in plan mode writes `spec/plan-review.md`) before `execute` may
start, and it is executed by a model that does **exactly what the artifact map says and nothing more**. Every hole in
the artifact map becomes missing code; every ambiguity in `validation` becomes the weakest test that passes. Write the
plan for that executor: exact paths, exact routes, exact test names, small tasks.

# Pipeline Contract

Read `spec/review.md` (if exists) first. Locate the verdict line under `## Summary`.

- **FAIL**: refuse. Print the blocker IDs and recommend rerunning the relevant upstream skill. Do not write `spec/tasks.yaml`.
- **PASS WITH CONDITIONS**: each major must be reflected in the task list, either as a dedicated task with `source: review/MAJOR-N` or a `risk` annotation on an existing task. Note in `assumptions`.
- **PASS**: proceed.

Risk Hotspots from the review surface as `risk` annotations on the relevant task, regardless of verdict.

# Inputs

- Proposal: @file:spec/proposal.md
- Spec: @file:spec/spec.md (including "State model", "Use cases", "Normative data", "Presentation and navigation", "Verification expectations")
- Criteria: @file:spec/criteria.md (the AC list every task's `covers.acs` draws from)
- Rules: @file:spec/rules.md (optional; its Security Surface and Testing Strategy tables shape the verification tasks)
- Review: @file:spec/review.md (optional, pipeline gate)
- Project conventions: `CLAUDE.md` / `AGENTS.md` / `GEMINI.md`, build files, source tree

Spec takes precedence over the proposal. Tasks never reference a table, state model or matrix by pointer to another
file; when a task's `description` needs the rows, transitions or routes, it lists them (or names the exact spec
section heading and row count the executor must reproduce).

# Codebase Grounding (run first)

Read agent guidance files. Note package layout, module boundaries, naming, build/test/deployment patterns, and architectural style (layered, hexagonal, feature-sliced). Tasks place artifacts in paths consistent with the existing structure.

Architecture governs **where artifacts go**; it does not govern **phase order**. Phase order is decided by the
selection ladder in *Phase Organization*. A layered codebase is not by itself a reason to phase data → domain →
application → presentation; that is the fallback rung, reached only when the stronger signals do not fire.

While grounding, also record the **seams the feature's happy path crosses that have never run end to end** in this
codebase (e.g. no login exists yet, no page for this actor, no call to the external model or solver, no persisted
entity of this kind). This list is the signal for rung 3 of the ladder and is quoted in `dec-1`.

# Phase Organization

Set `organizing_principle` using the selection ladder below. Stop at the first rung that fires. Record the choice, the
rung that fired, and the signal that triggered it as `dec-1` (see *Recording the choice*).

## Principles

- **walking_skeleton**: thin end-to-end slice first, then thicken. Also called the *Tracer Bullet* approach. Best when
  nothing in scope runs end to end yet.
- **layered**: data → domain → application → presentation. Best when extending an existing layered structure that
  already runs end to end.
- **feature_slice**: one phase per AC cluster, each independently demoable. Best when the ACs partition cleanly.
- **risk_first**: highest-risk decision first, as a spike phase. Best when a wrong call invalidates later work. Never
  stands alone — see rung 2.

## Selection Ladder

1. **Explicit mandate.** If `rules.md` or the invocation specifies a principle, use it. Note the source in `dec-1`. Do
   not second-guess it, even if a lower rung would fire.
2. **Architectural risk.** If `review.md` lists Risk Hotspots that are architectural rather than local — an unproven
   external dependency, a contract that may not hold, a performance or concurrency assumption load-bearing for the
   design — the plan **starts with** `risk_first`: one spike phase whose checkpoint criterion is "the risky assumption
   is proven, or the design is changed and the tasks re-derived". Then continue down rungs 3–5 for the remaining work
   and record the hybrid `risk_first then <result>`. Local hotspots (a tricky parsing edge case, a fiddly migration) do
   not qualify; they stay as task `risk` annotations.
3. **No existing skeleton.** If nothing within the spec's scope currently runs end to end — new service, new module,
   new integration boundary, a new actor with no login or page, or the spec's happy path crosses a seam that has never
   been exercised (see the seam list from Codebase Grounding) — use `walking_skeleton`. The skeleton phase is
   **phase-1**. The skeleton is the **primary use case's main success scenario, thin** — every step present, no
   extensions yet — plus the mandatory tasks. It MUST contain, thinly but really:
   - login for each role the feature introduces, with the security surface for the whole URL→role matrix;
   - **the HTTP surface**: a controller artifact for every `GET`/`POST` the primary UC's main scenario and the `cp-1`
     walkthrough touch, listed in the phase's `routes` inventory with an owning task (the *HTTP surface* mandatory
     task). Templates and security rules without controllers do not walk;
   - at least one page per role rendered inside the existing layout with that role's menu entries, signed-in state and
     logout (the *Presentation and navigation* mandatory task);
   - one persisted entity of the feature, created through its migration and read back;
   - one round trip across **every** external seam the happy path crosses, through the production interface with its
     test double (AI interpreter, solver, message broker, …);
   - the isolated test datasource and the clean-working-tree check (the *Test environment* mandatory task), and the
     *Plan guards* mandatory task;
   - the primary UC's HTTP-level end-to-end test as the phase's `skeleton_test`, written **first** and red until the
     last task (see *Outside-in Skeleton Ordering*);
   - the human walkthrough criterion at `cp-1`.
   A skeleton that is a service with a unit test is not a skeleton. A skeleton whose e2e test is the last task is a
   skeleton the executor will fill with domain code and close with whatever test passes. After rung 3 fires,
   **re-apply rungs 4–5 to the remaining ACs** to order the thickening phases and record the hybrid
   `walking_skeleton then <result>`.
4. **Separable ACs.** If the ACs partition into two or more clusters that could each ship on their own without the
   others, use `feature_slice`. One cluster is not a partition. Use cases are the natural clusters: a UC whose
   postcondition is reachable without the others is a slice. Order slices by the primary UC's extensions first (they
   thicken the skeleton), then secondary UCs.
5. **Fallback.** Mirror the existing architecture. For most codebases this is `layered`; use the structure you found
   during Codebase Grounding.

Rungs are ordered by signal strength: an explicit instruction beats an observable artifact, an observable artifact
beats a structural inference, and the fallback follows code that already exists. If a rung's signal is ambiguous,
treat it as not fired and continue down the ladder.

## Executor modifier

If the invocation names the executing model and it is a low-reasoning tier (fast/flash-class models), a **pure**
`layered` result is forbidden: rung 5 yields `walking_skeleton then layered` instead. Such executors work task by task
and do not infer conventions they have not yet touched; a working, convention-correct slice in phase-1 is what every
later task imitates, and it lets converge catch UI, security and localization drift at `cp-1` instead of at the last
checkpoint. This is a modifier on the fallback, not a rung; it never overrides rungs 1–4.

## Hybrids

Phasing is often hybrid in practice — a spike, then a skeleton, then slices. When the plan mixes principles, set
`organizing_principle` to the sequence, e.g. `"walking_skeleton then feature_slice"`, and name the phase where the
handoff occurs in `dec-1`. Do not distort phasing to fit a single label. Do not chain more than two principles; if you
need three, the feature is too large and you should recommend a split.

## Recording the choice

`dec-1` is always the phasing decision and carries three extra fields so converge can audit it:

- `principle` — the value of `organizing_principle`;
- `rung` — the rung number that fired (and, for hybrids, the rung that ordered the remainder);
- `signal` — a quote of the artifact that fired it: the RULE id or invocation text (rung 1), the Risk Hotspot title
  (rung 2), the list of seams that have never run (rung 3), the AC clusters (rung 4), or the architectural style found
  (rung 5). "PetClinic is layered" is a rung-5 signal and is only valid if rungs 2–4 were checked and did not fire.

## Demo sentence per phase

Every `phase.description` ends with a **demo sentence**: what a human can *do* against the running application at the
end of the phase ("an owner can log in, submit a request, and see one held suggestion in *My appointments*"). If no
truthful demo sentence exists for a phase — only "tests pass" — the phasing is wrong for that phase; revisit the
ladder. For a low-reasoning executor the demo sentence is also the clearest statement of the phase goal.

The demo sentence is the **postcondition of the use case(s) the phase completes**, phrased for a human
(`UC-1 postcondition: request Accepted, one Scheduled appointment under My appointments`). A phase that completes no
UC and no extension has no demo sentence — revisit the ladder.

# Task Granularity

- Completable in a single focused effort (rule of thumb: under an hour)
- Produces a verifiable artifact (file, passing test, documented decision)
- Small enough to roll back cleanly — and to commit alone: the executor makes one commit per task
- References ACs and RULES it covers via `covers`
- **At most 5 ACs per task.** Six or more means the task is really "write the tests for the phase" or "build the
  domain"; split it along the ACs' `Flow:` tags or along artifacts.
- **Complexity `L` is a smell, not a size.** Split an `L` task into `M`/`S` tasks with their own artifacts. If a task
  genuinely cannot be split (a single migration, a single security configuration), keep it `L` **and** follow it with an
  intermediate checkpoint `cp-N.M` so converge looks at it before the next task builds on it. Never more than one
  unsplit `L` task between two checkpoints in phase-1.
- **No single-line prose.** `description` and `validation` are YAML block scalars (`|`) or lists; a description that
  runs past ~600 characters on one line is unreadable to the tools the executor uses and will be skimmed. Use short
  paragraphs and bullets: what to build, which spec rows/steps to reproduce, what must not be touched.

# Artifact Exactness

`artifact` is the executor's definition of done and the first thing converge checks. It is a **YAML list of exact
repository-relative paths**, one per line. Rules:

- No globs (`templates/my/*.html`), no ellipses (`…`), no prose ("ownership guard service"), no "and friends". Name
  every file: every controller, every template, every entity, every repository, every configuration class, every
  message-bundle file the task adds keys to, every migration script.
- Test artifacts name the **class and the methods**, so a rename or a drop is visible:
  `src/test/java/org/example/app/security/SecurityMatrixWebTests.java (methods: anonymous_every_route_redirects_to_login, wrongRole_every_route_403_and_never_service, wrongOwner_every_owner_route_403_no_body_disclosure)`.
- A task whose `description` mentions a class, page, route or key that is not in its `artifact` (and not in an
  earlier task's) has a hole; add the path or move the sentence.
- The union of all `artifact` lists in a phase must be **sufficient to perform the phase's checkpoint criteria**:
  walk each criterion and each walkthrough URL and point at the artifact that renders it, guards it and localizes it.
  A criterion with no artifact behind it is a plan defect, not an executor problem.
- The executor may not rename an artifact. If you are unsure of a name, decide now and record the decision; do not
  leave it to the executor.

# Route Inventory per Phase

Every phase that adds or changes an HTTP route carries a `routes` list: `"<METHOD> <path> → task-N.M"`, one entry per
request mapping the phase introduces (form `GET` and its `POST` are two entries). Rules:

- Every URL named in the phase's `checkpoint.criteria`, its walkthrough script, or its use-case steps appears in
  `routes` with an owning task whose `artifact` contains the controller.
- Every route in `routes` appears in the rules' URL→role matrix (or the plan records the omission as an assumption for
  the `rules` skill to fix).
- A skeleton phase has a dedicated **HTTP surface** task (controllers + the security matrix entries for those routes)
  rather than smuggling controllers into template or security tasks.
- The plan-guard `RouteInventoryTest` asserts, for every task marked complete, that its routes resolve to a handler.

# Validation Shape

`validation` is the checklist the executor fulfils literally and converge re-executes, so it states the **assertion
shape**, not the activity. "Run the tests" or "tests pass" is never a valid `validation`. It is a **YAML list**, one
bullet per assertion, each of the form:

```
TestClass.method → AC-n → <what is asserted, at which level, with which strength>
```

for example `SeedMigrationTests.clinicHours_allSevenRows_byValue → AC-12 → every row of the clinic-hours table
compared field by field on a fresh H2 database; no count assertions`. Prose paragraphs are paraphrased by the executor
into whatever passes; a bullet with a class, a method, an AC and a shape survives. Derive the shape from the AC pattern
(see the rules' Testing Strategy table):

| AC pattern of the task's `covers.acs` | `validation` must say |
|---|---|
| Data exactness (seed, accounts, defaults) | which test asserts **every row by value** on a fresh database and verifies every credential with the encoder; "row count" is not acceptable |
| State transition / refusal | which service-level test sets up each disallowed state and asserts the named refusal **and** `never()` on collaborators |
| Negative authz | which web-slice test, over **which routes** (the whole matrix, pre-existing routes included), asserts status **and** no disclosure **and** no mutation, for anonymous, wrong-role and wrong-owner |
| Fidelity | which round-trip test compares every enumerated field |
| Lifecycle / end-to-end | which HTTP-level test drives **UC-n steps k…m** as each named actor on an isolated in-memory database, asserting state after every step; which extension legs it includes |
| Presentation / localization | which scan or test covers templates (text **and** attributes **and** inline expressions) and code-produced messages; plus the walkthrough criterion at the checkpoint |
| Boundary | the three values asserted |

Every task that ships a test double states in `description` that the double conforms to the production interface
(same exceptions, nullability, `Optional` semantics). Every task that ships tests names the isolated datasource.

Every AC in a task's `covers.acs` appears in at least one of its `validation` bullets; every test class named in
`validation` appears in `artifact`. Under a `skeleton_test`, every intermediate task's `validation` also carries one
bullet `skeleton_test → reaches UC-n step k (<what the step does>)` so the executor knows what "advanced" means.

# Mandatory Tasks

Whenever the spec contains the corresponding section, the task list contains a dedicated task (not a bullet inside
another task) for:

- **Normative data** — the migration/fixture *and* its by-value test, in the same task, so the test is written against
  the table rather than against the migration
- **State model** — the service-level guards for every action, *and* the refusal tests, before any controller that
  calls those actions is built
- **Security surface** — the security configuration for the **whole** URL→role matrix (pre-existing routes
  included) and the negative matrix test over it, as one task in the phase that introduces authentication
- **Presentation and navigation** — layout/menu/login/logout/landing pages, per role, as a task in the first UI phase,
  not as leftovers of the last one
- **Test environment** — the isolated test datasource and the "working tree unchanged after the suite" check, in the
  first phase that adds a test
- **Use cases** — for every UC, one HTTP-level end-to-end test task (main scenario + state-changing extensions) in the
  phase that completes the UC; the task `description` copies the step list from `spec.md` and names the actors
- **HTTP surface** — in every phase with a `routes` inventory, the controllers (and their security-matrix entries) for
  those routes as an explicit task, with each controller class in `artifact`; in a skeleton phase this task precedes the
  template and ownership-guard tasks that assume the routes exist
- **Plan guards** — in the first phase that adds a test, alongside *Test environment*, three small tests (~100 lines
  in total) that make the plan itself executable and are re-run at every task closure and every checkpoint:
  - `RouteInventoryTest`: for every task marked `COMPLETE` in `status.md`, every `METHOD path` assigned to it in the
    phase `routes` resolves to a handler (query the request-mapping registry); and every route in the rules' URL→role
    matrix that belongs to a completed task is mapped
  - `ArtifactInventoryTest`: reads `spec/tasks.yaml` and `spec/status.md`; for every task marked `COMPLETE`, every path
    in its `artifact` list exists
  - `AcTagCoverageTest`: for every task marked `COMPLETE`, every `AC-n` in its `covers.acs` appears as a `@Tag`,
    method name or `@DisplayName` in at least one test under `src/test`
  Their `artifact` lists the three classes; their `validation` states that each fails on a deliberately broken
  fixture (a missing path, an unmapped route, an untagged AC) and passes on the current tree. They are the difference
  between "REJECT after 6,500 lines" and "red build after 400".

Under a `walking_skeleton` plan all of the first group, the HTTP surface, the plan guards and the primary UC's
end-to-end test (as `skeleton_test`) land in phase-1 by construction (see rung 3); under any other principle each lands
in the phase named above, never later.

# Outside-in Skeleton Ordering

When rung 3 fires, order phase-1 **from the outside in** so the executor cannot spend the phase on domain code:

1. *Test environment* + *Plan guards* (the suite can run, the guards are red-capable).
2. The `skeleton_test`: the primary UC's HTTP-level end-to-end test (MockMvc or `MockMvcTester` over the real
   `SecurityFilterChain` with CSRF, as each named actor, on the isolated datasource, asserting state after every step).
   It is committed **red**; the phase declares it under `skeleton_test` so `execute` and `converge` know its failure is
   expected until the last task.
3. *Security surface* + *HTTP surface* (login works, every route in `routes` returns something other than 404).
4. *Normative data* and the one persisted entity.
5. *Presentation and navigation* (pages render inside the layout for every role).
6. The external-seam round trips (interpreter, solver, …) through their production interfaces with doubles.
7. The task that turns the `skeleton_test` green and adds the state-changing extension legs.

Every task from 3 onward names in `validation` the UC step the `skeleton_test` must reach after it. A task after which
the skeleton test does not advance is either misplaced or building something the skeleton does not need.

# Re-planning After Partial Execution

Sometimes the plan is regenerated after one or more phases have already been executed and converged (typically because
this skill's contract changed, or because a checkpoint exposed plan defects that the remaining phases share). In that
case the invocation says so ("phase-1 is as-built …") and you run in **re-plan mode**. Additional inputs:

- Progress: @file:spec/status.md (which phases/tasks are `COMPLETE`, which checkpoints are approved, the *Deviations*)
- The existing plan: `spec/tasks.yaml` (the ids and `covers` the committed work and the convergence reports refer to)
- Convergence reports: `spec/convergence/cp-*.md` (open `F-n` findings, accepted `Δ` deltas, plan weaknesses)
- The working tree (`git ls-files`, `git log`), which is the ground truth for what an executed phase produced

Rules for re-plan mode:

1. **Completed phases are records, not tasks.** A phase whose checkpoint is `APPROVED` (or whose closure is recorded in
   `status.md` under a user-approved waiver) keeps its `id`, its task ids, its task order, and its `checkpoint.id`.
   Never add, remove, split or reorder its tasks, and never rewrite its `description` into new work: a regenerated task
   for code that already exists invites the executor to "run the loop" against existing files, which is
   self-certification by another name. Mark the phase `status: as_built` and give each of its tasks `status: COMPLETE`
   with the commit that landed it (from `git log`).
2. **As-built artifact lists are exact and taken from the tree.** Replace each completed task's `artifact` with the
   list of paths that **now exist** and belong to that task (`git ls-files` + the commits in `status.md`), one per
   line, test classes with their methods — so the `ArtifactInventoryTest` plan guard can check them retroactively. A
   path that the original plan named but that does not exist is dropped (with a `decisions` entry naming the rename
   recorded in *Deviations*); a path that exists but no task owns is assigned to the task whose description it serves.
3. **Keep `covers` truthful.** A completed task's `covers.acs` becomes the set of ACs whose STRONG evidence converge
   located in that task's artifacts (the Evidence Ledger of the last `cp-N.md`). ACs the ledger graded WEAK / ABSENT /
   MISPLACED and that were not fixed are removed from the completed phase and re-planned in the first new phase (see
   rule 6). `phase.covers` remains the union of the tasks' `covers.acs`. Do not leave an AC claimed twice.
4. **Add `routes` retroactively.** Enumerate the request mappings the completed phase introduced (from the controllers
   in its as-built artifacts) and record them as `"<METHOD> <path> → task-N.M"` so the `RouteInventoryTest` guard has
   something to check for the completed phase too.
5. **Plan guards move to the first new phase.** If the completed phases did not ship the *Plan guards* mandatory task,
   it becomes the **first task** of the first new phase (`task-M.1`), and its `validation` states that the three guards
   pass over the as-built phases as well ("`ArtifactInventoryTest` passes for every task marked `COMPLETE`, including
   phase-1's as-built artifacts"). The same applies to the *Test environment* task if it is missing.
6. **Remaining phases are regenerated in full** under the current contract — exact `artifact` lists, `routes`, ≤ 5 ACs
   per task, `TestClass.method → AC-n → shape` validation bullets, no unsplit `L` without an intermediate checkpoint —
   from the spec inputs, not by patching the old tasks. Open `F-n` findings from the last convergence that were waived
   (not fixed) become tasks with `source: converge/cp-N/F-n`; accepted `Δ` deltas are treated as spec. ACs removed from
   a completed phase by rule 3 land in the first new phase that touches their area.
7. **Ids stay stable.** New phases continue the numbering (`phase-2`, `phase-3`, …) and their task ids start at
   `task-M.1` even when the old plan's phase-M looked different; `status.md`, the convergence reports and the commit
   messages key on the completed ids only, so the new ones are free.
8. **Record the re-plan.** `dec-1` keeps the phasing decision; add `dec-2` (`decision: "re-plan after phase-N
   as-built"`) naming the completed phases, the waiver or approval they close under, the ACs moved out of them, and the
   plan-guard relocation. Set top-level `replanned_after: cp-N`.

Everything else in this skill applies unchanged to the regenerated phases, with one addition: a new task that changes a
file an as-built task owns lists that path under `modifies` (exact paths, same rules as `artifact`), not under
`artifact`; `artifact` names only the files the task creates. The verification pass before writing adds: every
completed task's `artifact` path exists in the working tree (`ls`), and no new task's `artifact` names a path a
completed task already owns (it belongs in `modifies`).

# Dependency Rules

- No circular dependencies
- Minimize cross-phase dependencies
- Infrastructure before business logic; interfaces before implementations; fixtures before tests

# Checkpoint Patterns

Place checkpoints where human review meaningfully reduces risk:
- After project structure or scaffolding
- After the first end-to-end slice runs
- After core domain logic is in place
- After each major integration boundary
- After test suite green for a phase's ACs
- Before any irreversible step (migrations, deletions, API contract changes)

Every phase ends with a checkpoint. Intermediate checkpoints allowed within a phase, and **required** after any
unsplit `L` task in phase-1 and after the *HTTP surface* task of a skeleton phase (that is where "no controllers" is
caught at one task's cost instead of seven). An intermediate checkpoint `cp-N.M` is converged in task mode: categories
1, 2, 6 and 10 only, over the tasks since the previous checkpoint.

Checkpoints are verified by the `converge` skill, so write `checkpoint.criteria` as sentences converge can check, not
as goals: name the observable outcome and the evidence shape ("migration test asserts all 7 clinic-hour rows by value",
"anonymous GET on every route in the security matrix redirects to `/login` and no service is invoked"), never "tests
pass" or "UI works".

Every URL a criterion names must be in the phase's `routes`; every artifact a criterion needs must be in some task's
`artifact`. Every terminal checkpoint of a phase with routes also carries a **runtime evidence** criterion: "the
executor's checkpoint report in `spec/checkpoints/cp-N.md` records, for each actor, login and each walkthrough URL with
the HTTP status observed against the running application".

Any phase that ships templates gets one **human walkthrough** criterion: "signed in as <actor>, execute **UC-n main
scenario by hand** (and extensions <list>); pages render inside the existing layout with only that role's menu entries,
the signed-in user and a logout action are visible, and <role> is denied on <URLs>". A walkthrough is a use case
executed by a human; the script is not invented per phase. Automated tests cannot judge alignment or coherence; the
walkthrough is the acceptance gate for UI, and converge cannot approve the phase until the user confirms it.

Every terminal checkpoint also carries: "full suite green; `git status` shows no tracked file modified by the run".

# Stable IDs

- Phases: `phase-1`, `phase-2`, ... in execution order
- Tasks: `task-N.M` (phase number, task number)
- Checkpoints: `cp-N` (terminal) or `cp-N.M` (intermediate)
- Decisions: `dec-1`, `dec-2`, ...

# Soft Limits

Aim for ≤ 5 phases, ≤ 7 tasks per phase. If you exceed:
- The feature is probably too large. Recommend a split in `decisions` rather than padding.
- If a split isn't sensible, exceed the limit and note the reason in `decisions`.

Don't pad or merge to fit the numbers. In particular, never merge tasks to stay under 7: the 5-AC cap, the `L`-split
rule and the mandatory tasks (a skeleton phase-1 has at least eight) take precedence over the task count. A skeleton
phase-1 of 9–11 small tasks with two intermediate checkpoints is the expected shape, not a violation; note it in
`decisions` once.

# Coverage

Every AC in `criteria.md` appears in some task's `covers.acs`, OR in `coverage_deferrals` with a reason. No third option.

Coverage is per assertion, not per mention. A task whose `covers.acs` lists a refusal AC, a negative authz AC or a
data-exactness AC must have a `validation` of the matching shape (see *Validation Shape*); otherwise the AC is not
covered and the task must be split or its `validation` sharpened. A single task covering more than five ACs violates
*Task Granularity*: split it.

**Phase coverage equals task coverage.** `phase.covers` is exactly the union of its tasks' `covers.acs` — no AC that no
task in the phase asserts, and no task description that says an AC "is completed in phase-M" while the phase claims it.
An AC whose lifecycle spans phases is listed in the phase where its full assertion lands and nowhere earlier; a partial
leg in an earlier phase is described in that task's `description` without listing the AC. Two readings of what a phase
owes is a licence for the lenient one.

# Output Schema

```yaml
tasks:
  feature: "<name>"
  review_verdict: "<PASS | PASS WITH CONDITIONS>"
  organizing_principle: "<walking_skeleton | layered | feature_slice | risk_first>, or \"<A> then <B>\" for a hybrid"
  assumptions:
    - "<assumption to verify>"
  decisions:
    - id: dec-1                     # always the phasing decision
      decision: "<organizing_principle chosen>"
      principle: "<same value as organizing_principle>"
      rung: "<1-5; for hybrids: '3, then 4 for phases 2+'>"
      signal: "<quoted artifact: RULE id / hotspot title / seams never run / AC clusters / architecture found>"
      reason: "<why>"
      alternatives: ["<alt 1>", "<alt 2>"]
    - id: dec-2
      decision: "<judgment call>"
      reason: "<why>"
      alternatives: ["<alt 1>", "<alt 2>"]
  coverage_deferrals:
    - ac: AC-12
      reason: "<why not in a task>"
  phases:
    - id: phase-1
      name: "<phase name>"
      description: |
        <what this accomplishes>.
        Demo: <what a human can do against the running app at the end of this phase>
      covers: [AC-1, AC-2]                # exactly the union of the tasks' covers.acs
      entry_criteria: "<what must be true to start>"
      routes:                             # every request mapping this phase introduces, with its owning task
        - "GET /my/appointments → task-1.3"
        - "POST /my/requests → task-1.3"
      skeleton_test: "src/test/java/<pkg>/SchedulingUc1SkeletonTests.java"   # walking_skeleton phase-1 only; red until the last task
      tasks:
        - id: task-1.1
          name: "<task name>"
          description: |
            <what to do, in short paragraphs and bullets: what to build, which spec rows / UC steps to reproduce,
            what must not be touched>
          artifact:                       # exact repository-relative paths, one per line; no globs, no prose (<pkg> is a placeholder for this schema only)
            - "src/main/java/<pkg>/FooController.java"
            - "src/main/resources/templates/my/foo.html"
            - "src/main/resources/messages/messages.properties"
            - "src/test/java/<pkg>/FooWebTests.java (methods: ac1_anonymous_redirects_to_login, ac2_wrong_owner_403_no_disclosure)"
          covers:
            acs: [AC-1]                   # at most 5
            rules: [RULE-3, RULE-7]
          depends_on: []
          complexity: "S | M"             # L only when unsplittable, and then followed by an intermediate checkpoint
          validation:                     # one bullet per assertion: TestClass.method → AC-n → shape
            - "FooWebTests.ac1_anonymous_redirects_to_login → AC-1 → 302 to /login and never() on FooService, over the real SecurityFilterChain"
            - "skeleton_test → reaches UC-1 step 3 (owner sees the request form)"
          risk: "<from Risk Hotspots, if applicable>"
          source: "<review/MAJOR-N if addressing a review finding>"
      checkpoint:
        id: cp-1
        description: "<what to review>"
        criteria:
          - "<criterion 1>"
```

Intermediate checkpoints are expressed as a task-level `checkpoint:` block (`id: cp-N.M`, `description`, `criteria`)
on the task they follow.

**Required**: top-level `feature`, `review_verdict`, `organizing_principle`, `phases`, and `decisions` containing `dec-1` with `principle`, `rung`, `signal`; phase `id`, `name`, `description` (ending with a demo sentence), `covers`, `tasks`, `checkpoint`, and `routes` when the phase adds a request mapping; task `id`, `name`, `description`, `artifact` (list of exact paths), `covers`, `depends_on`, `validation` (list of bullets); checkpoint `id`, `description`, `criteria`.

**Optional**: `assumptions`, further `decisions`, `coverage_deferrals`, `entry_criteria`, `complexity`, `risk`, `source`, `skeleton_test` (required when rung 3 fired), task-level `checkpoint`.

**Re-plan mode only**: top-level `replanned_after: cp-N`; phase `status: as_built` on completed phases; task `status: COMPLETE` and `commit: <hash>` on their tasks; `dec-2` recording the re-plan; task `modifies` (exact paths of as-built files the task changes) on new tasks.

# Success Criteria

Complete only when ALL hold:

- Pipeline contract honored: FAIL refused; PASS WITH CONDITIONS reflected in tasks or risks
- Every AC in `criteria.md` in some task's `covers.acs` or in `coverage_deferrals`
- Every task has required fields
- Every `depends_on` references an earlier task in execution order
- No circular dependencies
- Every phase ends with a checkpoint whose criteria name an observable outcome and an evidence shape; UI phases have a walkthrough criterion; terminal checkpoints have the clean-working-tree criterion
- Every `validation` is a list of `TestClass.method → AC-n → shape` bullets matching the AC pattern (no "run tests"/"tests pass"); every AC in `covers.acs` appears in a bullet; every test class named appears in `artifact`
- Every `artifact` is a list of exact repository-relative paths (no globs, ellipses or prose); test artifacts name their methods
- Every phase that adds a request mapping has a `routes` inventory; every URL in its checkpoint criteria, walkthrough and UC steps is in `routes`; every route's owning task has the controller in `artifact`
- For every phase, the union of its tasks' `artifact` lists is sufficient to perform every checkpoint criterion (walk each criterion and point at the artifacts that render, guard and localize it)
- No task covers more than 5 ACs; no `L` task without a following intermediate checkpoint; no single-line `description` or `validation`
- `phase.covers` equals the union of the phase's tasks' `covers.acs`; no task description defers an AC the phase lists
- Mandatory tasks present for every spec section that triggers them (normative data, state model, security surface, HTTP surface, presentation and navigation, test environment, plan guards, use cases)
- If rung 3 fired, phase-1 declares `skeleton_test`, its task order follows *Outside-in Skeleton Ordering*, and every task from the HTTP-surface task onward names the UC step the skeleton test reaches
- No task references a table, state model or matrix by pointer to another file
- `organizing_principle` set by the selection ladder, with the rung and quoted signal in `dec-1`; hybrids name the handoff phase; a pure `layered` result is accompanied by evidence that rungs 2–4 did not fire
- If rung 3 fired, phase-1 contains every skeleton item listed under rung 3 and `cp-1` has the walkthrough criterion
- Every `phase.description` ends with a truthful demo sentence, which is the postcondition of the UC(s) the phase completes
- Every UC has an end-to-end task whose `validation` names its steps; walkthrough criteria name the UC executed; if rung 3 fired, phase-1 contains every step of the primary UC
- All Risk Hotspots reflected in task `risk` annotations; architectural hotspots produced a `risk_first` spike phase
- Soft limits met, or deviation justified in `decisions`
- In re-plan mode: completed phases keep their ids, task ids and order; their `artifact` lists name only paths that exist; their `covers` contain only ACs with STRONG evidence in the last convergence; plan guards are the first task of the first new phase if not already shipped; `dec-2` and `replanned_after` present

Verification pass before writing:
- Walk phases in order; every `depends_on` points to a task that has appeared
- AC IDs in `covers.acs` (across all tasks) equals AC IDs in `criteria.md` minus `coverage_deferrals`
- For each phase, `phase.covers` equals the union of its tasks' `covers.acs`
- Every RULE ID in `covers.rules` is real in `rules.md`
- For each phase with `routes`: every route has an owning task, that task's `artifact` contains a controller, and every URL in the checkpoint criteria is in `routes`
- For each checkpoint criterion: name the artifact(s) that satisfy it; a criterion with none is a hole — add the task
- No `artifact` entry contains `*`, `…`, `...` or a space-separated description instead of a path
- Every `validation` bullet parses as `<Class>.<method> → AC-n → <shape>` (or the `skeleton_test → reaches …` form)
- `feature`, `review_verdict`, `organizing_principle`, `dec-1` (`principle`, `rung`, `signal`), every phase checkpoint present

Do not write a partial file.

# Output

Write to `spec/tasks.yaml`. Then tell the user to run `spec-review` in plan mode (it writes `spec/plan-review.md`);
`execute` refuses to start without it.
