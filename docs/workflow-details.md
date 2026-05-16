# Superspec Workflow Details

A Superspec change moves through **five phases**, broken into **nine concrete steps**. This page walks through each step — what it is, **why it's required**, what concretely happens, which source system (OpenSpec or Superpowers) owns it, and what step from the other source is replaced.

Use it as the long-form companion to the [workflow overview](workflow.md), [top-level README](../README.md), and the schema's own [INTEGRATION.md](../openspec/schemas/superspec/INTEGRATION.md).

## At a glance

| Phase | # | Step | Brief Why | Source Phase Used |
|---|---|---|---|---|
| **1. Brainstorm** | 1 | `brainstorm` | Avoid building the wrong thing. | Superpowers: `brainstorming` |
| **2. Artifact Creation** | 2 | `proposal` | Define the change boundary. | OpenSpec: `proposal` artifact |
| | 3 | `design` *(optional)* | Explain the chosen solution. | Hybrid: Superpowers brainstorming + OpenSpec `design.md` |
| | 4 | `specs` | Create the testable contract. | OpenSpec: `specs` artifact |
| | 5 | `tasks` | Scope the work. | OpenSpec: `tasks` artifact |
| | 6 | `plan` | Make the work executable. | Superpowers: `writing-plans` |
| **3. Code Implementation** | 7 | `apply` | Change the system. | Superpowers: `using-git-worktrees`, `subagent-driven-development`, `test-driven-development`, `requesting-code-review` |
| **4. Spec Validation** | 8 | `verify` | Prove it matches intent. | OpenSpec: `/opsx:verify` |
| **5. Archival** | 9 | finish / archive boundary | Close the lifecycle cleanly. | Hybrid: Superpowers `finishing-a-development-branch` + OpenSpec `/opsx:archive` |

Read top-to-bottom for the full lifecycle, or jump to any step.

---

## Phase 1: Brainstorm

The Brainstorm phase nails down the idea for the change before any formal artifacts are produced. It contains a single step.

### Step 1. Discovery — `brainstorm`

> Front-end discovery and design shaping before formal artifacts are created.

**Brief why:** Avoid building the wrong thing.

**Why it's required.** This phase prevents the workflow from formalizing the wrong problem. It forces ambiguity, constraints, alternatives, and trade-offs to be explored *before* OpenSpec artifacts harden the change. Once `proposal` and `specs` exist, they become a contract; everything downstream — design decisions, task decomposition, code, tests, archive history — anchors to them. If the underlying problem was misframed, that misframing now lives in the contract and is expensive to walk back.

**What the step does.** Collaborative design exploration using **Superpowers' brainstorming skill**. It explores project context, asks clarifying questions one at a time, proposes 2–3 approaches with trade-offs, presents design sections for approval, and outputs the validated design.

**Source phase used.** `superpowers:brainstorming`.

**Step not used / replaced and why.** OpenSpec's `/opsx:explore` is mostly replaced. OpenSpec explore investigates unclear requirements, but Superpowers brainstorming is more structured for collaborative design validation. OpenSpec still has `/opsx:explore` available for ad-hoc inspection, but Superspec deliberately picks Superpowers as the stronger discovery phase.

---

## Phase 2: Artifact Creation

The Artifact Creation phase produces every governed artifact for the change — the proposal, optional design, delta specs, coarse tasks, and the executable micro-task plan. It contains five steps (steps 2–6).

### Step 2. Change Contract — `proposal`

> Converts the brainstormed design into an OpenSpec-style change contract.

**Brief why:** Define the change boundary.

**Why it's required.** This step creates the formal OpenSpec change boundary. Every later phase — specs, design, tasks, verification, archive — depends on a single shared answer to "what capability is changing, why, and with what impact?" Without a proposal, the workflow has nothing to validate against and no stable target for the delta-spec mechanism that powers OpenSpec's archival history.

**What the step does.** The agent reads `brainstorm.md` and creates `proposal.md`, extracting:

- **Why** (50–1000 chars, validated by OpenSpec's zod schema)
- **What Changes** (specific changes agreed during brainstorming)
- **Capabilities** (new vs. modified, in kebab-case — each becomes a `specs/<name>/spec.md`)
- **Impact** (affected code, APIs, dependencies, systems)

The Capabilities section is the load-bearing piece: it forms the contract between proposal and specs phases, and every name listed must end up with a corresponding spec file.

**Source phase used.** OpenSpec `proposal` artifact / `/opsx:propose` behavior.

**Step not used / replaced and why.** Superpowers has no equivalent standalone proposal phase. Brainstorming saves a design document, but it does not produce an OpenSpec proposal with capability-level spec-contract semantics. Superspec slots OpenSpec's proposal in immediately after Superpowers brainstorming so the design is captured as a governed change rather than a free-form note.

---

### Step 3. Architecture — `design` *(optional)*

> Captures implementation approach and technical rationale.

**Brief why:** Explain the chosen solution.

**Why it's required.** Design records the *technical* approach and rationale behind the change. This keeps requirements (`specs`) free of implementation details while preserving the decisions needed for code review, ongoing maintenance, rollout planning, and future audits. It is the only artifact that explains *why this implementation was chosen over alternatives*.

This artifact is **optional** — only write it when there are non-trivial technical decisions to record (per `openspec-conventions`). For simple changes the brainstorm + proposal + specs combination is enough.

**What the step does.** Create or refine `design.md`, covering:

- **Context** — background, current state, constraints, stakeholders
- **Goals / Non-Goals** — explicit scope
- **Decisions** — the chosen approach, with rationale
- **Risks / Trade-offs** — known compromises
- *(optional)* migration plan and open questions

Focus is on architecture and approach, not line-by-line implementation. If brainstorming already produced a design document, the agent reviews and refines it rather than starting over.

**Source phase used.** Hybrid — Superpowers `brainstorming` may pre-fill `design.md`, then the OpenSpec `design.md` artifact takes ownership.

**Step not used / replaced and why.** A pure-Superpowers design document at the default Superpowers location is replaced. The schema's `brainstorm` instruction explicitly redirects design output into the OpenSpec change directory (`openspec/changes/<name>/design.md`) instead of `docs/superpowers/specs/`. This keeps every artifact for one change co-located in one directory, so `openspec validate` and `openspec archive` see all of it.

---

### Step 4. Requirements — `specs`

> Defines testable system behavior.

**Brief why:** Create the testable contract.

**Why it's required.** Specs are the testable behavioral contract. OpenSpec needs them to produce *delta specs*, validate the change end-to-end, and — most importantly — prove after implementation that the delivered system actually satisfies the intended behavior. Without specs, "did we build the right thing?" has no objective answer.

**What the step does.** Create one specification file per capability listed in the proposal, at `specs/<capability>/spec.md`. Each file uses delta sections:

- `## ADDED Requirements` — new behavior
- `## MODIFIED Requirements` — existing requirements changed (header must match the live spec exactly; full new content required, not a diff)
- `## REMOVED Requirements`
- `## RENAMED Requirements` — header rename only

Hard formatting rules (validated by OpenSpec):

- Requirement sentences must contain `SHALL` or `MUST`.
- Each Requirement must have at least one `#### Scenario:` block.
- Scenarios must use level-4 headings (`####`); level-3 or bullet form silently fails validation.

**Source phase used.** OpenSpec `specs` artifact.

**Step not used / replaced and why.** Superpowers has no equivalent — this is a major contribution from the OpenSpec/spec-driven side. Superpowers has plans and tests, but not capability-level requirement deltas with archive semantics.

---

### Step 5. Implementation Scope — `tasks`

> Creates the coarse implementation checklist.

**Brief why:** Scope the work.

**Why it's required.** `tasks.md` turns the approved change into a *scoped body of implementation work*. It gives OpenSpec a trackable checklist that can be parsed during apply and verify, without dropping immediately into low-level coding steps. It also creates a coarse grouping that maps cleanly to commits and review boundaries.

**What the step does.** Create `tasks.md` with grouped, ordered, dependency-respecting checkbox tasks:

```
## 1. Group Name
- [ ] 1.1 Task description
- [ ] 1.2 Task description
```

The format matters: the apply phase parses `- [ ]` to track progress, so tasks not using checkboxes are invisible to it. Each task should be small enough to complete in one session.

**Source phase used.** OpenSpec `tasks` artifact.

**Step not used / replaced and why.** Superpowers' `writing-plans` is **not yet** invoked here. Superpowers planning is more granular (2–5 minute micro-steps). Superspec deliberately keeps a coarse OpenSpec layer first — it is the level a human reviewer actually wants to scan during proposal review — and decomposes into Superpowers-style micro-steps in the next phase.

---

### Step 6. Implementation Plan — `plan`

> Converts coarse OpenSpec tasks into executable micro-steps.

**Brief why:** Make the work executable.

**Why it's required.** This phase transforms high-level tasks into instructions an agent (or a human pairing with an agent) can execute consistently. It is where Superpowers adds file-level steps, exact test commands, the TDD sequence, and commit points. Without it, the apply loop has to re-decide *how* to execute every task, which is where drift between spec and code typically starts.

**What the step does.** Invokes `superpowers:writing-plans`. The skill reads `tasks.md` (and `design.md` if present), then for each task produces a `plan.md` section with:

- 2–5 minute TDD-style micro-steps (RED → GREEN → REFACTOR)
- exact file paths and code/test snippets
- the test command to run
- a commit point at the end of each task

**Source phase used.** `superpowers:writing-plans`.

**Step not used / replaced and why.** Going directly from OpenSpec tasks to `/opsx:apply` is delayed. Native OpenSpec can apply task-by-task, but Superspec inserts a stricter micro-planning layer between tasks and apply so the executor has unambiguous instructions for each TDD cycle.

---

## Phase 3: Code Implementation

The Code Implementation phase produces the actual code for the change, executed in an isolated git worktree under Superpowers' subagent + TDD + code-review loop. It contains a single step.

### Step 7. Implementation — `apply`

> Executes the implementation.

**Brief why:** Change the system.

**Why it's required.** This is the phase that actually changes code, config, or system state. In Superspec it is intentionally delegated to Superpowers' stricter execution loop instead of vanilla OpenSpec apply, so every task ships through the same TDD-and-review pipeline.

**What the step does.** The apply phase chains four Superpowers skills (with two more triggered transitively) and writes a receipt:

1. **`using-git-worktrees`** — creates an isolated workspace at `.worktrees/<change-name>/`, switches to a new branch, runs project setup, and confirms a clean test baseline.
2. **`subagent-driven-development`** (default path, requires subagent support) — the main agent reads `plan.md` and dispatches a fresh subagent per micro-task. Each subagent transitively activates:
   - **`test-driven-development`** — write a failing test first, watch it fail, then write the minimum code to make it pass. Implementation written before a failing test is deleted and redone.
   - **`requesting-code-review`** — after each task, a code-reviewer subagent checks spec compliance and code quality. A final review runs over the whole implementation before apply concludes.
   - As coarse tasks complete, `tasks.md` checkboxes flip to `- [x]`.
3. **Receipt** — at the end of the phase, a minimal `apply.md` is written per `openspec/schemas/superspec/templates/apply.md`: iteration counter, applied-at timestamp, executor identity, worktree path, branch, commit range, and tasks completed X of Y. This is the v2 DAG artifact that gates `verify`. If `apply.md` already exists, the iteration counter is incremented.
4. **`finishing-a-development-branch`** — only invoked at the very end; covered in phase 9 below.

**Pre-flight requirement.** Before creating the worktree, the change directory `openspec/changes/<name>/` must already be committed on the current branch. Otherwise, when the worktree merges back, git will refuse with "untracked files would be overwritten by merge."

**Fallback path.** If subagents are unavailable, `superpowers:executing-plans` is the documented fallback. It does *not* transitively activate TDD or code review — you take responsibility for both manually. On Claude Code, subagents are available, so this path is rarely needed.

**Source phase used.** `superpowers:using-git-worktrees`, `superpowers:subagent-driven-development`, `superpowers:test-driven-development`, `superpowers:requesting-code-review`.

**Step not used / replaced and why.** Vanilla `/opsx:apply` execution is effectively overridden. OpenSpec apply can work through tasks directly, but Superspec chooses Superpowers' stricter loop with isolated worktrees, subagents, TDD, and review. The OpenSpec docs show `/opsx:apply` "working through tasks," but in this schema Superpowers is the executor.

### Convergence loop (apply → verify → repeat)

Because `apply.md` and `verify.md` are both overwritten on each iteration and both carry the same `Iteration:` counter, Superspec supports (and recommends) a convergence loop where apply and verify run repeatedly until verify reports a clean state.

```text
        plan
         │
         ▼
       apply  ────► apply.md   (iteration N)
         │
         ▼
       verify ────► verify.md  (iteration N)
         │
         ├── PASS or PASS_WITH_WARNINGS ──► finishing-a-development-branch → archive
         │
         ├── FAIL, items fixable by code change ──► return to apply (N+1)
         │
         ├── FAIL, items in artifacts (spec drift) ──► fix artifact → apply (N+1)
         │
         └── Iteration > 5 ──► stop; report to the user
```

Termination rules (recorded in the verify artifact instruction and the apply: phase block step 4 in schema.yaml):

- **PASS** — proceed to `finishing-a-development-branch` and archive.
- **PASS_WITH_WARNINGS** — proceed; warnings are recorded for posterity but do not block.
- **FAIL with code-fixable items** — return to the apply phase, re-run, overwrite `apply.md` with iteration N+1, then re-run verify.
- **FAIL with artifact-level items** (e.g. spec drift, a requirement that is no longer satisfied by the plan) — fix the offending artifact first, then re-enter apply with iteration N+1.
- **Iteration > 5** — stop the loop and report to the user. This is a soft safeguard against non-convergence; the schema enforces nothing here, but the verify instruction tells the agent to halt the pattern.

The schema enforces only the file-existence dependency (`verify.requires: [apply]`). The iteration decision — whether to loop, stop, or escalate — is made by the agent (or, in a follow-up change, by a dedicated loop-runner command). No automated loop runner ships with v2.

---

## Phase 4: Spec Validation

The Spec Validation phase proves the implementation actually matches the proposal, specs, design, and tasks before the change is considered complete. It contains a single step.

### Step 8. Validation — `verify`

> Validates the completed implementation against the artifacts.

**Brief why:** Prove it matches intent.

**Why it's required.** Verify closes the loop between intent and implementation. It checks that the proposal, specs, design, tasks, and the actual code still agree before the change is considered complete. Code review alone — even a thorough Superpowers review — only checks that the code matches the *plan*; it does not check that the plan still matches the *specs*.

As of schema v2, `verify.requires: [apply]` — the DAG genuinely gates verify on the existence of `apply.md`. In v1 the dependency was declared as `[plan]` with a comment that "actually verify needs apply"; that mismatch was a frequent source of agents running verify before apply had executed. The v2 schema removes the mismatch.

**What the step does.** Invokes `openspec-verify-change` (the user-facing equivalent is `/opsx:verify`). Five checks are run, with results recorded in `verify.md`:

1. **Structural validation** — `openspec validate --all --json` returns all PASS.
2. **Task completion** — every `- [ ]` in `tasks.md` is now `- [x]`.
3. **Delta spec sync state** — `changes/<name>/specs/` has been synced into `openspec/specs/`.
4. **Design / specs coherence** — design decisions and spec requirements remain consistent (non-blocking warning).
5. **Implementation signal** — no unstaged files in the worktree.

If any check fails, the agent returns to the offending artifact, fixes it, and re-runs verify.

**Source phase used.** OpenSpec `/opsx:verify`.

**Step not used / replaced and why.** Superpowers code review alone is not enough. Review checks plan compliance and code quality; OpenSpec verify checks artifact-level correctness across specs, tasks, design, and implementation. Both layers are kept — review during apply, verify after apply.

---

## Phase 5: Archival

The Archival phase closes the lifecycle: it cleans up the development branch, applies the change's delta specs against the project's living specs, and moves the completed change into the archive. It contains a single step.

### Step 9. Finalization — finish / archive boundary

> Completes the development branch and prepares OpenSpec finalization.

**Brief why:** Close the lifecycle cleanly.

**Why it's required.** This phase separates *development completion* from *lifecycle closure*. Superpowers cleans up the branch, PR, and worktree (Git hygiene). OpenSpec applies delta specs to the live spec tree and moves the completed change into the archive (history hygiene). Skipping either leaves the repo in a half-finished state — either dangling worktrees and PRs, or specs that no longer reflect what shipped.

**What the step does.** Two distinct moves, in order:

1. **Branch cleanup** — `superpowers:finishing-a-development-branch` walks merge / PR / worktree teardown.
2. **Archive** — `/opsx:archive` (or `openspec archive`) applies the change's delta specs against `openspec/specs/` (apply order: RENAMED → REMOVED → MODIFIED → ADDED) and moves `changes/<name>/` into the archive.

**Recommended (non-blocking).** Before archiving, write a short retrospective at `retrospective.md` in the change directory. Suggested sections: Wins, Misses, Plan deviations, Skill/workflow compliance, Surprises, and Promote candidates (learnings worth moving into long-term memory or CLAUDE.md).

**Source phase used.** Hybrid — `superpowers:finishing-a-development-branch` + OpenSpec `/opsx:archive`.

**Step not used / replaced and why.** Neither source fully replaces the other. Superpowers handles the development branch's Git/PR/worktree closure; OpenSpec archives the spec-driven change so future audits can replay history. Both are needed to leave the repo in a clean state.

---

## Superpowers skill index & fallbacks

A flat view of every Superpowers skill Superspec invokes, where it's hooked in, and how to recover if it's unavailable.

### The 7 Superpowers touch points

| # | Skill | Hook | Trigger |
|---|---|---|---|
| 1 | `superpowers:brainstorming` | Phase 1 / Step 1 (`brainstorm`) | Direct |
| 2 | `superpowers:writing-plans` | Phase 2 / Step 6 (`plan`) | Direct |
| 3 | `superpowers:using-git-worktrees` | Phase 3 / Step 7 (`apply`), sub-step 1 | Direct |
| 4 | `superpowers:subagent-driven-development` | Phase 3 / Step 7 (`apply`), sub-step 2 | Direct |
| 5 | `superpowers:test-driven-development` | inside #4 | **Transitive** |
| 6 | `superpowers:requesting-code-review` | inside #4 | **Transitive** |
| 7 | `superpowers:finishing-a-development-branch` | Phase 5 / Step 9 (finalization) | Direct |

Direct triggers are invoked by the schema's artifact instructions; transitive triggers are activated *inside* another skill's loop (the subagent-driven-development workhorse — see [Step 7](#step-7-implementation--apply) for how it dispatches per-task subagents that enforce TDD and run code review).

### Manual fallbacks

If a Superpowers skill is unavailable (not installed, version mismatch), each artifact instruction includes a manual fallback so Superspec degrades gracefully to plain OpenSpec:

| Step | Skill normally invoked | Manual fallback |
|---|---|---|
| 1 — `brainstorm` | `superpowers:brainstorming` | Write `brainstorm.md` directly. |
| 6 — `plan` | `superpowers:writing-plans` | Write `plan.md` directly. |
| 7 — `apply` (subagents unavailable) | `superpowers:subagent-driven-development` | Use `superpowers:executing-plans`, or run tasks manually. Either path requires you to maintain TDD discipline and invoke `superpowers:requesting-code-review` yourself — neither is activated transitively when the subagent path is bypassed. |

---

## See also

- [Workflow overview](workflow.md) — visual overview and quick mental model
- [README](../README.md) — install and quick start
- [`openspec/schemas/superspec/schema.yaml`](../openspec/schemas/superspec/schema.yaml) — machine-readable definition that drives all of the above
- [`openspec/schemas/superspec/INTEGRATION.md`](../openspec/schemas/superspec/INTEGRATION.md) — CLI cheat sheet, lifecycle, and design-choice rationale
- [`openspec/schemas/superspec/README.md`](../openspec/schemas/superspec/README.md) — schema design log and fallback strategy
