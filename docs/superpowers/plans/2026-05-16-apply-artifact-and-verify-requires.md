# Apply Artifact + verify.requires Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Promote `apply` to a real DAG node in the Superspec schema so `verify.requires: [apply]` honestly gates `/opsx:verify` on a completed implementation; bump schema to v2; document the apply → verify → repeat convergence loop.

**Architecture:** Hybrid approach — `apply` is added to `artifacts:` (with `generates: apply.md`, `requires: [plan]`) AND the existing top-level `apply:` block is kept (so `/opsx:apply` CLI gating behavior is unchanged). The `apply:` block remains the canonical text surface for `/opsx:apply`; the apply artifact's own instruction is a short redirect (inverse pointer) to avoid drift. `apply.md` is a minimal receipt (no quality verdict — that stays in `verify.md`) and overwritten on each iteration with an `Iteration` counter. The convergence loop is documented as a recommended pattern; no auto-runner command is built in this change.

**Tech Stack:** YAML schema (consumed by OpenSpec's TypeScript zod parser at Fission-AI/OpenSpec), Markdown templates and docs. No code is executed; "tests" are structural validations (YAML parse, hand-walk against the OpenSpec zod schema rules, grep checks for doc/schema consistency).

**Reference spec:** `docs/superpowers/specs/2026-05-16-apply-artifact-and-verify-requires-design.md`

**Branch:** `spec/apply-artifact-and-verify-requires` (already created from `main`; spec already committed at `a99e9c9`).

---

## File map

| File | Action | Responsibility |
|---|---|---|
| `openspec/schemas/superspec/templates/apply.md` | **create** | Minimal apply-receipt template (referenced by new apply artifact via `template:` field) |
| `openspec/schemas/superspec/templates/verify.md` | modify | Add `Iteration` line + loop-return guidance |
| `openspec/schemas/superspec/schema.yaml` | modify | Bump version 1→2; add `apply` artifact entry; change `verify.requires` to `[apply]`; trim apply: block instruction body to add receipt-writing step; update top-level description |
| `openspec/schemas/superspec/INTEGRATION.md` | modify | Section 3 (DAG diagram) redraw with apply as a real node; Section 6 (Design Choice #5) rewrite; new "Migration from v1" subsection |
| `openspec/schemas/superspec/README.md` | modify | Workflow overview diagram and "Differences from spec-driven" table |
| `docs/workflow.md` | modify | Phase 3 primary outputs now include `apply.md` |
| `docs/workflow-details.md` | modify | Step 7 outputs include `apply.md`; Step 8 drops the "graph-only" comment; **new "Convergence loop" subsection** with ASCII diagram + termination rules + iteration ceiling 5 |
| `README.md` (root) | modify | `Schema version 1` → `Schema version 2`; brief Quick-start note that `/opsx:apply` writes `apply.md` |
| `docs/assets/superspec-phases-flowchart.svg` | **unchanged** | Phase boundaries are unchanged |

Each commit ends a task. Commits are sequential on `spec/apply-artifact-and-verify-requires`.

---

## Task 1: Create `apply.md` template

**Files:**
- Create: `openspec/schemas/superspec/templates/apply.md`

- [ ] **Step 1: Confirm the file does not yet exist**

Run: `ls openspec/schemas/superspec/templates/`
Expected: `brainstorm.md  design.md  plan.md  proposal.md  spec.md  tasks.md  verify.md` (no `apply.md`).

- [ ] **Step 2: Create the template**

Write exactly the following to `openspec/schemas/superspec/templates/apply.md`:

```markdown
# Apply Receipt

> Generated at the end of the apply phase to mark code-implementation
> complete and provide verify with the state it needs.
> Overwritten on each apply iteration; iteration counter grows.

**Change**: `<change-name>`
**Iteration**: `1`
**Applied at**: `YYYY-MM-DD HH:mm`
**Executor**: `subagent-driven-development` | `executing-plans`

---

## Workspace

- **Worktree**: `.worktrees/<change-name>/`
- **Branch**: `<branch-name>`

---

## Commits

- **Range**: `<from-sha>..<to-sha>` (or `none` if nothing committed yet)
- **Count**: `<n>`

---

## Tasks

- **Completed**: `X of Y` checkboxes in tasks.md flipped to `- [x]`
- **Remaining**: `<list ids of unfinished tasks, or "none">`

---

## Next step

`<e.g., "Run /opsx:verify" or "Re-run apply to address <issue>">`
```

The Quality verdict belongs in `verify.md`, not here — do not add a "Final review" section.

- [ ] **Step 3: Confirm the file ends with a single newline (pre-commit hook will enforce this)**

Run: `tail -c 1 openspec/schemas/superspec/templates/apply.md | xxd`
Expected: ends with `0a` (single LF).

- [ ] **Step 4: Confirm no trailing whitespace**

Run: `grep -nE ' +$' openspec/schemas/superspec/templates/apply.md`
Expected: no output (no matches).

- [ ] **Step 5: Commit**

```bash
git add openspec/schemas/superspec/templates/apply.md
git commit -m "feat(schema): add apply.md template for v2 apply artifact

Minimal receipt structure capturing iteration counter, workspace,
commit range, and task completion. Verification outcome stays in
verify.md."
```

---

## Task 2: Update `verify.md` template (Iteration + loop guidance)

**Files:**
- Modify: `openspec/schemas/superspec/templates/verify.md` (around lines 7–9 and the closing Overall Decision section)

- [ ] **Step 1: Read the current template**

Run: `cat openspec/schemas/superspec/templates/verify.md`
Expected: 86 lines starting `# Verification Report`. Current state has `**Change**`, `**Verified at**`, `**Verifier**` near the top, no `Iteration` field.

- [ ] **Step 2: Add the `Iteration` line directly after `**Verified at**`**

Use Edit to replace:

```markdown
**Change**: `<change-name>`
**Verified at**: `YYYY-MM-DD HH:mm`
**Verifier**: `<who / which agent>`
```

with:

```markdown
**Change**: `<change-name>`
**Verified at**: `YYYY-MM-DD HH:mm`
**Iteration**: `<integer; copy from apply.md>`
**Verifier**: `<who / which agent>`
```

- [ ] **Step 3: Replace the "Next step" closing block with loop-return guidance**

Use Edit to replace:

```markdown
**Next step**:

<describe the next action>
```

with:

```markdown
**Next step**:

<describe the next action>

> **Convergence loop reminder**: If you marked FAIL with items
> fixable by a code change, return to the apply artifact and re-run
> `/opsx:apply`, then re-run `/opsx:verify` — this overwrites both
> apply.md and verify.md with the next iteration. If the iteration
> counter has exceeded 5, stop the loop and report the situation to
> the user instead of continuing.
```

- [ ] **Step 4: Verify diff is exactly two hunks**

Run: `git diff openspec/schemas/superspec/templates/verify.md | head -60`
Expected: one `+ **Iteration**:` line near the top and one `+ > **Convergence loop reminder**: …` block near the bottom; no other changes.

- [ ] **Step 5: Trailing whitespace + final newline check**

Run: `grep -nE ' +$' openspec/schemas/superspec/templates/verify.md && tail -c 1 openspec/schemas/superspec/templates/verify.md | xxd`
Expected: no trailing whitespace; file ends with `0a`.

- [ ] **Step 6: Commit**

```bash
git add openspec/schemas/superspec/templates/verify.md
git commit -m "feat(schema): add Iteration field and loop guidance to verify.md

Verify.md now tracks the same iteration counter as apply.md and tells
the agent how to return to apply on code-fixable FAIL outcomes."
```

---

## Task 3: Modify `schema.yaml` (version, apply artifact, verify.requires, apply: block)

This is the largest change. Break it into sub-steps so each edit is reviewable.

**Files:**
- Modify: `openspec/schemas/superspec/schema.yaml` (lines 1–10 description; 162–211 verify artifact; 213–343 apply: block; new apply artifact inserted between plan and verify)

- [ ] **Step 1: Bump version and update top-level description**

Use Edit on `openspec/schemas/superspec/schema.yaml` to replace:

```yaml
name: SuperSpec
version: 1
description: >
  Spec-driven workflow integrated with Superpowers skills.
  brainstorm → proposal → specs → tasks → plan → verify.
  design is optional (produced from brainstorm but not required by tasks).
  Apply phase uses git worktrees + subagent-driven-development
  (brings TDD and code-review transitively). executing-plans is
  documented only as a fallback for platforms without subagent support.
```

with:

```yaml
name: SuperSpec
version: 2
description: >
  Spec-driven workflow integrated with Superpowers skills.
  brainstorm → proposal → specs → tasks → plan → apply → verify.
  design is optional (produced from brainstorm but not required by tasks).
  Apply is both a DAG artifact (generates apply.md, a minimal receipt)
  and a top-level apply: phase block (canonical /opsx:apply instruction
  body). Verify requires apply, so /opsx:verify cannot run before
  /opsx:apply has executed. Apply uses git worktrees +
  subagent-driven-development (brings TDD and code-review transitively).
  executing-plans is documented only as a fallback for platforms without
  subagent support. v2: apply was promoted from a phase-only block to
  a DAG artifact; verify.requires now points at apply rather than plan.
```

- [ ] **Step 2: Insert the new `apply` artifact between `plan` and `verify`**

Locate the closing line of the `plan` artifact (line 161 in v1: the `- tasks` under `requires:`). Use Edit to replace the boundary between plan and verify:

```yaml
    requires:
      - tasks

  - id: verify
    generates: verify.md
```

with:

```yaml
    requires:
      - tasks

  - id: apply
    generates: apply.md
    description: Implementation receipt confirming the apply phase completed
    template: apply.md
    instruction: |
      This artifact is generated by `/opsx:apply`, not `/opsx:continue`.
      If you reached this via `/opsx:continue`, exit and run `/opsx:apply`
      instead. The canonical implementation instructions live in the
      top-level `apply:` phase block below.

      apply.md is a minimal receipt written at the end of the apply
      phase: change name, iteration counter, applied-at timestamp,
      executor, worktree path, branch, commit range, tasks completed
      X of Y, and remaining tasks. Verification outcome is recorded in
      verify.md (Overall Decision section), not here.

      If apply.md already exists when re-entering apply, read its
      `Iteration:` field and increment by one; otherwise default to
      iteration 1. Overwrite the file each time.
    requires:
      - plan

  - id: verify
    generates: verify.md
```

- [ ] **Step 3: Update the `verify` artifact's `requires` and instruction**

Use Edit to replace the entire verify artifact block (the block whose `instruction: |` currently starts with "Use the Skill tool to invoke **openspec-verify-change**" and whose final lines are `requires:\n      - plan`). The new block must be:

```yaml
  - id: verify
    generates: verify.md
    description: Post-implementation verification against specs, design, and tasks
    template: verify.md
    instruction: |
      Use the Skill tool to invoke **openspec-verify-change** (the
      `/opsx:verify` slash command is its user-facing equivalent).

      apply.md MUST exist before this artifact runs — the DAG enforces
      this. If apply.md is missing, run `/opsx:apply` first. After every
      apply iteration (whether the first or a re-run), verify should be
      invoked so the convergence loop can decide whether more apply
      iterations are needed.

      verify.md is overwritten each run. Copy the `Iteration:` value
      from apply.md so verify and apply stay in lockstep.

      The verify step MUST perform the following checks and record
      results in verify.md using the template:

      1. **Structural validation**: Run `openspec validate --all --json`
         in the repository root and confirm every item returns
         `"valid": true`. If any item fails, record the issues and
         return to fix the underlying artifact before proceeding.

      2. **Task completion**: Confirm every checkbox in tasks.md is
         `- [x]`. For any `- [ ]` remaining, document the reason
         (e.g. manual / out-of-scope / blocked) and whether it blocks
         archive.

      3. **Delta spec sync state**: For each directory under
         `openspec/changes/<name>/specs/`, compare against the
         corresponding `openspec/specs/<capability>/spec.md` and
         record one of:
         - ✓ Already synced
         - ✗ Needs sync (list the capabilities)
         - N/A (no delta specs produced)

      4. **Design/specs coherence**: Spot-check that design.md
         decisions reference or align with the requirements listed
         in specs/. Record any drift as a warning (non-blocking).

      5. **Implementation signal**: Confirm all code changes are
         committed (no unstaged files in the worktree). Cite the
         commit range if known.

      Recommended convergence loop:
      - PASS → proceed to finishing-a-development-branch + archive.
      - PASS_WITH_WARNINGS → proceed; record warnings.
      - FAIL with code-fixable items → return to apply, re-run, re-verify.
      - FAIL with artifact-level items (e.g. spec drift) → fix the
        offending artifact, then re-enter apply.
      - Iteration > 5 → stop looping; report to the user.

      See docs/workflow-details.md for the full convergence pattern.

      If `openspec-verify-change` skill is unavailable, fall back to
      running the 5 checks manually and recording results in verify.md.
    requires:
      - apply
```

Note: the misleading "IMPORTANT timing note" paragraph from the v1 verify instruction is gone (the DAG now enforces the ordering, so the comment is obsolete). The `requires:` changes from `- plan` to `- apply`. The five checks are textually preserved.

- [ ] **Step 4: Append the receipt-writing step to the `apply:` block instruction**

Locate the `apply:` block at the bottom of `schema.yaml`. Find step 3 (Verification) and step 4 (Completion). Insert a new step between them so the receipt is written *before* invoking finishing-a-development-branch. Use Edit to replace:

```yaml
    3. **Verification**: When all tasks are done, produce the
       `verify` artifact by invoking **openspec-verify-change**
       to check completeness, correctness, and coherence.
       Re-run verify until all blocking issues are resolved.

    4. **Completion**: Once verify.md shows no blocking issues,
       use the Skill tool to invoke
       **superpowers:finishing-a-development-branch** for branch
       cleanup, PR creation, etc.
```

with:

```yaml
    3. **Receipt**: Write `apply.md` per `templates/apply.md`. If a
       previous `apply.md` exists in the change directory, read its
       `Iteration:` field and increment by one; otherwise set
       iteration to `1`. Overwrite the file with current state:
       worktree path, branch, commit range, tasks completed X of Y,
       remaining tasks, executor identity, and an ISO timestamp.
       The receipt is the artifact that satisfies the DAG so
       `verify` becomes reachable.

    4. **Verification**: Once the receipt exists, produce the
       `verify` artifact by invoking **openspec-verify-change**
       to check completeness, correctness, and coherence. The DAG
       blocks `verify` until apply.md exists.

       Convergence loop: if verify reports FAIL with items fixable
       by a code change, return to step 2 (Executor), re-run with
       the same iteration mechanics (apply.md is overwritten,
       iteration counter increments), then re-run verify. Stop and
       report to the user if iteration > 5.

    5. **Completion**: Once verify.md shows no blocking issues,
       use the Skill tool to invoke
       **superpowers:finishing-a-development-branch** for branch
       cleanup, PR creation, etc.
```

- [ ] **Step 5: Validate the YAML parses**

Run:

```bash
python3 -c "import yaml,sys; d=yaml.safe_load(open('openspec/schemas/superspec/schema.yaml')); print('version:', d['version']); print('artifact ids:', [a['id'] for a in d['artifacts']]); print('verify.requires:', next(a['requires'] for a in d['artifacts'] if a['id']=='verify')); print('apply (artifact).requires:', next(a['requires'] for a in d['artifacts'] if a['id']=='apply')); print('apply (artifact).generates:', next(a['generates'] for a in d['artifacts'] if a['id']=='apply')); print('apply: block requires:', d['apply']['requires']); print('apply: block tracks:', d['apply']['tracks'])"
```

Expected output:

```
version: 2
artifact ids: ['brainstorm', 'proposal', 'design', 'specs', 'tasks', 'plan', 'apply', 'verify']
verify.requires: ['apply']
apply (artifact).requires: ['plan']
apply (artifact).generates: apply.md
apply: block requires: ['plan']
apply: block tracks: tasks.md
```

If any line differs, fix the corresponding Edit and re-run before moving on.

- [ ] **Step 6: Hand-walk the OpenSpec zod rules (no openspec CLI installed locally)**

Run:

```bash
python3 << 'PY'
import yaml
d = yaml.safe_load(open('openspec/schemas/superspec/schema.yaml'))
ids = [a['id'] for a in d['artifacts']]
assert len(set(ids)) == len(ids), f"duplicate ids: {ids}"
for a in d['artifacts']:
    for k in ('id','generates','description','template'):
        assert a.get(k), f"missing {k} in {a.get('id')}"
    for r in a.get('requires', []):
        assert r in ids, f"{a['id']}.requires references unknown id: {r}"
def has_cycle(adj, start, seen, stack):
    seen.add(start); stack.add(start)
    for d2 in adj.get(start, []):
        if d2 not in seen:
            if has_cycle(adj, d2, seen, stack): return True
        elif d2 in stack:
            return True
    stack.discard(start)
    return False
adj = {a['id']: a.get('requires', []) for a in d['artifacts']}
seen, stack = set(), set()
for n in ids:
    if n not in seen and has_cycle(adj, n, seen, stack):
        raise SystemExit(f"cycle reachable from {n}")
print("OK: zod-equivalent structural checks pass")
PY
```

Expected: `OK: zod-equivalent structural checks pass`. Any AssertionError or `cycle reachable from …` is a blocker — fix the schema before continuing.

- [ ] **Step 7: Confirm trailing whitespace + final newline**

Run: `grep -nE ' +$' openspec/schemas/superspec/schema.yaml; tail -c 1 openspec/schemas/superspec/schema.yaml | xxd`
Expected: no trailing whitespace; file ends with `0a`.

- [ ] **Step 8: Commit**

```bash
git add openspec/schemas/superspec/schema.yaml
git commit -m "feat(schema): promote apply to v2 DAG artifact, gate verify on apply

- Bump schema version 1 -> 2 with updated description
- Add apply artifact (generates apply.md, requires plan); artifact
  instruction is a short redirect to the apply: phase block
- Change verify.requires from [plan] to [apply]
- Rewrite verify instruction: drop the obsolete \"graph-only\"
  comment, add iteration mechanics, add the convergence loop
- Insert a new \"Receipt\" step in the apply: block so apply.md is
  written before verify is invoked

OpenSpec CLI behavior preserved: /opsx:apply still surfaces the
apply: block instruction; only the DAG dependency changes."
```

---

## Task 4: Update `INTEGRATION.md` (DAG diagram + design choice #5 + migration note)

**Files:**
- Modify: `openspec/schemas/superspec/INTEGRATION.md` (lines 38–86 DAG diagram + key points; lines 281–284 design choice #5; new "Migration from v1" subsection)

- [ ] **Step 1: Redraw Section 3 DAG diagram with apply as a real node**

Use Edit to replace the existing code-block diagram (`┌──────────────┐ │  brainstorm  │ …`) and the "Key points" bullets at lines 38–86 with:

````markdown
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
````

- [ ] **Step 2: Rewrite design choice #5 (Section 6)**

Use Edit to replace:

```markdown
### 5. Verify is a leaf in the schema graph but runs after apply

`verify`'s `requires: [plan]` is only there to keep the schema graph complete; its instruction explicitly states "**MUST run on a completed implementation, NOT during planning**." This is a deliberate mismatch between the OpenSpec DAG and actual sequencing, so that `openspec status` can display verify progress.
```

with:

```markdown
### 5. Apply is a real artifact, not a hidden phase (v2)

In schema v1, `verify.requires: [plan]` was a deliberate lie — the comment said "this edge exists only for the graph; actually verify must run after apply." That was unenforceable: agents read the DAG, saw verify reachable as soon as plan was done, and ran verify before apply.

In v2, `apply` is promoted to a real artifact (generating `apply.md`, a minimal receipt) so `verify.requires: [apply]` is honest. The schema graph and the actual sequencing now agree. The top-level `apply:` block is preserved so `/opsx:apply` continues to surface the canonical worktree+subagent instruction body — the apply artifact's own instruction is a short redirect to keep a single source of truth.

This change also unlocks the documented apply → verify → repeat convergence loop, since `verify.md` outcomes can now feed cleanly back into a re-run of apply with an incremented iteration counter. See `docs/workflow-details.md` for the loop pattern.
```

- [ ] **Step 3: Add a "Migration from v1" subsection at the end of Section 6 (before Section 7)**

Use Edit to insert a new subsection. Find the divider line that immediately precedes `## 7. Recommended Snapshot Section …` and replace it with:

```markdown
### Migration from schema v1

If a project pinned to schema v1 has in-flight changes whose `verify.md` was authored before v2 landed but no `apply.md` exists in the change directory, `/opsx:verify` will report the change as blocked under v2 (missing required artifact `apply`).

Migration: author a minimal `apply.md` by hand from `openspec/schemas/superspec/templates/apply.md` — set `Iteration: 1`, fill the worktree path, branch, commit range, and task counts from the existing implementation, then re-run `/opsx:verify`. This is a one-time migration cost per in-flight change; no archived changes are affected.

---

## 7. Recommended Snapshot Section for Projects Adopting This Schema
```

(In other words: replace the `---\n\n## 7.` boundary with the migration content + the same `---\n\n## 7.` boundary.)

- [ ] **Step 4: Verify the three edits landed**

Run:

```bash
grep -nE "apply.*DAG node|Migration from schema v1|Apply is a real artifact" openspec/schemas/superspec/INTEGRATION.md
```

Expected: at least three matches — one for the new key-points bullet, one for the new design choice #5 heading, and one for the migration section.

Run: `grep -n "requires: \[plan\]" openspec/schemas/superspec/INTEGRATION.md`
Expected: zero matches in any "verify" context. (May still appear in unrelated contexts; if a verify-related match remains, it's a missed edit.)

- [ ] **Step 5: Trailing whitespace + final newline**

Run: `grep -nE ' +$' openspec/schemas/superspec/INTEGRATION.md; tail -c 1 openspec/schemas/superspec/INTEGRATION.md | xxd`
Expected: no trailing whitespace; file ends with `0a`.

- [ ] **Step 6: Commit**

```bash
git add openspec/schemas/superspec/INTEGRATION.md
git commit -m "docs(schema): update INTEGRATION for v2 apply artifact

Redraw the DAG diagram with apply as a real node, rewrite design
choice #5 to match the new honest ordering, and add a Migration
from v1 subsection for projects with in-flight changes."
```

---

## Task 5: Update the schema's `README.md`

**Files:**
- Modify: `openspec/schemas/superspec/README.md` (lines 28–45 workflow overview + Differences table)

- [ ] **Step 1: Replace the workflow overview diagram**

Use Edit to replace:

```text
brainstorm ──→ proposal ──→ specs ──→ tasks ──→ plan
                  │                     ↑
                  └──→ design ──────────┘
```

with:

```text
brainstorm ──→ proposal ──→ specs ──→ tasks ──→ plan ──→ apply ──→ verify
                  │                     ↑
                  └──→ design ──────────┘
```

- [ ] **Step 2: Update the "Differences from spec-driven" table**

Use Edit to replace:

```markdown
| | spec-driven | sdd-plus-superpowers |
|---|---|---|
| Starting point | proposal (written manually) | **brainstorm** (invokes brainstorming skill) |
| Endpoint | tasks (coarse-grained) | **plan** (micro TDD steps) |
| apply requires | tasks | **plan** |
| apply method | Standard task-by-task | **worktree + subagent-driven-development** |
| Additional artifacts | — | brainstorm, plan |
```

with:

```markdown
| | spec-driven | sdd-plus-superpowers (v2) |
|---|---|---|
| Starting point | proposal (written manually) | **brainstorm** (invokes brainstorming skill) |
| Endpoint | tasks (coarse-grained) | **verify** (post-implementation report; apply is the executable midpoint) |
| apply requires | tasks | **plan** |
| apply method | Standard task-by-task | **worktree + subagent-driven-development** |
| Additional artifacts | — | brainstorm, plan, **apply (receipt)** |
| verify requires | tasks | **apply** (v2 — was plan in v1) |
```

- [ ] **Step 3: Add a short note to the "Why brainstorm Is an Artifact, Not a Hook" decision log**

Append a new entry to the Design Decision Log at the bottom of the file (immediately before the "Fallback Strategy" subsection):

```markdown
### Why apply Is Both an Artifact and a Phase Block (v2)

The OpenSpec CLI builds its DAG strictly from the `artifacts:` list, and the `ApplyPhase` zod schema has no `generates` field — so the apply phase alone cannot mark itself complete or be referenced by another artifact's `requires`. In v1 this manifested as `verify.requires: [plan]` plus an inline comment saying "actually verify needs apply, but the schema can't say so." Agents predictably ran verify before apply.

v2 solves this by representing apply twice: a real artifact (`generates: apply.md`, `requires: [plan]`) so `verify.requires: [apply]` is honest, *and* the existing `apply:` top-level block so `/opsx:apply` CLI behavior is unchanged (the handler reads `schema.apply.instruction` and would not see the artifact's instruction). To avoid drift, the canonical body lives in the `apply:` block; the apply artifact's instruction is a short redirect.
```

- [ ] **Step 4: Verify edits**

Run:

```bash
grep -n "apply (receipt)\|verify ──→\|Why apply Is Both" openspec/schemas/superspec/README.md
```

Expected: three matches.

- [ ] **Step 5: Trailing whitespace + final newline**

Run: `grep -nE ' +$' openspec/schemas/superspec/README.md; tail -c 1 openspec/schemas/superspec/README.md | xxd`
Expected: no trailing whitespace; file ends with `0a`.

- [ ] **Step 6: Commit**

```bash
git add openspec/schemas/superspec/README.md
git commit -m "docs(schema): update schema README workflow + table for v2

Workflow overview now shows apply → verify on the main path. Diff
table calls out the new \"apply requires plan\" / \"verify requires
apply\" relationship. Adds a decision log entry explaining why
apply lives in both artifacts: and apply:."
```

---

## Task 6: Update `docs/workflow.md`

**Files:**
- Modify: `docs/workflow.md` (Phase 3 row in the At a Glance table; Phase 3 phase-summary subsection on lines around 38–42)

- [ ] **Step 1: Confirm current state**

Run: `grep -n "Code Implementation\|apply\.md" docs/workflow.md`
Expected: "Code Implementation" appears in the table (line ~19) and the heading "### 3. Code Implementation"; "apply.md" appears nowhere.

- [ ] **Step 2: Update the Phase 3 phase summary**

Use Edit to replace:

```markdown
### 3. Code Implementation

Implementation runs through the Superpowers execution loop: isolated worktree, subagent-driven development, TDD, code review, and task checkbox updates.

Primary outputs: code, tests, commits, and completed `tasks.md` checkboxes.
```

with:

```markdown
### 3. Code Implementation

Implementation runs through the Superpowers execution loop: isolated worktree, subagent-driven development, TDD, code review, and task checkbox updates. At the end of the phase, an `apply.md` receipt is written so the DAG can gate `verify` on apply having completed.

Primary outputs: code, tests, commits, completed `tasks.md` checkboxes, and `apply.md` (the minimal completion receipt).
```

- [ ] **Step 3: Update the See Also list (if it references "tasks completed" specifically)**

Run: `grep -n "tasks completed\|primary output" docs/workflow.md`
Expected: only the line we just modified contains "primary output"; no further edits needed.

- [ ] **Step 4: Trailing whitespace + final newline**

Run: `grep -nE ' +$' docs/workflow.md; tail -c 1 docs/workflow.md | xxd`
Expected: no trailing whitespace; file ends with `0a`.

- [ ] **Step 5: Commit**

```bash
git add docs/workflow.md
git commit -m "docs: note apply.md as a phase 3 primary output

workflow.md phase summary for Code Implementation now mentions the
v2 apply.md receipt alongside code, tests, and tasks.md checkboxes."
```

---

## Task 7: Update `docs/workflow-details.md` (Step 7 + Step 8 + new Convergence Loop subsection)

**Files:**
- Modify: `docs/workflow-details.md` (Step 7 around lines 174–197, Step 8 around lines 205–225, new subsection inserted between Phase 3 and Phase 4 boundaries around line 199)

- [ ] **Step 1: Update Step 7 (apply) — add apply.md to outputs and mention the receipt**

Use Edit to replace the existing "**What the step does.**" paragraph in Step 7 with a version that adds a 4th item:

Find:

```markdown
**What the step does.** The apply phase chains four Superpowers skills (with two more triggered transitively):

1. **`using-git-worktrees`** — creates an isolated workspace at `.worktrees/<change-name>/`, switches to a new branch, runs project setup, and confirms a clean test baseline.
2. **`subagent-driven-development`** (default path, requires subagent support) — the main agent reads `plan.md` and dispatches a fresh subagent per micro-task. Each subagent transitively activates:
   - **`test-driven-development`** — write a failing test first, watch it fail, then write the minimum code to make it pass. Implementation written before a failing test is deleted and redone.
   - **`requesting-code-review`** — after each task, a code-reviewer subagent checks spec compliance and code quality. A final review runs over the whole implementation before apply concludes.
   - As coarse tasks complete, `tasks.md` checkboxes flip to `- [x]`.
3. **`finishing-a-development-branch`** — only invoked at the very end; covered in phase 9 below.
```

Replace with:

```markdown
**What the step does.** The apply phase chains four Superpowers skills (with two more triggered transitively) and writes a receipt:

1. **`using-git-worktrees`** — creates an isolated workspace at `.worktrees/<change-name>/`, switches to a new branch, runs project setup, and confirms a clean test baseline.
2. **`subagent-driven-development`** (default path, requires subagent support) — the main agent reads `plan.md` and dispatches a fresh subagent per micro-task. Each subagent transitively activates:
   - **`test-driven-development`** — write a failing test first, watch it fail, then write the minimum code to make it pass. Implementation written before a failing test is deleted and redone.
   - **`requesting-code-review`** — after each task, a code-reviewer subagent checks spec compliance and code quality. A final review runs over the whole implementation before apply concludes.
   - As coarse tasks complete, `tasks.md` checkboxes flip to `- [x]`.
3. **Receipt** — at the end of the phase, a minimal `apply.md` is written per `openspec/schemas/superspec/templates/apply.md`: iteration counter, applied-at timestamp, executor identity, worktree path, branch, commit range, and tasks completed X of Y. This is the v2 DAG artifact that gates `verify`. If `apply.md` already exists, the iteration counter is incremented.
4. **`finishing-a-development-branch`** — only invoked at the very end; covered in phase 9 below.
```

- [ ] **Step 2: Update Step 8 (verify) — drop the obsolete "graph completeness" comment**

Use Edit to replace the "**Why it's required.**" paragraph in Step 8:

Find:

```markdown
**Why it's required.** Verify closes the loop between intent and implementation. It checks that the proposal, specs, design, tasks, and the actual code still agree before the change is considered complete. Code review alone — even a thorough Superpowers review — only checks that the code matches the *plan*; it does not check that the plan still matches the *specs*.
```

Replace with:

```markdown
**Why it's required.** Verify closes the loop between intent and implementation. It checks that the proposal, specs, design, tasks, and the actual code still agree before the change is considered complete. Code review alone — even a thorough Superpowers review — only checks that the code matches the *plan*; it does not check that the plan still matches the *specs*.

As of schema v2, `verify.requires: [apply]` — the DAG genuinely gates verify on the existence of `apply.md`. In v1 the dependency was declared as `[plan]` with a comment that "actually verify needs apply"; that mismatch was a frequent source of agents running verify before apply had executed. The v2 schema removes the mismatch.
```

- [ ] **Step 3: Insert a new "Convergence loop" subsection between Phase 3 and Phase 4**

Find the line that separates Phase 3 from Phase 4:

```markdown
**Step not used / replaced and why.** Vanilla `/opsx:apply` execution is effectively overridden. OpenSpec apply can work through tasks directly, but Superspec chooses Superpowers' stricter loop with isolated worktrees, subagents, TDD, and review. The OpenSpec docs show `/opsx:apply` "working through tasks," but in this schema Superpowers is the executor.

---

## Phase 4: Spec Validation
```

Use Edit to replace it with:

````markdown
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

Termination rules (recorded in the verify instruction and the `apply.md` "Next step" field):

- **PASS** — proceed to `finishing-a-development-branch` and archive.
- **PASS_WITH_WARNINGS** — proceed; warnings are recorded for posterity but do not block.
- **FAIL with code-fixable items** — return to the apply phase, re-run, overwrite `apply.md` with iteration N+1, then re-run verify.
- **FAIL with artifact-level items** (e.g. spec drift, a requirement that is no longer satisfied by the plan) — fix the offending artifact first, then re-enter apply with iteration N+1.
- **Iteration > 5** — stop the loop and report to the user. This is a soft safeguard against non-convergence; the schema enforces nothing here, but the verify instruction tells the agent to halt the pattern.

The schema enforces only the file-existence dependency (`verify.requires: [apply]`). The iteration decision — whether to loop, stop, or escalate — is made by the agent (or, in a follow-up change, by a dedicated loop-runner command). No automated loop runner ships with v2.

---

## Phase 4: Spec Validation
````

- [ ] **Step 4: Verify the three edits**

Run:

```bash
grep -nE "iteration counter|Convergence loop \(apply|verify\.requires: \[apply\]" docs/workflow-details.md
```

Expected: at least three matches.

Run: `grep -n "graph completeness\|requires: \[plan\].*verify" docs/workflow-details.md`
Expected: zero matches.

- [ ] **Step 5: Trailing whitespace + final newline**

Run: `grep -nE ' +$' docs/workflow-details.md; tail -c 1 docs/workflow-details.md | xxd`
Expected: no trailing whitespace; file ends with `0a`.

- [ ] **Step 6: Commit**

```bash
git add docs/workflow-details.md
git commit -m "docs: document v2 apply.md receipt and the convergence loop

Step 7 (apply) gains a Receipt sub-step listing apply.md as a primary
output. Step 8 (verify) explicitly notes the new requires:[apply]
gate. New Convergence loop subsection between Phase 3 and Phase 4
explains the termination rules and the iteration-5 ceiling."
```

---

## Task 8: Update root `README.md` (tagline + Quick Start mention)

**Files:**
- Modify: `README.md` (line 10 tagline, lines ~174–187 Quick Start command listings)

- [ ] **Step 1: Bump the schema version in the tagline**

Use Edit to replace:

```markdown
  MIT licensed · Schema version 1 · Requires OpenSpec + Superpowers
```

with:

```markdown
  MIT licensed · Schema version 2 · Requires OpenSpec + Superpowers
```

- [ ] **Step 2: Add a short note to the step-by-step Quick Start**

Find the "/opsx:apply" line in the step-by-step flow:

```markdown
/opsx:apply            # Worktree + Superpowers TDD loop — produces the actual code
/opsx:verify           # Validate implementation matches the delta specs and tasks
```

Replace with:

```markdown
/opsx:apply            # Worktree + Superpowers TDD loop — produces the actual code and writes apply.md (the v2 receipt)
/opsx:verify           # Validate implementation matches the delta specs and tasks (requires apply.md to exist)
```

Do the same in the fast-forward flow block immediately below it (the `/opsx:ff … /opsx:apply / /opsx:verify` block uses identical wording).

- [ ] **Step 3: Verify edits**

Run:

```bash
grep -n "Schema version 2\|writes apply.md\|requires apply.md to exist" README.md
```

Expected: four matches (1 tagline, 2 apply lines, 2 verify lines = actually 5, depending on whether the fast-forward block reuses the lines; accept any count >= 3 across the listed phrases).

Run: `grep -n "Schema version 1" README.md`
Expected: zero matches.

- [ ] **Step 4: Trailing whitespace + final newline**

Run: `grep -nE ' +$' README.md; tail -c 1 README.md | xxd`
Expected: no trailing whitespace; file ends with `0a`.

- [ ] **Step 5: Commit**

```bash
git add README.md
git commit -m "docs: bump schema version to 2 in tagline and Quick Start

Root README now reflects schema v2 (apply.md receipt + verify gated
on apply). Quick Start command listings call out the new artifact."
```

---

## Task 9: Repo-wide consistency sweep

**Files:** all repo markdown + the schema YAML

- [ ] **Step 1: Confirm no stale references to schema version 1 remain anywhere**

Run: `grep -rn "Schema version 1\|version: 1" --include="*.md" --include="*.yaml" .`

Expected: zero matches in any user-facing file. (A historical mention inside a "Migration from v1" section is acceptable if it explicitly contrasts v1 vs v2; if such a match exists, confirm it is intentional context — not a leftover claim that current schema is still v1.)

- [ ] **Step 2: Confirm verify.requires is consistently described as `[apply]`**

Run: `grep -rn "verify\.\?requires.*plan\|requires: \[plan\].*verify\|verify.*requires.*plan" --include="*.md" --include="*.yaml" .`

Expected: zero matches in current-state assertions. A "v1 used to say" sentence inside the new design choice #5 is acceptable; anything else is a missed edit.

- [ ] **Step 3: Confirm apply.md is referenced as a real artifact in all four doc surfaces**

Run: `grep -rn "apply\.md" --include="*.md" --include="*.yaml" .`

Expected: matches in all of:
- `openspec/schemas/superspec/schema.yaml` (artifact + apply: block + verify instruction)
- `openspec/schemas/superspec/templates/apply.md` (the template itself)
- `openspec/schemas/superspec/templates/verify.md` (Iteration line guidance)
- `openspec/schemas/superspec/INTEGRATION.md`
- `openspec/schemas/superspec/README.md`
- `docs/workflow.md`
- `docs/workflow-details.md`
- `README.md`
- `docs/superpowers/specs/2026-05-16-apply-artifact-and-verify-requires-design.md` (the spec we wrote)
- `docs/superpowers/plans/2026-05-16-apply-artifact-and-verify-requires.md` (this plan)

If any of the four required documentation files (workflow, workflow-details, INTEGRATION, schema README) lacks an `apply.md` reference, an edit was missed — go back to the relevant task.

- [ ] **Step 4: Re-run the structural validator on the schema**

Run:

```bash
python3 -c "import yaml; d=yaml.safe_load(open('openspec/schemas/superspec/schema.yaml')); ids=[a['id'] for a in d['artifacts']]; print(ids); assert d['version']==2; assert 'apply' in ids; assert next(a['requires'] for a in d['artifacts'] if a['id']=='verify')==['apply']; assert d['apply']['requires']==['plan']; print('OK')"
```

Expected: `['brainstorm', 'proposal', 'design', 'specs', 'tasks', 'plan', 'apply', 'verify']` then `OK`.

- [ ] **Step 5: Run pre-commit on the whole tree to surface any whitespace issues**

Run: `pre-commit run --all-files`
Expected: all hooks pass. If any hook fails (typically trailing-whitespace or end-of-file-fixer), re-stage the auto-fixed files and re-run.

- [ ] **Step 6: Final commit only if pre-commit modified anything**

If the previous step produced auto-fixes:

```bash
git add -A
git commit -m "chore: pre-commit auto-fixes for v2 schema rollout"
```

Otherwise skip this commit.

- [ ] **Step 7: Confirm the branch is ready**

Run: `git log --oneline main..HEAD`
Expected: roughly nine commits — one spec commit (already present at `a99e9c9`) plus one per task above (template, verify template, schema, INTEGRATION, schema README, workflow, workflow-details, root README, optional pre-commit cleanup). The order should be readable as the file-map progression.

---

## Self-Review Notes (run after writing the plan)

**Spec coverage (each section of the spec maps to a task here):**
- "Schema changes (machine-readable diff sketch)" → Task 3 (steps 1–4).
- "New file: templates/apply.md" → Task 1.
- "Modified file: templates/verify.md" → Task 2.
- "Doc updates" table → Tasks 4–8 (one task per file listed in the spec's table).
- "Convergence loop" subsection → Task 7 step 3.
- "Testing" — "openspec validate against the new schema" → Task 3 step 6 (zod-equivalent hand-walk in python since openspec CLI isn't installed locally) + Task 9 step 4.
- "Hand-walk: scaffold openspec/changes/_test-apply-artifact/" — the spec lists this as recommended testing; intentionally not scripted here because it depends on a developer running openspec interactively. Listed as a manual follow-up below.
- "Edge cases" — encoded in the apply artifact instruction text (iteration default-to-1) and verify instruction text (iteration > 5 stop), both written verbatim from the spec in Task 3.
- "Migration from schema v1" → Task 4 step 3 (the new subsection in INTEGRATION.md).
- "Risks / Out of scope" — documented in the spec; no implementation work.

**Placeholder scan:** every code block above is concrete; no "TODO" or "TBD"; every grep command states an exact expected outcome.

**Type consistency:** field names used in instructions match the template (`Iteration`, `Applied at`, `Executor`, `Worktree`, `Branch`, `Commits`, `Tasks`). The artifact id `apply` is used identically across the schema, all docs, and grep expectations.

---

## Manual follow-ups (outside this plan's automation)

These are recommended for the implementer to run by hand before opening the PR, but are not automated steps because they need a developer-installed `openspec` CLI and an interactive shell:

1. `brew install openspec` (or equivalent), then `openspec validate --all --json` against this repo's `openspec/schemas/superspec/schema.yaml` — confirm the zod parser is happy with v2.
2. Scaffold `openspec/changes/_test-apply-artifact/` with empty stubs (brainstorm.md, proposal.md, specs/<cap>/spec.md, tasks.md, plan.md), then run `openspec status --change _test-apply-artifact --json` — confirm `apply` shows as a `ready` node and `verify` shows as `blocked` until `apply.md` is added.
3. Delete the test change directory before pushing.
