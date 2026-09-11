---
name: spec-review
description: Stress-test the spec pipeline outputs against each other and the codebase before implementation begins (spec mode), and gate the implementation plan in spec/tasks.yaml before execution starts (plan mode). Use plan mode whenever the user asks to review, check or validate the plan / tasks.yaml, or right after the tasks skill has written it.
---

# Spec Review Skill

Stress-test the spec pipeline against itself and the codebase. Surface anything that would cause an implementing agent to fail or build the wrong thing. Trust upstream's self-verification by default; investigate the seams.

Pipeline position: proposal → spec → criteria → rules → **review** → tasks → **review (plan)** → execute ⇄ converge

Review runs **twice**:

- **Spec mode** (default) — before `tasks`: categories 1–5 below over proposal / spec / criteria / rules. Output
  `spec/review.md`.
- **Plan mode** — after `tasks`, before `execute`: categories P1–P6 under *Plan Review* over `spec/tasks.yaml`. Output
  `spec/plan-review.md`. Choose this mode when `spec/tasks.yaml` exists and the user asks to review the plan, or when the
  `tasks` skill has just run. `execute` refuses to start without a non-FAIL `plan-review.md`.

A plan that was never reviewed is how a phase named "walking skeleton" ships without a single controller: the spec was
sound, the artifact map had a hole, and the executor built exactly what the map said.

# Role

You produce a report and, when needed, a Fix Plan that sequences the user's resolution work. You do not fix issues yourself. You do not ask the user questions. If findings need resolution, the user reruns the relevant upstream skill in the order the Fix Plan prescribes (in plan mode, that skill is `tasks`).

# Operating Principle

Each upstream step has its own Success Criteria and verification pass. Do not re-derive what upstream already verified within its own artifact. Review's job is the cross-document and cross-codebase layer that no single step can see:

- The chain from behaviors to criteria to rules holds end-to-end
- The composed design is consistent with itself and with the codebase
- The risks specific to this feature, even with a passing spec, are surfaced
- Any fixes are sequenced so the user can resolve them in pipeline order

If you find yourself rebuilding a coverage table or re-deriving a forward trace, you are doing upstream's work. Verify the existing structures instead.

# Pipeline Contract

FAIL blocks the plan step. PASS WITH CONDITIONS requires fixes via the relevant upstream skill before plan runs. The Fix Plan sequences the work; the user follows it top to bottom and reruns review once at the end.

# Inputs

- Proposal: @file:spec/proposal.md
- Spec: @file:spec/spec.md
- Criteria: @file:spec/criteria.md
- Rules: @file:spec/rules.md
- Project conventions: `CLAUDE.md` / `AGENTS.md` / `GEMINI.md`, build files, source layout, test setup

# Codebase Grounding (run first)

Before checking anything else, ground the review in the actual project state:

- Read the build file. Confirm declared libraries exist at the declared versions; confirm new libraries proposed in rules.md can actually be added (presence in the registry, no obvious license conflicts).
- Read the source layout. Confirm the package locations rules.md prescribes match the project structure or are creatable without breaking it.
- Read existing patterns for persistence, error handling, logging, testing. Confirm the Design section and rules align, or that deviations are justified in rules' `Reason:` lines.
- Read the test setup. Confirm the proposed test strategy is achievable with the project's existing test infrastructure. Note which datasource tests use today and whether a runtime data file is tracked in the repository.
- Read the security configuration and **enumerate every route in the codebase** (pre-existing controllers, static resources, login/logout, error pages). You will compare this list against the rules' URL→role matrix in category 2.
- **Verify every existence claim against `HEAD`.** For every route, file, class, property or library the spec, criteria or rules call "pre-existing", "stock", "existing", "already present" or "inherited", confirm it is in the working tree now (`git log --diff-filter=D` catches files deleted in an earlier commit; the framework's reference for the declared version catches renamed properties). A claim about something that no longer exists is a BLOCKER: the executor will either build against a ghost or quietly "fix" the reference.
- Read the presentation conventions (layout fragment, menu fragment, form-field fragments, stylesheet, message bundles) so you can tell whether rules.md restates them for the feature.

This grounding feeds the Codebase Grounding category later. Without it, that category can only check claims, not reality.

# Severity

- **BLOCKER**: implementation cannot proceed (broken trace chain, AC contradicted by a rule, declared library missing, codebase incompatible with a rule)
- **MAJOR**: implementation will produce wrong behavior or significant rework (unmeasurable non-functional threshold, missing test approach for an EARS pattern present in criteria, scope item reintroduced)
- **MINOR**: clarity or consistency issues that don't change the result (naming inconsistency, redundant rule, stale reference)

If uncertain BLOCKER vs MAJOR, treat as BLOCKER. If MAJOR vs MINOR, treat as MAJOR.

# Verdict

- **PASS**: zero blockers, zero majors (no Fix Plan)
- **PASS WITH CONDITIONS**: zero blockers, one or more majors (Fix Plan required)
- **FAIL**: one or more blockers (Fix Plan required)

Minors don't affect the verdict and don't appear in the Fix Plan.

# Review Categories

Run all five. Record findings inline per category, or confirm pass.

## 1. Discipline Check (thin verification)

Sample upstream's claimed structure; do not rebuild it.

- Every `Covers: B-N` reference in criteria.md points to a real B-N in spec.md
- Every `Covers: AC-N` reference in rules.md points to a real AC in criteria.md
- Every B-N in spec.md appears in at least one AC's `Covers:` line (no orphan behaviors)
- Every AC appears in rules.md Cross-Reference table or is explicitly marked "(none needed)" (no orphan criteria)
- The Cross-Reference table in rules.md is accurate: spot-check 3 entries against the actual rule contents
- Every AC matches one of the five EARS templates (or the Combined variant); no template-free criteria
- **No pointer references.** No B-N, AC or RULE says "matching the tables in …", "as defined in the proposal", "see …" in place of the content itself. Every normative table appears verbatim in spec.md and its rows appear in the exactness ACs. A pointer is a BLOCKER: nothing downstream can assert it.
- **State models are tables, not prose.** Every lifecycle entity has a state table in spec.md, one transition AC per allowed transition and one refusal AC per action in criteria.md, and a rule naming the service-level refusal.
- **Use cases are traced.** Every UC step and extension in spec.md cites a real B-N; every step and extension is named in ≥ 1 AC `Flow:` tag; every AC has a `Flow:` tag (`UC-n step k`, `UC-n ext ka` or `cross-cutting`). An extension with no B-N or no AC is MAJOR (an untested branch); a step that names a widget or endpoint is MINOR.

This category is fast. If it produces more than a handful of findings, upstream skipped its own verification pass and should be rerun before continuing review.

## 2. Cross-Document Conflicts

The centerpiece. No upstream step can detect these.

- **AC ↔ AC**: two ACs that cannot both hold
- **AC ↔ RULE**: a rule that prevents an AC's outcome
- **RULE ↔ RULE**: two rules that cannot both hold. Check boundary rules against framework-requirement rules in
  particular: "no `<library>` import outside package X" versus "the entity / solution / provider classes carry
  `<library>` annotations" cannot both hold unless one names the exempt package. Left unresolved, the executor resolves
  it silently (an exemption in the architecture test) instead of reporting it
- **Design ↔ Rules**: a rule that contradicts the Design section in rules.md (e.g., Design says "synchronous flow," a rule introduces async messaging)
- **Scope reintroduction**: anything in `Out of scope` (spec), `Coverage exclusions` (criteria), or `Design exclusions` (rules) that resurfaces elsewhere in the pipeline
- **Negative-decision violations**: any rule that proposes adopting an ecosystem option that another rule explicitly declined (e.g., a rule references Spring Batch APIs when another rule declined Spring Batch)
- **Authorization scope narrowing**: for every negative authz AC, confirm the covering rule enumerates the **entire** URL space of the application — pre-existing routes included — and that the AC itself is phrased over "any page or action that displays …", not over the feature's own objects. Compare the rules' URL→role matrix against the route list from grounding: every route absent from the matrix, and every `permitAll` on a route that displays owner-scoped data while an AC requires isolation, is a conflict (AC ↔ RULE, BLOCKER).
- **Loaded words left open**: any AC or rule that applies *full*, *complete*, *coherent*, *appropriate*, *all*, *minimal* to a screen, data set or test suite without a definition in spec or rules. The executor will pick the smallest reading; this is a MAJOR against the document that should define it.
- **Verification-shape drift**: an AC whose EARS pattern demands a level (HTTP end-to-end, whole-surface authz, by-value data) while the rules' Testing Strategy table permits a weaker level (service-level, feature-prefix, count-only) for that pattern.
- **State model ⇄ use cases**: a transition in the state table that no UC step or extension performs, or a state-changing step or extension that names no existing transition, is MAJOR against spec.md. The two models must agree in both directions.
- **Use case ⇄ scope**: a UC step or extension that requires an item listed under `Out of scope` is scope reintroduction. **Use case ⇄ authz**: a UC whose actor performs a step the URL→role matrix forbids is AC ↔ RULE (BLOCKER). **Use case ⇄ precondition**: a UC precondition that contradicts a state table or an out-of-scope item is MAJOR.

For each conflict, quote both sources verbatim. Conflicts are nearly always at least MAJOR; often BLOCKER.

## 3. Codebase Grounding

Use the grounding pass above to verify the composed design holds against reality.

- Every library named in rules.md exists in the project or can be added (registry presence, version compatibility)
- Every package and module path the design uses exists or is creatable without breaking existing structure
- Every pattern the rules invoke (transactional boundaries, error handling shape, persistence approach, observability hooks) is achievable with the declared frameworks at their declared versions
- The proposed test setup is compatible with existing test infrastructure (framework, runner, container support, fixture conventions), and the rules' test-data isolation statement is achievable (a separate test datasource exists or is specified; no test writes to a tracked runtime data file)
- Anything in the Design section that implies a code structure has a viable place to land in the current layout
- If the feature ships templates, rules.md restates the project's presentation conventions (layout fragment, menu fragment, form fragments, stylesheet, message bundles) for the feature — the executor cannot inherit a convention nobody named
- If the feature seeds data, the rules' normative source is the copied table in spec.md and the migration test is required to assert by value
- Every actor named in a UC has a way to authenticate in the codebase (a seeded account or a documented fixture) — otherwise the end-to-end test cannot be written as the rules specify

Findings here are usually BLOCKER (incompatible library, missing capability) or MAJOR (achievable but with non-trivial work not yet specified).

## 4. EARS Template ↔ Test Strategy Fit

For each EARS pattern present in criteria.md, confirm rules.md's testing strategy can validate it.

- **Ubiquitous** (`The system shall ...`): invariant or continuous-state test
- **Event-driven** (`When X, the system shall ...`): event-simulation test
- **State-driven** (`While X, the system shall ...`): state-setup test
- **Unwanted behavior** (`If X, then the system shall ...`): negative-path test that asserts the prohibited outcome is absent
- **Combined** (`While X, when Y, ...`): both state and event setup
- **Boundary patterns** (within / at / beyond): three-point coverage
- **Negative criteria** (authz, side-effect paths): explicit assertions on the "and not" half — denial **and** no disclosure **and** no mutation; a strategy that says "assert 403" does not fit
- **Refusal criteria** (state machine): service-level test that sets up the disallowed state and asserts refusal plus absence of side effect
- **Data-exactness criteria**: migration/fixture test on a fresh database asserting every row by value and every credential with the encoder; count-only strategies do not fit
- **Fidelity criteria**: round-trip test comparing every enumerated field
- **Lifecycle / end-to-end criteria**: HTTP-level test **per UC main success scenario** as each named actor, with the steps copied into the rule and one leg per state-changing extension; a single "lifecycle test" with no step list, a test derived from a path no UC describes, or a service-level chain does not fit (MAJOR)

Also confirm the strategy fixes the **level** (web slice vs full context vs HTTP), the **fixture** (isolated test datasource) and the **double contract** (no stub may return what production cannot). If any of the three is unspecified, it is a MAJOR: the executor will choose the cheapest and the tests will pass without proving the AC.

If the rules' testing strategy doesn't cover one of these patterns that criteria uses, it's a MAJOR finding. The implementing agent will write tests anyway, but without guidance the test shape may not match the AC shape.

## 5. Risk Hotspots (required)

Up to five areas most likely to go wrong even with a passing spec. For each: area / reason / mitigation.

If you cannot identify any, write `Hotspots: none considered material` with a one-line justification. Empty is acceptable; absent is not.

This is the section most likely to be useful to the implementing agent. It's the only output review produces that upstream cannot.

# Plan Review (plan mode)

Run only in plan mode, after the `tasks` skill has written `spec/tasks.yaml`. Inputs: `tasks.yaml`, `criteria.md`,
`rules.md` (URL→role matrix, testing strategy), `spec.md` (use cases, checkpoint walkthroughs), `review.md` (hotspots
that must appear as `risk`), and the codebase grounding above. Severity and verdict are as in spec mode; fixes route to
the `tasks` skill. Run all six categories.

## P1. Checkpoint Reachability (the centerpiece)

For every phase, take the **union of every `artifact` in the phase** and walk every line of `checkpoint.criteria`, the
walkthrough script, and the phase's demo sentence with the question: *can a human perform this with only these files?*
For each URL: which artifact maps it (controller), renders it (template), guards it (security matrix entry), and
localizes it (message keys)? For each assertion criterion: which test class in which task's `artifact` asserts it?

- A criterion, walkthrough URL or demo-sentence action with no artifact behind it is a BLOCKER against `tasks`.
- A controller that appears first in a later phase than the checkpoint that needs its route is a BLOCKER.
- A phase named for an outcome ("walking skeleton", "UC-1 end to end") whose artifacts do not include the HTTP surface
  for that outcome is a BLOCKER.

## P2. Artifact Exactness

- Every `artifact` is a list of exact repository-relative paths; any glob, ellipsis (`…`, `...`), or prose entry
  ("ownership guard service", "templates for staff") is a MAJOR — the executor will pick a name, and the pipeline will
  not notice the rename.
- Every test artifact names its class and methods.
- Every class, page, route or message key mentioned in a task `description` appears in that task's `artifact` or an
  earlier task's; an unowned mention is a MAJOR.
- Every path is creatable in the current layout (package exists or is consistent with grounding).

## P3. Route Inventory

- Every phase that introduces a request mapping has a `routes` list; each entry has an owning task whose `artifact`
  contains a controller class. Missing `routes` on such a phase is a BLOCKER.
- Every URL in the phase's checkpoint criteria, walkthrough and UC steps is in `routes`. Every route in `routes` is in
  the rules' URL→role matrix (or recorded as an assumption for `rules`). Cross-check both directions.
- A skeleton phase has a dedicated *HTTP surface* task (MAJOR if controllers are folded into template/security tasks).

## P4. Coverage Consistency

- `phase.covers` equals the union of the phase's tasks' `covers.acs` — both directions. An AC the phase claims that no
  task asserts is a BLOCKER (the checkpoint will demand it; no task will produce it).
- No task `description` defers an AC that its own `covers.acs` or its phase's `covers` lists ("AC-138's full lifecycle
  is completed in phase-5" while phase-1 lists AC-138). Two readings is a MAJOR.
- Every AC in `criteria.md` is in some `covers.acs` or in `coverage_deferrals` (spot-check; `tasks` verified this).
- Every Risk Hotspot from `review.md` appears as a `risk` on a task; every MAJOR condition appears as a task or risk.

## P5. Task Shape and Validation Literalness

- No task covers more than 5 ACs; no `L` task without a following intermediate checkpoint; no single-line
  `description` or `validation` beyond ~600 characters. Each is a MAJOR: dense prose is what the executor paraphrases.
- Every `validation` is a list of `TestClass.method → AC-n → shape` bullets; every AC in `covers.acs` appears in a
  bullet; every named test class is in `artifact`. Any "run the tests" / "tests pass" / prose paragraph is a MAJOR.
- Every bullet's shape matches the AC pattern per the rules' Testing Strategy (by value for data exactness; `never()` +
  no disclosure for negative authz; HTTP level with real `SecurityFilterChain` and CSRF for lifecycle). A bullet that
  permits a weaker level than the AC demands is a MAJOR (verification-shape drift, now in the plan).
- Every bullet is **literally satisfiable** in this codebase with the declared rules and versions: the property exists
  in the declared framework version, the test can be written without violating a MUST NOT, the route it redirects to
  exists. An unsatisfiable bullet is a BLOCKER — the executor's only correct move would be to stop, and the plan
  should not invite it.

## P6. Skeleton Ordering and Plan Guards

When `organizing_principle` starts with `walking_skeleton`:

- Phase-1 declares `skeleton_test` (the primary UC's HTTP-level e2e test), written in the first test-bearing task and
  red until the last task; every task from the HTTP-surface task onward names the UC step the skeleton test reaches.
  An e2e test that is the **last** task with no `skeleton_test` declaration is a MAJOR (the executor will write it
  last, as whatever passes).
- Phase-1 contains the *Plan guards* task (`RouteInventoryTest`, `ArtifactInventoryTest`, `AcTagCoverageTest`) and
  the *Test environment* task before any other test-bearing task. Missing guards is a MAJOR.
- Phase-1 follows the outside-in order (environment + guards → skeleton test → security + HTTP surface → data →
  presentation → seams → green); an order that puts domain/solver/interpreter tasks before the HTTP surface is a MAJOR.
- Intermediate checkpoints exist after the HTTP-surface task and after any unsplit `L` task.

For any principle: the first phase that adds a test contains the *Plan guards* task.

# Finding Format

Each finding has:
- Stable ID: `BLOCKER-1`, `MAJOR-1`, `MINOR-1` (numbered per severity)
- Title
- Source document(s)
- Issue with exact problematic text quoted
- Impact: what goes wrong if not fixed
- Resolution: which upstream skill to rerun, and on what
- Optional `Related:` for clusters

# Fix Plan Format

Produce a Fix Plan when the verdict is PASS WITH CONDITIONS or FAIL. Omit for PASS. Minors do not appear in the Fix Plan.

Structure:

- One header line naming the execution order through the pipeline, e.g. `Execution order: spec → criteria → rules → review`. Include only steps that have fixes; always end with `review`. In plan mode the order is `tasks → review (plan)`, unless a plan finding exposes an upstream defect (a route missing from the rules' matrix, a UC step with no AC), in which case the upstream step precedes `tasks` and the spec-mode review must be rerun before `tasks`.
- One subsection per upstream step that has fixes, in pipeline order (spec, then criteria, then rules; then tasks in plan mode).
- Within each subsection, a numbered list of fixes. Each fix has:
    - A `[cascades]` or `[localized]` tag
    - The finding ID this fix resolves (`BLOCKER-N`, `MAJOR-N`)
    - A concrete instruction on what to change
    - For cascading fixes: a one-line note on what will need to be rerun downstream

A fix **cascades** if any of:
- It adds, removes, or changes a B-N in spec.md
- It changes an assumption, edge case, or scope item that has downstream coverage
- It changes the EARS pattern of an AC, or removes or adds an AC that has covering rules
- It changes the Design section, an Ecosystem Survey decision, or a rule that other rules depend on

A fix is **localized** if it stays inside a single section of a single document and produces no change to any structure that downstream depends on. Typical localized fixes: naming cleanup, stale references, rationale clarification, removing redundant rules with no downstream `Covers:` references.

If uncertain, mark `[cascades]`. False cascades cost one unnecessary rerun; false localized marks corrupt the chain.

The plan ends with `### review` indicating when to rerun this skill (once all upstream fixes are in, not per-fix).

# Success Criteria

Complete only when ALL hold:

- Codebase grounding pass executed before any category, including the full route enumeration and the verification of every "pre-existing" / "stock" claim against `HEAD`
- Spec mode: all five categories run; plan mode: all six plan categories run; each either confirms pass or lists findings
- Plan mode: for every phase, every checkpoint criterion and walkthrough URL is mapped to the artifact(s) that satisfy it, or is a finding; `phase.covers` was compared with the union of task `covers.acs` in both directions
- No artifact references a decision, table or definition by pointer to another file; every normative table, state table and URL→role matrix was checked for presence, not just for being mentioned
- Every negative authz AC was checked against the full route list
- Risk Hotspots populated (or explicitly empty with reason)
- Every finding has all required fields
- Every quoted text matches the upstream file verbatim
- Verdict matches severity counts
- Fix Plan present for non-PASS verdicts: groups by step, orders by pipeline position, marks every fix `[cascades]` or `[localized]`, ends with `### review`

Do not write a partial file.

# Output

Spec mode: write to `spec/review.md`. Plan mode: write to `spec/plan-review.md` (template below the spec one). For a
clean spec or plan, this should be a short document. Bloat is a smell.

```
# Spec Review: <Feature>

## Summary
- Feature: <name>
- Verdict: <PASS | PASS WITH CONDITIONS | FAIL>
- Counts: <N blockers, N majors, N minors>
- Action: <one line; references Fix Plan for non-PASS verdicts>

## Discipline Check
<one paragraph: confirmed clean, or list of findings>

## Conflicts
<empty if none; otherwise findings with both sources quoted>

## Codebase Grounding
<one paragraph: design fits codebase, or list of findings>

## EARS ↔ Test Strategy
<one paragraph: each pattern has a fitting test approach, or list of findings>

## Risk Hotspots
<up to five: area / reason / mitigation; or "none considered material" with reason>

## Fix Plan
For PASS WITH CONDITIONS or FAIL only; omit entirely for PASS.

Execution order: <pipeline steps with fixes, in order, ending with review>

### <step> (rerun <position>)
1. [cascades|localized] <FINDING-ID>: <what to change>
   <if cascades: one line on what needs rerunning downstream>

### review
Rerun once all upstream fixes are in.
```

Plan mode:

```
# Plan Review: <Feature>

## Summary
- Feature: <name>
- Plan: spec/tasks.yaml (<n> phases, <n> tasks, organizing principle <…>)
- Verdict: <PASS | PASS WITH CONDITIONS | FAIL>
- Counts: <N blockers, N majors, N minors>
- Action: <one line; references Fix Plan for non-PASS verdicts>

## Checkpoint Reachability
<per phase: each criterion / walkthrough URL → the artifact(s) that satisfy it, or the finding id>

## Artifact Exactness
<confirmed clean, or findings>

## Route Inventory
<per phase: routes ↔ owning tasks ↔ URL→role matrix; or findings>

## Coverage Consistency
<phase.covers vs union of task covers.acs per phase; deferral contradictions; hotspots as risks; or findings>

## Task Shape and Validation Literalness
<confirmed clean, or findings>

## Skeleton Ordering and Plan Guards
<n/a unless walking_skeleton; otherwise confirmed order, skeleton_test, guards, intermediate checkpoints; or findings>

## Conditions for execute
<for PASS WITH CONDITIONS: per task, the conditions execute must treat as extra validation bullets; omit for PASS>

## Fix Plan
For PASS WITH CONDITIONS or FAIL only; omit entirely for PASS.

Execution order: <e.g. tasks → review (plan), or rules → review → tasks → review (plan)>

### tasks (rerun <position>)
1. [cascades|localized] <FINDING-ID>: <what to change in tasks.yaml>

### review (plan)
Rerun once the plan fixes are in.
```
