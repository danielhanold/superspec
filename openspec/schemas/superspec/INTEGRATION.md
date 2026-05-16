# OpenSpec × Superpowers Integration Guide

> This document explains how the `sdd-plus-superpowers` schema integrates OpenSpec's artifact governance workflow with Superpowers' execution skills into a single workflow. It serves as a reference table for new member onboarding, change reviews, and as required reading before modifying the schema.
>
> Corresponding schema version: `sdd-plus-superpowers` v2

---

## 1. The Nature of the Integration: What Goes Where

OpenSpec handles **"WHAT"** — governance, validation, and archival of markdown artifacts like proposal / specs / design / tasks.
Superpowers handles **"HOW"** — execution skills such as brainstorming conversations, TDD discipline, subagent dispatch, code review, etc.

The two are integrated through a custom schema [schema.yaml](./schema.yaml). The integration is not at the code level — instead, OpenSpec artifact instructions contain directives like "at this step, use the Skill tool to invoke `superpowers:xxx`." **No superpowers skill files are modified**, nor is the OpenSpec CLI — the integration is purely at the instruction layer.

---

## 2. Overview of the 7 Superpowers Touch Points

| # | Superpowers skill | Where it hooks in | Trigger method |
|---|---|---|---|
| 1 | `superpowers:brainstorming` | `brainstorm` artifact instruction | Direct |
| 2 | `superpowers:writing-plans` | `plan` artifact instruction | Direct |
| 3 | `superpowers:using-git-worktrees` | apply step 1 | Direct |
| 4 | `superpowers:subagent-driven-development` | apply step 2a | Direct |
| 5 | `superpowers:test-driven-development` | (auto-triggered inside #4) | **Transitive** (SKILL.md L205 / L274) |
| 6 | `superpowers:requesting-code-review` | (auto-triggered inside #4) | **Transitive** (SKILL.md L270) |
| 7 | `superpowers:finishing-a-development-branch` | apply step 5 | Direct |

There is also one **fallback**:

- `superpowers:executing-plans` (apply step 2b) — only used when the current platform lacks subagent support. On Claude Code, always use 2a. Per `superpowers:executing-plans` SKILL.md L14: "If subagents are available, use `superpowers:subagent-driven-development` instead of this skill."

---

## 3. Artifact DAG (with Superpowers Injection Points)

```text
┌──────────────┐
│  brainstorm  │ ◄── superpowers:brainstorming
│  (root)      │     (2-3 approaches + Alternatives Considered)
└──────┬───────┘
       │
       ├──► ┌──────────┐
       │    │ proposal │    Why (50-1000 chars) / What Changes / Capabilities
       │    └────┬─────┘
       │         │
       │         ▼
       │    ┌──────────────────┐
       │    │ specs/**/*.md    │    ADDED / MODIFIED / REMOVED / RENAMED
       │    │ (delta specs)    │    Each requirement includes SHALL/MUST + scenario
       │    └────┬─────────────┘
       │         │
       │         ▼
       │    ┌──────────┐
       │    │  tasks   │    Coarse-grained checkboxes (tracking vehicle for apply)
       │    └────┬─────┘
       │         │
       │         ▼
       │    ┌──────────┐
       │    │  plan    │ ◄── superpowers:writing-plans
       │    └────┬─────┘     (2-5 minute micro-steps)
       │         │
       │         ▼
       │    ┌──────────┐
       │    │  apply   │ ◄── superpowers:using-git-worktrees
       │    │ (DAG +   │ ◄── superpowers:subagent-driven-development
       │    │  apply:  │         ├── superpowers:test-driven-development (transitive)
       │    │  phase)  │         └── superpowers:requesting-code-review (transitive)
       │    │          │ ◄── superpowers:finishing-a-development-branch
       │    │ writes   │
       │    │ apply.md │
       │    └────┬─────┘
       │         │
       ▼         ▼
    ┌──────────┐ ┌──────────┐
    │  design  │ │  verify  │ ◄── openspec-verify-change (5 checks)
    │(optional)│ └──────────┘
    └──────────┘
```

**Key points**:

- `design` is an **optional leaf**. Brainstorm still attempts to pre-populate design.md, but tasks no longer hard-depend on it (`tasks.requires: [specs]`). Per OpenSpec conventions: `design.md` is only written when non-trivial technical decisions need explanation.
- `apply` is now a **real DAG node** as of schema v2. It generates `apply.md` (a minimal receipt — iteration counter, worktree, branch, commit range, task counts) so the DAG can honestly express "verify depends on apply having run." The canonical `/opsx:apply` instruction body still lives in the top-level `apply:` phase block; the apply artifact's own instruction is a short redirect to avoid drift.
- `verify` now requires `apply` (was `plan` in v1). The OpenSpec CLI will refuse to surface verify as a `ready` artifact until `apply.md` exists.
- The convergence loop (apply → verify → loop back on code-fixable FAILs, capped at 5 iterations) is documented in `docs/workflow-details.md`. The schema enforces the file-existence dependency; the iteration decision is made by the agent or by a future loop-runner command (not in v2 scope).

---

## 4. Complete Development Workflow (Lifecycle of a Single Change)

### Step 0: Decide Whether to Use the Change Process

Ask yourself: is this a behavioral change?

| Type | Requires a change? | Which schema |
|---|---|---|
| New feature / new capability | Yes | `sdd-plus-superpowers` |
| Breaking change | Yes | `sdd-plus-superpowers` |
| Architecture change | Yes | `sdd-plus-superpowers` |
| Bug fix (restoring original behavior) | No | Direct PR |
| Adding/backfilling tests | No | Direct PR |
| Build tool tweaks (linter rules, coverage thresholds, etc.) | No | Direct PR |
| Non-breaking dependency upgrades | No | Direct PR |
| Documentation updates | No | Direct PR |

This decision logic is documented in the "When Not to Create a Spec" section of [openspec/specs/README.md](../../specs/README.md).

---

### Step 1: Create a Change + Enter Brainstorming

```bash
/opsx:new my-feature --schema sdd-plus-superpowers
# → Creates openspec/changes/my-feature/ empty directory + .openspec.yaml
# → Displays brainstorm artifact instructions
```

Then:

```bash
/opsx:continue
# → Triggers the brainstorm artifact
# → Instruction says "use the Skill tool to invoke superpowers:brainstorming"
# → Enters multi-turn interactive conversation: context exploration → clarify questions → 2-3 approaches + trade-offs → approve design
# → After conversation ends, writes brainstorm.md (with Alternatives Considered)
# → If design artifacts were produced, simultaneously writes design.md (pre-populated)
```

**Key**: This step is the alignment ceremony for the entire workflow. All subsequent proposal / specs are distilled from brainstorm.md.

---

### Step 2: Sequentially Produce proposal → specs → tasks → plan

You can `/opsx:continue` step by step (with human review opportunity at each step), or `/opsx:ff` to fill in all remaining artifacts at once.

| Step | Output | Key rules |
|---|---|---|
| 2a | `proposal.md` | Why section 50-1000 chars; Capabilities section lists new/modified capabilities |
| 2b | `specs/<capability>/spec.md` | 4 delta types (ADDED / MODIFIED / REMOVED / RENAMED); each requirement includes SHALL/MUST + `#### Scenario:` |
| 2c (opt) | `design.md` | Only written when technical decisions need explanation; brainstorm may have already pre-populated it |
| 2d | `tasks.md` | Coarse-grained checkboxes (`- [ ] X.Y description`), tracked during apply |
| 2e | `plan.md` | `/opsx:continue` triggers `superpowers:writing-plans`, breaking tasks into 2-5 minute micro-steps |

After completion, run:

```bash
openspec validate --all --json
# → A local git hook is already set up as pre-commit, automatically validating on commit
```

---

### Step 3: Apply (Implementation Phase)

```bash
/opsx:apply
```

This triggers the steps in [schema.yaml](./schema.yaml) `apply.instruction`:

#### 3-0. Pre-flight — Commit change artifacts to the current branch first

Before creating the worktree, confirm that `openspec/changes/<name>/` is tracked on the current branch. If still untracked (`git status --porcelain` output contains `??`), **commit only that change directory** (do not use `git add -A`) as `docs(openspec): scaffold <name> change`.

**Why this step is needed**: The worktree branches off from the current branch; if the change directory is still untracked on main, merging the worktree back to main later will hit an "untracked files would be overwritten by merge" error. This step separates "planning phase artifacts" and "implementation phase artifacts" into two commits, ensuring the main branch never has a drifting untracked copy.

#### 3-1. Workspace — Invoke `superpowers:using-git-worktrees`

- Creates an isolated workspace at `.worktrees/<change-name>/`
- Switches to a new branch
- Runs project setup, confirms clean test baseline

#### 3-2. Executor — Invoke `superpowers:subagent-driven-development` (2a default path)

- Main agent reads plan.md, dispatches a **fresh subagent** for each micro-task
- Each subagent automatically:
  - **Enforces TDD** (`superpowers:test-driven-development` triggered transitively)
    - Write a failing test first
    - Watch it fail
    - Write the minimum code to make it pass
    - No test before production code? Delete and redo
  - **Per-task code review** (`superpowers:requesting-code-review` triggered transitively)
    - Spec compliance review (does it match the plan?)
    - Code quality review (any smells?)
    - Critical issues block progress
- Updates `tasks.md` checkboxes as coarse tasks complete
- After all tasks finish, runs a final code review on the entire implementation

> **2b fallback**: Only use `superpowers:executing-plans` when the current platform lacks subagent support. Claude Code has subagents, so always use 2a. If forced to use 2b, you must manually maintain TDD discipline and invoke `superpowers:requesting-code-review`.

#### 3-3. Receipt — Write `apply.md`

Before invoking verify, the executor writes a minimal `apply.md` receipt per `openspec/schemas/superspec/templates/apply.md`: change name, iteration counter (1 on first apply, incremented on re-entry), applied-at timestamp, executor identity, worktree path, branch, commit range, and `X of Y` tasks completed. This is the v2 DAG artifact that satisfies `verify.requires: [apply]`. If `apply.md` already exists in the change directory, read its `Iteration:` field, increment by one, and overwrite the file.

#### 3-4. Verification — Invoke `openspec-verify-change` (produces `verify.md`)

5 checks:

1. **Structural validation**: `openspec validate --all --json` all PASS
2. **Task completion**: All `- [ ]` in `tasks.md` changed to `- [x]`
3. **Delta spec sync state**: Has `changes/<name>/specs/` been synced to `openspec/specs/`?
4. **Design / specs coherence**: Spot-check that design decisions and spec requirements are consistent (non-blocking warning)
5. **Implementation signal**: No unstaged files in the worktree

If any check fails, go back to the corresponding artifact, fix it, and re-run verify.

#### 3-5. Completion — Invoke `superpowers:finishing-a-development-branch`

- Confirm all tests are green
- Present options: merge / PR / keep branch / discard
- Clean up the worktree

#### 3-6. Retrospective (Recommended, Non-blocking)

**Not a mandatory step, but strongly recommended**: Before archiving, produce a `retrospective.md` in the change directory. The retrospective is a "self-review" of the entire change — it captures things the diff cannot show: why a decision was made, what surprised you, and which lessons learned are worth promoting to long-term memory.

**Why it's worth doing**: Every retrospective improves the quality of the next change. Without retros, blind spots accumulate repeatedly; with retros, both the team and the AI learn from each execution.

Recommended 6 sections (evidence-first — each claim cites a commit / file / quantifiable fact):

1. **Wins** — What went well (with commit / test evidence)
2. **Misses** — What didn't go well (🔴 blocking / 🟡 painful / 📌 nit)
3. **Plan deviations** — Which tasks changed in scope, and why
4. **Skill / workflow compliance** — Which skills were actually used, which were deliberately skipped (with reasons)
5. **Surprises** — Which original assumptions turned out to be wrong
6. **Promote candidates** — Lessons worth promoting to long-term memory / CLAUDE.md / schema or skill updates (categorize each, so insights don't quietly die in the archive)

If `workflow-retrospective` skill is available in the environment, it automates evidence collection; otherwise write manually. For trivial single-commit fixes where the overhead exceeds the value, this can be skipped.

---

### Step 4: Archive

```bash
/opsx:archive my-feature
```

- Validates + checks task completion (incomplete tasks warn but don't block)
- Syncs delta specs back to `openspec/specs/<capability>/spec.md`
  - Order: RENAMED → REMOVED → MODIFIED → ADDED
  - If already manually synced, use `--skip-specs`
- Moves `changes/my-feature/` to `changes/archive/YYYY-MM-DD-my-feature/`
- History is frozen; the unix timeline is treated as the source of truth

---

## 5. CLI Cheat Sheet

| Scenario | Command |
|---|---|
| **After first clone** | `bash scripts/install-git-hooks.sh` |
| New change (interactive, step by step) | `/opsx:new <name> --schema sdd-plus-superpowers` then `/opsx:continue` several times |
| New change (auto-fill all artifacts at once) | `/opsx:ff <name>` |
| Resume an interrupted change | `/opsx:continue <name>` |
| Enter implementation | `/opsx:apply <name>` |
| Manual verify | `/opsx:verify <name>` |
| Archive | `/opsx:archive <name>` |
| Use the native OpenSpec schema (skip brainstorm) | `/opsx:new <name> --schema spec-driven` |
| View all project schemas | `openspec schemas` |
| View current change progress | `openspec status --change <name> --json` |
| List active changes | `openspec list` |
| Full project validation | `openspec validate --all --json` |

---

## 6. Elegant Design Choices in the Integration (5 Worth Remembering)

### 1. Output redirection

Superpowers' brainstorming normally writes to `docs/superpowers/specs/`, and writing-plans writes to `docs/superpowers/plans/`. Our artifact instructions **override this behavior** by injecting "write to the change directory" directives via prompt context. No superpowers source code is changed, and no OpenSpec CLI changes are needed.

### 2. Schema-level vs prompt-level integration

The integration happens entirely in the `instruction` field (pure prompt). If superpowers upgrades a skill's behavior, we **don't need to touch the schema at all**. The schema.yaml only needs updating if a skill is renamed or removed.

### 3. Making transitive dependencies explicit

TDD and code-review are originally hidden inside subagent-driven-development (only visible in its SKILL.md). The schema **explicitly lists** these two transitive activations in the apply step 2a instruction, so readers can immediately understand "what exactly happens during the apply phase."

### 4. Honest fallback path labeling

2b (executing-plans) exists but is labeled as a "platforms without subagent support" fallback, citing the official superpowers SKILL.md L14 verbatim. We don't invent custom rules like "use 2b for small changes."

### 5. Apply is a real artifact, not a hidden phase (v2)

In schema v1, `verify.requires: [plan]` was a deliberate lie — the comment said "this edge exists only for the graph; actually verify must run after apply." That was unenforceable: agents read the DAG, saw verify reachable as soon as plan was done, and ran verify before apply.

In v2, `apply` is promoted to a real artifact (generating `apply.md`, a minimal receipt) so `verify.requires: [apply]` is honest. The schema graph and the actual sequencing now agree. The top-level `apply:` block is preserved so `/opsx:apply` continues to surface the canonical worktree+subagent instruction body — the apply artifact's own instruction is a short redirect to keep a single source of truth.

This change also unlocks the documented apply → verify → repeat convergence loop, since `verify.md` outcomes can now feed cleanly back into a re-run of apply with an incremented iteration counter. See `docs/workflow-details.md` for the loop pattern.

### Migration from schema v1

If a project pinned to schema v1 has in-flight changes whose `verify.md` was authored before v2 landed but no `apply.md` exists in the change directory, `/opsx:verify` will report the change as blocked under v2 (missing required artifact `apply`).

Migration: author a minimal `apply.md` by hand from `openspec/schemas/superspec/templates/apply.md` — set `Iteration: 1`, fill the worktree path, branch, commit range, and task counts from the existing implementation, then re-run `/opsx:verify`. This is a one-time migration cost per in-flight change; no archived changes are affected.

---

## 7. Recommended Snapshot Section for Projects Adopting This Schema

It's recommended that every project adopting `sdd-plus-superpowers` maintains a snapshot in the following format in the relevant repo document, so new members can see "what this repo currently looks like" at a glance during onboarding:

```markdown
## Project Status (snapshot: YYYY-MM-DD)

- **OpenSpec CLI**: v<version>
- **Schema**: `sdd-plus-superpowers` v<n>
- **Specs (bounded-context granularity)**: <n> domains exist, <n> domains reserved for lazy backfill
  - Existing: `<capability-a>` / `<capability-b>` / ...
  - Reserved: `<capability-c>` / ...
- **Automation**: <what openspec commands pre-commit / CI runs>
- **Superpowers plugin**: `superpowers@<version>` installed at `<path>`, this integration uses N skills
```

> This snapshot section will become stale over time; for authoritative state, query live with `openspec list` + `openspec schemas`.

---

## 8. The Most Important Takeaway

The core value of the integration is not "chaining many skills together" — it is:

> **Connecting "requirements alignment" (OpenSpec) with "rigorous execution" (Superpowers) so that the entire path from "what we want to do" to "code that has passed TDD + code review" is fully traceable, reproducible, and auditable for a single change.**

The break points in traditional workflows are:

- Requirements live in Slack / conversations → during apply the LLM works from memory → doesn't match the spec
- Or: spec is written in Confluence → code is in the repo → the two drift apart

The two-layer constraint of sdd-plus-superpowers solves this problem:

1. **OpenSpec's delta spec governance** → ensures "what to do" doesn't drift
2. **Superpowers' subagent-driven + TDD + review** → ensures "what was done" has quality discipline

Put another way: OpenSpec is responsible for **rescuing requirements from conversations**, and Superpowers is responsible for **rescuing discipline from human willpower**. Only when combined do they form complete spec-driven development.

---

## Related Documents

- [schema.yaml](./schema.yaml) — Machine-readable definition of this schema
- [README.md](./README.md) — Design motivation and high-level overview of the schema
- [templates/](./templates/) — Markdown templates for each artifact
- [../../specs/README.md](../../specs/README.md) — Capability domain classification guide
- [openspec-conventions spec](https://github.com/Fission-AI/OpenSpec/blob/main/openspec/specs/openspec-conventions/spec.md) — Official OpenSpec conventions
- [obra/superpowers](https://github.com/obra/superpowers) — Superpowers skill source
