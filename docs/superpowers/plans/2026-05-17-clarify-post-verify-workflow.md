# Clarify Post-Verify Workflow (finalize artifact, v3) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Promote `finalize` to a real DAG artifact (`requires: [verify]`, `generates: finalize.md`) so the post-verify git-side closeout has an OPSX-vocabulary entry point via `/opsx:continue`; restructure the workflow into 6 phases (split Phase 5 Archival into Phase 5 Finalization and Phase 6 Archival); document the canonical PR-review golden path with `/opsx:archive`-before-merge; bump schema to v3.

**Architecture:** Add `finalize` to the `artifacts:` list, mirroring the v2 pattern that promoted `apply`. The new artifact's instruction invokes `superpowers:finishing-a-development-branch` and tells the agent to write finalize.md per `templates/finalize.md`. The recommended retrospective (currently in the apply: block) moves into finalize's instruction. apply: block step 5 (Completion) is removed entirely since finalize subsumes that responsibility. `/opsx:archive` is unchanged; the workflow docs document the canonical PR-review golden path where `/opsx:archive` runs on the feature branch BEFORE the PR merges (so archive commits land in the PR for unified review).

**Tech Stack:** YAML schema (consumed by OpenSpec's TypeScript zod parser at Fission-AI/OpenSpec), Markdown templates and docs. No code is executed; "tests" are structural validations (YAML parse, hand-walk against OpenSpec zod schema rules, grep checks for doc/schema consistency).

**Reference spec:** `docs/superpowers/specs/2026-05-17-clarify-post-verify-workflow-design.md` (commit `011f8cd`).

**Branch:** `spec/clarify-post-verify-workflow` (already created off the tip of `spec/apply-artifact-and-verify-requires`, which is in PR #3). Spec already committed and pushed at `011f8cd`.

**Baseline assumption:** PR #3's v2 state. The schema is at `version: 2`, the apply artifact exists, verify.requires is `[apply]`, and apply: block step 5 still exists. All v3 changes are on top of that.

---

## File map

| File | Action | Responsibility |
|---|---|---|
| `openspec/schemas/superspec/templates/finalize.md` | **create** | Minimal finalize-receipt template (referenced by new finalize artifact via `template:` field) |
| `openspec/schemas/superspec/templates/verify.md` | modify | Expand the Convergence loop reminder blockquote to include PASS bullets pointing at finalize |
| `openspec/schemas/superspec/schema.yaml` | modify | Bump `version: 2 → 3`; update description; add `finalize` artifact entry; rewrite verify's convergence-loop bullets to point at finalize; remove apply: block step 5; remove the retrospective section from apply: block (will reappear inside finalize's instruction) |
| `openspec/schemas/superspec/INTEGRATION.md` | modify | Update header version (v2→v3); update Section 2 7-touch-points table (touch point 7's hook becomes "finalize artifact"); redraw Section 3 DAG with finalize node; update Section 4 walkthrough — remove 3-5 Completion + 3-6 Retrospective, add new Step 4 Finalization between current Verification (now 3-3 Receipt + 3-4 Verification) and Step 5 Archive; add new design choice #6 "Finalize is a real artifact, not a hidden phase (v3)"; add "Migration from schema v2" subsection |
| `openspec/schemas/superspec/README.md` | modify | Workflow overview diagram extends through finalize; "Differences from spec-driven" table grows by one row (`verify → finalize` relationship); new decision log entry "Why finalize Is a DAG Artifact (v3)" |
| `docs/workflow.md` | modify | At-a-glance table grows from 5 phases / 9 steps to 6 phases / 10 steps; phase summary subsections gain Phase 5 (Finalization) and a renamed/repurposed Phase 6 (Archival) |
| `docs/workflow-details.md` | modify | Top description (five → six phases, nine → ten steps); at-a-glance table grows by one row; split current Phase 5 into Phase 5 (Finalization, Step 9: finalize) and Phase 6 (Archival, Step 10: archive); Phase 6 documents the canonical PR-review golden path (7 numbered steps) plus the local-merge variant callout; update the Step 7 Convergence loop subsection ASCII diagram so its PASS branch shows `→ finalize → archive` instead of `→ finishing-a-development-branch → archive` |
| `docs/project-layout.md` | modify | Add `finalize.md` to the templates tree listing and to the per-change-directory description |
| `README.md` (root) | modify | Tagline `Schema version 2 → Schema version 3`; both Quick Start blocks (step-by-step + fast-forward) add a `/opsx:continue` line between `/opsx:verify` and `/opsx:archive` mentioning finalize |
| `docs/assets/superspec-phases-flowchart.svg` | **deferred** | Phase count grows 5 → 6; SVG regeneration is non-trivial and the workflow.md table is the canonical structured source. Flag in the spec but don't fail this plan if not regenerated. |

Each commit ends a task. Commits are sequential on `spec/clarify-post-verify-workflow`.

---

## Task 1: Create `finalize.md` template

**Files:**
- Create: `openspec/schemas/superspec/templates/finalize.md`

- [ ] **Step 1: Confirm the file does not yet exist**

Run: `ls openspec/schemas/superspec/templates/`
Expected: list includes `apply.md  brainstorm.md  design.md  plan.md  proposal.md  spec.md  tasks.md  verify.md` and does NOT include `finalize.md`.

- [ ] **Step 2: Create the template**

Write exactly the following content to `openspec/schemas/superspec/templates/finalize.md`:

```markdown
# Finalize Receipt

> Generated after superpowers:finishing-a-development-branch completes,
> marking git-side closeout before /opsx:archive.
> Overwritten if you re-enter finalize (e.g., moved from "keep as-is" to PR later).

**Change**: `<change-name>`
**Finalized at**: `YYYY-MM-DD HH:mm`
**Outcome**: `merge-locally` | `pr-created` | `kept-as-is` | `discarded`

---

## Branch state

- **Branch**: `<branch-name>`
- **Base branch**: `<main / master / etc.>`
- **Final state**: `merged` | `pr-open` | `kept-open` | `deleted`
- **PR URL**: `<https://github.com/.../pull/N or N/A>`

---

## Workspace

- **Worktree**: `<path or N/A if normal repo>`
- **Cleanup**: `removed` | `preserved (PR / keep-as-is)` | `N/A (normal repo)`

---

## Tests

- **Baseline status at finish**: `passing` | `N/A (no test runner)`

---

## Next step

`<varies by outcome:>`

- `merge-locally`: "Run `/opsx:archive` on main to sync delta specs and move the change directory into the archive."
- `pr-created`: "Wait for PR review. After approval, run `/opsx:archive` on this feature branch (commits land here), push the archive commits to update the PR, then merge the PR (`gh pr merge --squash --delete-branch` or GitHub UI) — that single merge lands both the implementation and the archive into main."
- `kept-as-is`: "Resume later. Re-enter finalize when ready, or pick up where you left off; the worktree is preserved."
- `discarded`: "No further action; the change directory may also be discarded if appropriate."
```

The triple-backtick code fence above delimits the file content; the actual file starts at `# Finalize Receipt` and ends after the `discarded` bullet line, with a single trailing newline.

- [ ] **Step 3: Confirm the file ends with a single newline**

Run: `tail -c 1 openspec/schemas/superspec/templates/finalize.md | xxd`
Expected: `0000000: 0a`.

- [ ] **Step 4: Confirm no trailing whitespace**

Run: `grep -nE ' +$' openspec/schemas/superspec/templates/finalize.md`
Expected: no output (exit code 1 is fine).

- [ ] **Step 5: Commit**

```bash
git add openspec/schemas/superspec/templates/finalize.md
git commit -m "feat(schema): add finalize.md template for v3 finalize artifact

Minimal receipt structure capturing finalized-at timestamp, outcome
(merge-locally/pr-created/kept-as-is/discarded), branch state, workspace
cleanup status, test baseline, and outcome-specific Next step wording."
```

---

## Task 2: Update `verify.md` template (Convergence loop reminder)

**Files:**
- Modify: `openspec/schemas/superspec/templates/verify.md` (the existing Convergence loop reminder blockquote near the bottom of the file)

- [ ] **Step 1: Read the current blockquote**

Run: `tail -10 openspec/schemas/superspec/templates/verify.md`
Expected: see the existing blockquote starting `> **Convergence loop reminder**: If you marked FAIL with items` and ending `the user instead of continuing.`. This is the block to replace.

- [ ] **Step 2: Replace the blockquote with the expanded version**

Use Edit to replace this exact block:

```markdown
> **Convergence loop reminder**: If you marked FAIL with items
> fixable by a code change, return to the apply artifact and re-run
> `/opsx:apply`, then re-run `/opsx:verify` — this overwrites both
> apply.md and verify.md with the next iteration. If the iteration
> counter has exceeded 5, stop the loop and report the situation to
> the user instead of continuing.
```

with:

```markdown
> **Convergence loop reminder**:
> - PASS / PASS_WITH_WARNINGS → `/opsx:continue` advances to the finalize
>   artifact. Invoke superpowers:finishing-a-development-branch through
>   it, then `/opsx:archive` once finalize.md exists.
> - FAIL with items fixable by code change → return to the apply artifact
>   and re-run `/opsx:apply`, then re-run `/opsx:verify` — this overwrites
>   both apply.md and verify.md with the next iteration.
> - FAIL with artifact-level items → fix the offending artifact, then
>   re-enter apply.
> - Iteration > 5 → stop the loop and report to the user.
```

- [ ] **Step 3: Verify the edit landed cleanly**

Run: `git diff openspec/schemas/superspec/templates/verify.md | head -30`
Expected: one hunk near the bottom of the file with 5 lines removed (the old single-paragraph reminder) and the new 4-bullet reminder added. No other changes.

- [ ] **Step 4: Trailing whitespace + final newline**

Run: `grep -nE ' +$' openspec/schemas/superspec/templates/verify.md`
Expected: no output.

Run: `tail -c 1 openspec/schemas/superspec/templates/verify.md | xxd`
Expected: `0a`.

- [ ] **Step 5: Commit**

```bash
git add openspec/schemas/superspec/templates/verify.md
git commit -m "feat(schema): expand verify.md convergence reminder with PASS bullets

The reminder block previously only addressed FAIL and iteration-ceiling
cases. v3 adds PASS / PASS_WITH_WARNINGS bullets pointing at the finalize
artifact via /opsx:continue, plus an explicit FAIL-with-artifact-level
bullet for spec drift. The template now self-describes the full
post-verify decision tree."
```

---

## Task 3: Modify `schema.yaml` (version, finalize artifact, verify update, remove apply: step 5)

This is the largest task. Five sub-changes; each as a separate Edit.

**Files:**
- Modify: `openspec/schemas/superspec/schema.yaml`

### Step 1: Bump version and update top-level description

Use Edit on `openspec/schemas/superspec/schema.yaml` to replace this exact block (the current v2 version+description):

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

with:

```yaml
name: SuperSpec
version: 3
description: >
  Spec-driven workflow integrated with Superpowers skills.
  brainstorm → proposal → specs → tasks → plan → apply → verify → finalize.
  design is optional (produced from brainstorm but not required by tasks).
  Apply is both a DAG artifact (generates apply.md, a minimal receipt)
  and a top-level apply: phase block (canonical /opsx:apply instruction
  body). Verify requires apply, so /opsx:verify cannot run before
  /opsx:apply has executed. Finalize requires verify and is reached via
  /opsx:continue after verify reports PASS; its instruction invokes
  superpowers:finishing-a-development-branch and records the outcome.
  Apply uses git worktrees + subagent-driven-development (brings TDD and
  code-review transitively). executing-plans is documented only as a
  fallback for platforms without subagent support. v3: finalize is
  promoted from a manual post-verify step (documented inside the apply:
  block) to a real DAG artifact; /opsx:continue is now the OPSX-vocabulary
  entry point for git-side closeout.
```

### Step 2: Insert the new `finalize` artifact AFTER the verify artifact

Locate the end of the verify artifact block. In the v2 file, verify ends with:

```yaml
    requires:
      - apply
```

followed by a blank line, then the top-level `apply:` block beginning with `apply:`. Use Edit to insert the finalize artifact between them. Replace this exact 4-line boundary:

```yaml
    requires:
      - apply

apply:
```

with:

```yaml
    requires:
      - apply

  - id: finalize
    generates: finalize.md
    description: Git-side closeout (PR / merge / worktree cleanup) before archive
    template: finalize.md
    instruction: |
      Use the Skill tool to invoke superpowers:finishing-a-development-branch.

      That skill presents 4 options (or 3 for detached HEAD): merge
      locally, push and create a PR, keep as-is, or discard. Pick the
      appropriate one and let the skill execute it (it verifies tests,
      manages worktree cleanup, and creates the PR or merge as chosen).

      Then write finalize.md per templates/finalize.md: outcome chosen,
      PR URL if applicable, final branch state, worktree cleanup status,
      test-baseline confirmation, timestamp.

      Recommended (non-blocking): before writing finalize.md, write a
      short retrospective.md in the change directory. Six suggested
      sections: Wins, Misses, Plan deviations, Skill/workflow compliance,
      Surprises, Promote candidates. Evidence first, opinion second.
      Skippable for trivial single-commit fixes.

      Once finalize.md exists, the change is git-clean and ready for
      /opsx:archive. See docs/workflow-details.md Phase 6 (Archival)
      for the canonical PR-review golden path, including the
      archive-before-merge ordering.
    requires:
      - verify

apply:
```

### Step 3: Update the verify artifact's Convergence loop bullets

Locate the verify artifact's `Recommended convergence loop:` bullet list. In the v2 file the bullets read:

```yaml
      Recommended convergence loop:
      - PASS → proceed to finishing-a-development-branch + archive.
      - PASS_WITH_WARNINGS → proceed; record warnings.
      - FAIL with code-fixable items → return to apply, re-run, re-verify.
      - FAIL with artifact-level items (e.g. spec drift) → fix the
        offending artifact, then re-enter apply.
      - Iteration > 5 → stop looping; report to the user.
```

Use Edit to replace that exact block with:

```yaml
      Recommended convergence loop:
      - PASS → /opsx:continue advances to the finalize artifact; invoke
        superpowers:finishing-a-development-branch through it.
        /opsx:archive runs once finalize.md exists.
      - PASS_WITH_WARNINGS → same path as PASS; warnings are recorded
        but do not block.
      - FAIL with code-fixable items → return to apply, re-run, re-verify.
      - FAIL with artifact-level items (e.g. spec drift) → fix the
        offending artifact, then re-enter apply.
      - Iteration > 5 → stop looping; report to the user.
```

### Step 4: Remove apply: block step 5 (Completion)

In the v2 file the apply: block's step list ends with this exact block (step 5 plus the blank line that follows it):

```yaml
    5. **Completion**: Once verify.md shows no blocking issues,
       use the Skill tool to invoke
       **superpowers:finishing-a-development-branch** for branch
       cleanup, PR creation, etc.

    **Recommended — Retrospective before archive (non-blocking)**:
```

Use Edit to replace that block with just:

```yaml
    **Recommended — Retrospective before archive (non-blocking)**:
```

(The blank line + step 5 paragraph is removed; the "**Recommended …**" header that follows is preserved verbatim so the next Edit can target it.)

### Step 5: Remove the retrospective subsection from apply: block (it moves into finalize's instruction)

The apply: block in v2 ends with a "**Recommended — Retrospective before archive (non-blocking)**:" header followed by ~30 lines of guidance (Why, six suggested sections, fallback note, etc.). In v3 this entire trailing subsection is removed because the equivalent guidance now lives in finalize's artifact instruction (added in Task 3 step 2 above).

Read `openspec/schemas/superspec/schema.yaml` and locate the trailing block starting with `    **Recommended — Retrospective before archive (non-blocking)**:` (after step 4 of the apply: block) through the end of the file.

Use Edit to replace this exact trailing block (verbatim — read the file to get the precise current text; the block starts with `    **Recommended — Retrospective before archive (non-blocking)**:` and ends with `using the standard task-by-task loop from tasks.md.` which is the last line of the file):

```yaml
    **Recommended — Retrospective before archive (non-blocking)**:

    Before archiving the change, it is strongly recommended (but not
    required) to write a short retrospective at `retrospective.md` in
    the change directory. A good retrospective raises the quality of
    every subsequent change because it captures what the diff alone
    cannot: why decisions were made, what surprised you, and which
    learnings deserve to be promoted to long-term memory.

    Evidence first, opinion second — every claim should cite a
    commit, file, or measurable fact. Suggested sections:

    - **Wins** — what worked well (with commit / test evidence)
    - **Misses** — what didn't work (🔴 blocking / 🟡 painful / 📌 nit)
    - **Plan deviations** — tasks whose scope changed, and why
    - **Skill / workflow compliance** — skills invoked vs. deliberately
      skipped (and the reason)
    - **Surprises** — assumptions that turned out wrong
    - **Promote candidates** — learnings to move into long-term memory,
      CLAUDE.md, or schema/skill updates (classify each candidate so
      insights don't die silently in the archive)

    If a `workflow-retrospective` skill is available in the environment,
    it can automate evidence collection; otherwise write the six
    sections manually. Skipping is acceptable for trivial changes
    (single-commit fixes) where the overhead exceeds the value.

    If any skill is unavailable, fall back to manual implementation
    using the standard task-by-task loop from tasks.md.
```

with just:

```yaml
    If any skill is unavailable, fall back to manual implementation
    using the standard task-by-task loop from tasks.md.
```

This preserves the original fallback line as the apply: block's closing sentence (it is still relevant to apply itself). The retrospective recommendation is now exclusively in finalize's instruction.

**Note:** before invoking Edit, run `Read openspec/schemas/superspec/schema.yaml` with an offset around line 367 to verify the exact whitespace and content of the block to remove. Indentation matters.

### Step 6: Validate the YAML parses + structural checks

Run:

```bash
python3 -c "import yaml; d=yaml.safe_load(open('openspec/schemas/superspec/schema.yaml')); print('version:', d['version']); print('artifact ids:', [a['id'] for a in d['artifacts']]); print('verify.requires:', next(a['requires'] for a in d['artifacts'] if a['id']=='verify')); print('finalize.requires:', next(a['requires'] for a in d['artifacts'] if a['id']=='finalize')); print('finalize.generates:', next(a['generates'] for a in d['artifacts'] if a['id']=='finalize')); print('finalize.template:', next(a['template'] for a in d['artifacts'] if a['id']=='finalize')); print('apply: block requires:', d['apply']['requires']); print('apply: block tracks:', d['apply']['tracks'])"
```

Expected output:

```
version: 3
artifact ids: ['brainstorm', 'proposal', 'design', 'specs', 'tasks', 'plan', 'apply', 'verify', 'finalize']
verify.requires: ['apply']
finalize.requires: ['verify']
finalize.generates: finalize.md
finalize.template: finalize.md
apply: block requires: ['plan']
apply: block tracks: tasks.md
```

If anything differs, identify the responsible Edit and fix it before continuing.

### Step 7: Hand-walk the OpenSpec zod rules

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
    for dep in adj.get(start, []):
        if dep not in seen:
            if has_cycle(adj, dep, seen, stack): return True
        elif dep in stack:
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

Expected: `OK: zod-equivalent structural checks pass`.

### Step 8: Confirm trailing whitespace + final newline

Run: `grep -nE ' +$' openspec/schemas/superspec/schema.yaml` — expect no output (exit code 1 is fine).
Run: `tail -c 1 openspec/schemas/superspec/schema.yaml | xxd` — expect `0a`.

### Step 9: Commit

```bash
git add openspec/schemas/superspec/schema.yaml
git commit -m "feat(schema): promote finalize to v3 DAG artifact

- Bump version 2 -> 3 with updated description noting the v3
  promotion
- Add finalize artifact (requires verify, generates finalize.md);
  instruction invokes superpowers:finishing-a-development-branch
  and includes the recommended retrospective guidance
- Update verify artifact's convergence loop bullets so PASS /
  PASS_WITH_WARNINGS route to finalize via /opsx:continue
- Remove apply: block step 5 (Completion) and the trailing
  'Recommended Retrospective before archive' subsection — both
  responsibilities now live in finalize's instruction

OpenSpec CLI behavior preserved: /opsx:apply still surfaces the
apply: block instruction; /opsx:continue now surfaces finalize after
verify completes; /opsx:archive is unchanged."
```

---

## Task 4: Update INTEGRATION.md

**Files:**
- Modify: `openspec/schemas/superspec/INTEGRATION.md`

Five edits in different regions of the file. Read the file first to confirm exact whitespace.

### Step 1: Bump header version

Use Edit to replace:

```markdown
> Corresponding schema version: `sdd-plus-superpowers` v2
```

with:

```markdown
> Corresponding schema version: `sdd-plus-superpowers` v3
```

### Step 2: Update Section 2 (7-touch-points table) — touch point 7 hook

Use Edit to replace:

```markdown
| 7 | `superpowers:finishing-a-development-branch` | apply step 5 | Direct |
```

with:

```markdown
| 7 | `superpowers:finishing-a-development-branch` | `finalize` artifact instruction | Direct |
```

### Step 3: Redraw Section 3 DAG diagram with finalize as a real node

Locate the current diagram (which already shows apply as a real node from PR #3). The bottom of the diagram looks like:

```text
       ▼         ▼
    ┌──────────┐ ┌──────────┐
    │  design  │ │  verify  │ ◄── openspec-verify-change (5 checks)
    │(optional)│ └──────────┘
    └──────────┘
```

Use Edit to replace this exact 5-line block with:

```text
       ▼         ▼
    ┌──────────┐ ┌──────────┐
    │  design  │ │  verify  │ ◄── openspec-verify-change (5 checks)
    │(optional)│ └────┬─────┘
    └──────────┘      │
                      ▼
                  ┌──────────┐
                  │ finalize │ ◄── superpowers:finishing-a-development-branch
                  │          │     writes finalize.md
                  └──────────┘
                      │
                      ▼
                  /opsx:archive  (not in DAG; OpenSpec CLI)
```

### Step 4: Update Key points bullets after the DAG diagram

In the v2 file the Key points section (which immediately follows the diagram block) has 4 bullets. Use Edit to replace the entire Key points block (find the exact start with `**Key points**:` and the end at the last v2 bullet about "convergence loop") with a 5-bullet version that adds finalize:

```markdown
**Key points**:

- `design` is an **optional leaf**. Brainstorm still attempts to pre-populate design.md, but tasks no longer hard-depend on it (`tasks.requires: [specs]`). Per OpenSpec conventions: `design.md` is only written when non-trivial technical decisions need explanation.
- `apply` is a **real DAG node** as of schema v2. It generates `apply.md` (a minimal receipt — iteration counter, worktree, branch, commit range, task counts) so the DAG can honestly express "verify depends on apply having run." The canonical `/opsx:apply` instruction body still lives in the top-level `apply:` phase block; the apply artifact's own instruction is a short redirect to avoid drift.
- `verify` requires `apply` (was `plan` in v1). The OpenSpec CLI will refuse to surface verify as a `ready` artifact until `apply.md` exists.
- `finalize` is a **real DAG node** as of schema v3. It generates `finalize.md` (a minimal git-closeout receipt: outcome, PR URL, final branch state) and requires `verify`. `/opsx:continue` surfaces finalize's instruction after verify completes; that instruction invokes `superpowers:finishing-a-development-branch`. `/opsx:archive` is the lifecycle close that follows finalize and is not in the DAG (it remains an OpenSpec CLI command).
- The convergence loop (apply → verify → loop back on code-fixable FAILs, capped at 5 iterations) is documented in `docs/workflow-details.md`. The schema enforces the file-existence dependency; the iteration decision is made by the agent or by a future loop-runner command (not in scope for v2 or v3).
```

When constructing the `old_string` for this Edit, copy the exact 4-bullet block from the current file as-is. If your Edit tool requires exact whitespace, read the file with an offset around the Key points heading and copy from there.

### Step 5: Update Section 4 walkthrough — restructure steps 3-5 / 3-6

The v2 file has Section 4 Step 3 (Apply) split into sub-steps 3-0 through 3-6, where 3-5 is "Completion — Invoke `superpowers:finishing-a-development-branch`" and 3-6 is "Retrospective (Recommended, Non-blocking)". In v3:

- Sub-step 3-5 (Completion) is REMOVED from Step 3 (Apply).
- Sub-step 3-6 (Retrospective) is REMOVED from Step 3 (Apply).
- A new top-level Step 4 (Finalization) is inserted with two sub-steps: 4-1 Finalize (invoke finishing-a-development-branch + write finalize.md), 4-2 Retrospective (recommended).
- The previous Step 4 (Archive) is renumbered to Step 5 (Archive).
- The new Step 5 (Archive) prose documents the canonical PR-review golden path.

The exact edits:

**Edit 5a — Remove sub-steps 3-5 and 3-6 from Step 3 (Apply)**

Use Edit to replace this exact block (read the file around line ~210 to copy the exact text):

```markdown
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
```

with:

```markdown
---

### Step 4: Finalization

`/opsx:continue` after verify completes surfaces the `finalize` artifact's instruction.

#### 4-1. Finalize — Invoke `superpowers:finishing-a-development-branch`

- Confirm all tests are green
- Present options: merge / PR / keep branch / discard
- Clean up the worktree (preserved if Option 2 PR or Option 3 keep-as-is)
- Write `finalize.md` per `openspec/schemas/superspec/templates/finalize.md`: outcome chosen, PR URL if applicable, final branch state, worktree cleanup status, test-baseline confirmation, timestamp.

#### 4-2. Retrospective (Recommended, Non-blocking)

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

### Step 5: Archive
```

**Edit 5b — Renumber the Archive heading at the bottom of Section 4**

In v2 the file's Section 4 closes with `### Step 4: Archive`. After Edit 5a above, the heading is now at `### Step 5: Archive` (the Finalization step took position 4). No further heading edit is needed if Edit 5a's `new_string` already includes the `### Step 5: Archive` heading line as shown.

After Edit 5a the prose of the archive step should be augmented to document the canonical PR-review golden path. Use a separate Edit to find and replace the v2 archive prose. The v2 prose (immediately under `### Step 5: Archive` after Edit 5a, formerly under `### Step 4: Archive`) reads:

```markdown
### Step 5: Archive

```bash
/opsx:archive my-feature
```

- Validates + checks task completion (incomplete tasks warn but don't block)
- Syncs delta specs back to `openspec/specs/<capability>/spec.md`
  - Order: RENAMED → REMOVED → MODIFIED → ADDED
  - If already manually synced, use `--skip-specs`
- Moves `changes/my-feature/` to `changes/archive/YYYY-MM-DD-my-feature/`
- History is frozen; the unix timeline is treated as the source of truth
```

Replace with:

````markdown
### Step 5: Archive

```bash
/opsx:archive my-feature
```

Behavior:

- Validates + checks task completion (incomplete tasks warn but don't block)
- Syncs delta specs back to `openspec/specs/<capability>/spec.md`
  - Order: RENAMED → REMOVED → MODIFIED → ADDED
  - If already manually synced, use `--skip-specs`
- Moves `changes/my-feature/` to `changes/archive/YYYY-MM-DD-my-feature/`
- History is frozen; the unix timeline is treated as the source of truth

`/opsx:archive` does NOT merge git branches and does NOT create PRs. The git-side closeout is the `finalize` artifact's responsibility (Step 4-1 above).

#### Canonical PR-review golden path

```text
1. verify completes (verify.md committed on feature branch)
2. finalize (creates PR via finishing-a-development-branch; finalize.md says pr-open; worktree preserved)
3. [PAUSE: human review on the PR; reviewer approves]
4. /opsx:archive on the feature branch (syncs delta specs, moves change dir; new commits land on the branch)
5. Push the archive commits to update the PR
6. PR merge (gh pr merge --squash --delete-branch or GitHub UI)
7. Local worktree cleanup if still present (git worktree remove + git worktree prune)
```

The archive-before-merge ordering keeps the PR's diff complete: every commit that went into the change is in the PR, including the archive sync. If the PR is merged before archive runs, the archive commits would have to be authored on main after the fact — recoverable, but loses the unified PR audit trail.

#### Local-merge variant (acceptable for solo / local-only changes)

If finalize chose Option 1 (Merge locally), the skill performs the merge inline and removes the worktree. `/opsx:archive` then runs on main directly. This inverts the archive/merge order vs. the canonical path. Acceptable for solo or local-only changes where the PR audit trail isn't relevant.
````

### Step 6: Add a new design choice #6 in Section 6

Section 6 in the v2 file is "## 6. Elegant Design Choices in the Integration (5 Worth Remembering)". The v2 changes added a design choice #5 about apply. v3 adds design choice #6.

First, update the section heading. Use Edit to replace:

```markdown
## 6. Elegant Design Choices in the Integration (5 Worth Remembering)
```

with:

```markdown
## 6. Elegant Design Choices in the Integration (6 Worth Remembering)
```

Then add the new design choice. Find the end of design choice #5 (right before the `### Migration from schema v1` heading or right before the closing `---` divider). Use Edit to replace the divider line that separates design choice #5 from the migration section:

```markdown
This change also unlocks the documented apply → verify → repeat convergence loop, since `verify.md` outcomes can now feed cleanly back into a re-run of apply with an incremented iteration counter. See `docs/workflow-details.md` for the loop pattern.

### Migration from schema v1
```

with:

```markdown
This change also unlocks the documented apply → verify → repeat convergence loop, since `verify.md` outcomes can now feed cleanly back into a re-run of apply with an incremented iteration counter. See `docs/workflow-details.md` for the loop pattern.

### 6. Finalize is a real artifact, not a hidden phase (v3)

In v2, post-verify git closeout (PR creation, worktree cleanup) was reachable only via the apply: block's step 5 prose, which an agent that just ran `/opsx:verify` typically doesn't re-read. The verify artifact's instruction said "proceed" on PASS without naming the next call site, and verify.md's convergence-loop reminder only covered FAIL paths. As a result, agents (and humans) routinely jumped straight to `/opsx:archive`, skipping `superpowers:finishing-a-development-branch` and leaving the branch and PR unfinished.

In v3, `finalize` is promoted to a real DAG artifact (`generates: finalize.md`, `requires: [verify]`). `/opsx:continue` surfaces its instruction after verify completes; that instruction invokes `superpowers:finishing-a-development-branch` and records the outcome. No new slash command is needed — the existing OpenSpec workflow vocabulary already gives the call site.

This change also moves the recommended retrospective guidance from the apply: block (where it was misplaced — apply ends with verify, not with archive) into finalize's instruction, where it logically belongs as a pre-archive activity.

### Migration from schema v1
```

### Step 7: Add a new "Migration from schema v2" subsection before Section 7

In v2 the file already has a `### Migration from schema v1` subsection added by PR #3. v3 adds a parallel `### Migration from schema v2` subsection immediately after it.

Use Edit to replace the boundary between the v1 migration paragraph and the start of Section 7. The v2 file's current text is:

```markdown
Migration: author a minimal `apply.md` by hand from `openspec/schemas/superspec/templates/apply.md` — set `Iteration: 1`, fill the worktree path, branch, commit range, and task counts from the existing implementation, then re-run `/opsx:verify`. This is a one-time migration cost per in-flight change; no archived changes are affected.

---

## 7. Recommended Snapshot Section for Projects Adopting This Schema
```

Replace with:

```markdown
Migration: author a minimal `apply.md` by hand from `openspec/schemas/superspec/templates/apply.md` — set `Iteration: 1`, fill the worktree path, branch, commit range, and task counts from the existing implementation, then re-run `/opsx:verify`. This is a one-time migration cost per in-flight change; no archived changes are affected.

### Migration from schema v2

If a project pinned to schema v2 has in-flight changes that already have `verify.md` but no `finalize.md`, `/opsx:continue` under v3 will report finalize as the next ready artifact and refuse to advance further. Migration:

1. Author a minimal `finalize.md` by hand from `openspec/schemas/superspec/templates/finalize.md`. Fill the fields from the actual branch state — pick the outcome that matches what was already done (e.g., `pr-created` if a PR exists), copy branch state from `git status` / `gh pr view`, and record current worktree state.
2. Re-run `/opsx:continue`; it should now advance past finalize, and the change is ready for `/opsx:archive`.

This is a one-time migration cost per in-flight v2 change; no archived changes are affected.

---

## 7. Recommended Snapshot Section for Projects Adopting This Schema
```

### Step 8: Verify the edits

Run:

```bash
grep -nE "version: \`sdd-plus-superpowers\` v3|finalize.*artifact instruction|finalize.*DAG node|Finalize is a real artifact|Migration from schema v2" /Users/homer/dev/superspec/openspec/schemas/superspec/INTEGRATION.md
```

Expected: at least 5 matches (header version, touch-point 7 hook, key-points bullet, design choice #6 heading, migration heading).

Run:

```bash
grep -n "apply step 5\|step 5 (Completion)" /Users/homer/dev/superspec/openspec/schemas/superspec/INTEGRATION.md
```

Expected: zero matches (the old stale references are gone).

### Step 9: Trailing whitespace + final newline

Run: `grep -nE ' +$' openspec/schemas/superspec/INTEGRATION.md` — no output.
Run: `tail -c 1 openspec/schemas/superspec/INTEGRATION.md | xxd` — `0a`.

### Step 10: Commit

```bash
git add openspec/schemas/superspec/INTEGRATION.md
git commit -m "docs(schema): update INTEGRATION for v3 finalize artifact

- Header version v2 -> v3
- 7-touch-points table: finishing-a-development-branch hook becomes
  the finalize artifact instruction
- Section 3 DAG diagram: add finalize node between verify and the
  /opsx:archive sink
- Key points: new bullet for finalize as a real DAG node in v3
- Section 4 walkthrough: remove 3-5 Completion and 3-6 Retrospective
  from Step 3 (Apply); insert new Step 4 (Finalization) with 4-1
  Finalize and 4-2 Retrospective; renumber Archive to Step 5 and
  augment with the canonical PR-review golden path plus local-merge
  variant
- Section 6: new design choice #6 'Finalize is a real artifact, not
  a hidden phase (v3)'
- New 'Migration from schema v2' subsection in Section 6"
```

---

## Task 5: Update schema's `README.md`

**Files:**
- Modify: `openspec/schemas/superspec/README.md`

### Step 1: Extend the workflow overview diagram

Use Edit to replace:

```text
brainstorm ──→ proposal ──→ specs ──→ tasks ──→ plan ──→ apply ──→ verify
                  │                     ↑
                  └──→ design ──────────┘
```

with:

```text
brainstorm ──→ proposal ──→ specs ──→ tasks ──→ plan ──→ apply ──→ verify ──→ finalize
                  │                     ↑
                  └──→ design ──────────┘
```

### Step 2: Update the "Differences from spec-driven" table

Use Edit to replace the entire v2 table:

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

with:

```markdown
| | spec-driven | sdd-plus-superpowers (v3) |
|---|---|---|
| Starting point | proposal (written manually) | **brainstorm** (invokes brainstorming skill) |
| Endpoint | tasks (coarse-grained) | **finalize** (git-side closeout receipt; archive follows as a CLI step) |
| apply requires | tasks | **plan** |
| apply method | Standard task-by-task | **worktree + subagent-driven-development** |
| Additional artifacts | — | brainstorm, plan, apply (receipt), **finalize (receipt)** |
| verify requires | tasks | **apply** (v2 — was plan in v1) |
| finalize requires | — | **verify** (v3) |
```

### Step 3: Add a new decision log entry above "Why apply Is Both an Artifact and a Phase Block"

The v2 README has a Design Decision Log section with entries "Why brainstorm Is an Artifact, Not a Hook", "Why plan Is Separate from tasks", "Why apply Is Both an Artifact and a Phase Block (v2)", and "Fallback Strategy".

Use Edit to add a new "Why finalize Is a DAG Artifact (v3)" entry between the apply entry and the fallback strategy. Find this exact line in the file:

```markdown
### Fallback Strategy
```

Replace with:

```markdown
### Why finalize Is a DAG Artifact (v3)

In v2 the post-verify git closeout (PR creation / merge / worktree cleanup) lived only in the apply: block's step 5 — a place an agent that just ran `/opsx:verify` typically doesn't re-read. The verify artifact's PASS bullets said "proceed" without naming the next call site, so agents routinely jumped straight to `/opsx:archive`, skipping `superpowers:finishing-a-development-branch` entirely.

v3 promotes `finalize` to a real DAG artifact (`generates: finalize.md`, `requires: [verify]`). `/opsx:continue` surfaces its instruction after verify completes, which invokes `superpowers:finishing-a-development-branch` and records the outcome. The recommended retrospective guidance moves from the apply: block (where it was misplaced — apply ends with verify) into finalize's instruction, where it belongs as a pre-archive activity. `/opsx:archive` is unchanged; it remains an OpenSpec CLI command that runs after finalize and is documented in `docs/workflow-details.md` Phase 6 with the canonical archive-before-merge golden path.

### Fallback Strategy
```

### Step 4: Verify edits

Run:

```bash
grep -n "──→ finalize\|finalize (receipt)\|finalize requires\|Why finalize Is a DAG Artifact" /Users/homer/dev/superspec/openspec/schemas/superspec/README.md
```

Expected: at least 4 matches.

### Step 5: Trailing whitespace + final newline

Run: `grep -nE ' +$' openspec/schemas/superspec/README.md` — no output.
Run: `tail -c 1 openspec/schemas/superspec/README.md | xxd` — `0a`.

### Step 6: Commit

```bash
git add openspec/schemas/superspec/README.md
git commit -m "docs(schema): update schema README workflow + table for v3

Workflow overview extends through finalize. Differences table grows
by one row (finalize requires verify). New decision log entry
explains why finalize is a DAG artifact rather than apply: block
prose, including the v2 problem it solves."
```

---

## Task 6: Update `docs/workflow.md`

**Files:**
- Modify: `docs/workflow.md`

### Step 1: Update the introductory paragraph

The current intro says "A Superspec change moves through five phases." Use Edit to replace:

```markdown
A Superspec change moves through five phases. OpenSpec governs the artifacts and lifecycle; Superpowers supplies the execution discipline for brainstorming, planning, TDD, review, and branch cleanup.

Use this page as the quick mental model. For the full nine-step rationale, see [workflow details](workflow-details.md).
```

with:

```markdown
A Superspec change moves through six phases. OpenSpec governs the artifacts and lifecycle; Superpowers supplies the execution discipline for brainstorming, planning, TDD, review, and branch cleanup.

Use this page as the quick mental model. For the full ten-step rationale, see [workflow details](workflow-details.md).
```

### Step 2: Update the At a Glance table

Use Edit to replace the entire v2 table (which has 9 rows including the Phase 5 Archival row):

```markdown
| Phase | # | Step | Brief why | Owner |
|---|---:|---|---|---|
| **1. Brainstorm** | 1 | `brainstorm` | Avoid building the wrong thing. | Superpowers |
| **2. Artifact Creation** | 2 | `proposal` | Define the change boundary. | OpenSpec |
| | 3 | `design` *(optional)* | Explain the chosen solution. | Hybrid |
| | 4 | `specs` | Create the testable contract. | OpenSpec |
| | 5 | `tasks` | Scope the work. | OpenSpec |
| | 6 | `plan` | Make the work executable. | Superpowers |
| **3. Code Implementation** | 7 | `apply` | Change the system. | Superpowers |
| **4. Spec Validation** | 8 | `verify` | Prove it matches intent. | OpenSpec |
| **5. Archival** | 9 | finish / archive boundary | Close the lifecycle cleanly. | Hybrid |
```

with:

```markdown
| Phase | # | Step | Brief why | Owner |
|---|---:|---|---|---|
| **1. Brainstorm** | 1 | `brainstorm` | Avoid building the wrong thing. | Superpowers |
| **2. Artifact Creation** | 2 | `proposal` | Define the change boundary. | OpenSpec |
| | 3 | `design` *(optional)* | Explain the chosen solution. | Hybrid |
| | 4 | `specs` | Create the testable contract. | OpenSpec |
| | 5 | `tasks` | Scope the work. | OpenSpec |
| | 6 | `plan` | Make the work executable. | Superpowers |
| **3. Code Implementation** | 7 | `apply` | Change the system. | Superpowers |
| **4. Spec Validation** | 8 | `verify` | Prove it matches intent. | OpenSpec |
| **5. Finalization** | 9 | `finalize` | Close out the git side cleanly before archive. | Superpowers |
| **6. Archival** | 10 | `archive` boundary | Sync deltas and freeze the change. | OpenSpec |
```

### Step 3: Update Phase 5 summary subsection and add Phase 6

In the v2 file the Phase 5 subsection reads:

```markdown
### 5. Archival

Superpowers handles branch / PR / worktree closure. OpenSpec then applies the delta specs into the living `openspec/specs/` tree and archives the completed change.

Primary outputs: updated living specs and an archived change directory.
```

Use Edit to replace it with:

```markdown
### 5. Finalization

The `finalize` artifact (introduced in v3) is reached via `/opsx:continue` after verify reports PASS. Its instruction invokes `superpowers:finishing-a-development-branch` — which presents merge / PR / keep / discard options — and writes `finalize.md` recording the outcome. The recommended retrospective lives here as well, before archive.

Primary outputs: `finalize.md` (the closeout receipt); optional `retrospective.md`.

### 6. Archival

`/opsx:archive` syncs the change's delta specs into the living `openspec/specs/` tree and moves the change directory into the archive. It does not merge git branches; in the canonical PR-review golden path, `/opsx:archive` runs on the feature branch BEFORE the PR merges, so the archive commits land in the PR for a unified review.

Primary outputs: updated living specs and an archived change directory.
```

### Step 4: Update the alt text in the SVG embed line (no SVG regen)

The file embeds the phase flowchart with `![Superspec 5-phase pipeline](assets/superspec-phases-flowchart.svg)`. Use Edit to replace the alt text:

```markdown
![Superspec 5-phase pipeline](assets/superspec-phases-flowchart.svg)
```

with:

```markdown
![Superspec 6-phase pipeline](assets/superspec-phases-flowchart.svg)
```

(The SVG itself is not regenerated in this plan — see Task 9 / scope. The text reference is updated for accuracy.)

### Step 5: Verify edits

Run:

```bash
grep -nE "six phases|6-phase pipeline|ten-step|finalize.*Close out|6\. Archival" /Users/homer/dev/superspec/docs/workflow.md
```

Expected: at least 5 matches.

Run: `grep -n "five phases\|5-phase pipeline\|nine-step\|finish / archive boundary" /Users/homer/dev/superspec/docs/workflow.md`
Expected: zero matches.

### Step 6: Trailing whitespace + final newline

Run: `grep -nE ' +$' docs/workflow.md` — no output.
Run: `tail -c 1 docs/workflow.md | xxd` — `0a`.

### Step 7: Commit

```bash
git add docs/workflow.md
git commit -m "docs: workflow.md splits Phase 5 into Finalization + Archival

Intro updates from 5 phases / 9 steps to 6 phases / 10 steps. At a
Glance table grows by one row (Step 9 finalize, Step 10 archive).
Phase 5 summary becomes Finalization; new Phase 6 summary covers
archival including the archive-before-merge golden path."
```

---

## Task 7: Update `docs/workflow-details.md`

**Files:**
- Modify: `docs/workflow-details.md`

Largest doc edit. Four regions.

### Step 1: Update the introductory paragraph + at-a-glance table

Use Edit to replace the v2 intro + table block:

```markdown
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
```

with:

```markdown
A Superspec change moves through **six phases**, broken into **ten concrete steps**. This page walks through each step — what it is, **why it's required**, what concretely happens, which source system (OpenSpec or Superpowers) owns it, and what step from the other source is replaced.

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
| **5. Finalization** | 9 | `finalize` | Close out the git side cleanly before archive. | Superpowers: `finishing-a-development-branch` (via `/opsx:continue`) |
| **6. Archival** | 10 | `archive` boundary | Sync deltas and freeze the change. | OpenSpec: `/opsx:archive` |
```

### Step 2: Update the convergence-loop diagram inside Step 7

Step 7 contains a "Convergence loop (apply → verify → repeat)" subsection added in PR #3. Its ASCII diagram currently routes PASS to `finishing-a-development-branch → archive`. In v3 it should route to finalize. Use Edit to replace the diagram block. The current text reads:

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

Replace with:

```text
        plan
         │
         ▼
       apply  ────► apply.md   (iteration N)
         │
         ▼
       verify ────► verify.md  (iteration N)
         │
         ├── PASS or PASS_WITH_WARNINGS ──► /opsx:continue → finalize → /opsx:archive
         │
         ├── FAIL, items fixable by code change ──► return to apply (N+1)
         │
         ├── FAIL, items in artifacts (spec drift) ──► fix artifact → apply (N+1)
         │
         └── Iteration > 5 ──► stop; report to the user
```

### Step 3: Update the Step 7 "Termination rules" PASS bullet

Just below the diagram, the Termination rules list has a PASS bullet that should reference finalize. Use Edit to replace:

```markdown
- **PASS** — proceed to `finishing-a-development-branch` and archive.
```

with:

```markdown
- **PASS** — `/opsx:continue` advances to the finalize artifact (Step 9). After finalize.md is written, run `/opsx:archive` (Step 10).
```

### Step 4: Restructure Phase 5 — split into Phase 5 (Finalization) and Phase 6 (Archival)

The v2 file's Phase 5 (Archival) is a single block from `## Phase 5: Archival` through the start of `## Superpowers skill index & fallbacks`. Use Edit to replace the entire Phase 5 block. The current text (anchor on `## Phase 5: Archival` through `Both are needed to leave the repo in a clean state.` plus the trailing `---`):

```markdown
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
```

Replace with:

````markdown
## Phase 5: Finalization

The Finalization phase performs the git-side closeout for the change — merging the branch (if local-only), creating a PR (if going through code review), cleaning up the worktree, and recording the outcome. It contains a single step.

### Step 9. Finalize — `finalize`

> Closes out the development branch in git terms; writes the finalize.md receipt.

**Brief why:** Close out the git side cleanly before archive.

**Why it's required.** In v2 the post-verify git closeout was reachable only via prose in the apply: block — out of the line of sight for an agent that just ran `/opsx:verify`. In v3 finalize is a real DAG artifact: `/opsx:continue` surfaces it as the next ready step after verify, and its instruction names `superpowers:finishing-a-development-branch` as the skill to invoke. The DAG makes the post-verify handoff visible to both agents and humans.

**What the step does.** `/opsx:continue` advances to the finalize artifact, whose instruction invokes `superpowers:finishing-a-development-branch`. That skill:

- Verifies the project's test baseline passes (precondition).
- Presents 4 options (or 3 for detached HEAD): merge locally / push & create PR / keep as-is / discard.
- Executes the chosen option (creates the PR, performs the merge, etc.).
- Manages worktree cleanup based on the option (removed for merge/discard, preserved for PR/keep).

Then the agent writes `finalize.md` per `openspec/schemas/superspec/templates/finalize.md` — a minimal receipt: outcome, PR URL if any, final branch state, worktree cleanup status, test-baseline confirmation, timestamp.

**Recommended (non-blocking).** Before writing finalize.md, write a short `retrospective.md` in the change directory. Six suggested sections: Wins, Misses, Plan deviations, Skill/workflow compliance, Surprises, Promote candidates. The retrospective recommendation was moved into the finalize artifact in v3 because it belongs as a pre-archive activity, not as part of the apply phase.

**Source phase used.** `superpowers:finishing-a-development-branch` (invoked via `/opsx:continue` on the finalize artifact).

**Step not used / replaced and why.** Vanilla OpenSpec has no equivalent step. v2 documented this responsibility only inside the apply: block step 5, which agents that had just run `/opsx:verify` typically did not re-read. v3 promotes it to a DAG artifact so the OPSX-vocabulary entry point (`/opsx:continue`) surfaces it automatically.

---

## Phase 6: Archival

The Archival phase syncs the change's delta specs into the living spec tree and moves the change directory into the archive. It contains a single step.

### Step 10. Archive — `/opsx:archive`

> Syncs delta specs and freezes the change directory.

**Brief why:** Sync deltas and freeze the change.

**Why it's required.** Once the git side is closed out (Phase 5), the OpenSpec change still needs to be merged into the project's *living specs* and archived for history. `/opsx:archive` does both. It is intentionally git-agnostic — it does NOT merge branches or create PRs; that's Phase 5's job.

**What the step does.** Runs `/opsx:archive my-feature` (or `openspec archive`). Behavior:

- Validates the change (`openspec validate`) and checks task completion (unchecked items warn but don't block).
- Syncs delta specs into `openspec/specs/<capability>/spec.md`. Apply order: RENAMED → REMOVED → MODIFIED → ADDED. If already manually synced, use `--skip-specs`.
- Moves `openspec/changes/<name>/` into `openspec/changes/archive/YYYY-MM-DD-<name>/`. Both moves are committed on the current branch — typically the feature branch in the canonical golden path below.

#### Canonical PR-review golden path

```text
1. verify completes (verify.md committed on feature branch)
2. finalize (creates PR via finishing-a-development-branch; finalize.md says pr-open; worktree preserved)
3. [PAUSE: human review on the PR; reviewer approves]
4. /opsx:archive on the feature branch (syncs delta specs, moves change dir; new commits land on the branch)
5. Push the archive commits to update the PR (git push)
6. PR merge (gh pr merge --squash --delete-branch or GitHub UI)
7. Local worktree cleanup if still present (cd to main repo root; git worktree remove .worktrees/<name>; git worktree prune)
```

The archive-before-merge ordering keeps the PR's diff complete: every commit that went into the change is in the PR, including the archive sync. If the PR is merged before archive runs, the archive commits have to be authored on main after the fact — recoverable, but it loses the unified PR audit trail.

#### Local-merge variant (solo / local-only changes)

If finalize chose Option 1 (Merge locally), the skill performs the merge inline and removes the worktree. `/opsx:archive` then runs on main directly. This inverts the archive/merge order vs. the canonical path. Acceptable for solo or local-only changes where the PR audit trail isn't relevant.

**Source phase used.** OpenSpec `/opsx:archive` (or `openspec archive`).

**Step not used / replaced and why.** Superpowers has no equivalent step. Archive is OpenSpec's mechanism for promoting a change's delta specs into the living specs tree and freezing the change directory; it is fundamental to the spec-driven workflow and is not replaced by anything.

---
````

### Step 5: Verify edits

Run:

```bash
grep -nE "six phases|ten concrete steps|Phase 5: Finalization|Phase 6: Archival|Step 9\. Finalize|Step 10\. Archive|/opsx:continue → finalize → /opsx:archive|Canonical PR-review golden path|Local-merge variant" /Users/homer/dev/superspec/docs/workflow-details.md
```

Expected: at least 8 matches.

Run: `grep -n "five phases\|nine concrete\|finish / archive boundary\|finishing-a-development-branch and archive" /Users/homer/dev/superspec/docs/workflow-details.md`
Expected: zero matches.

### Step 6: Trailing whitespace + final newline

Run: `grep -nE ' +$' docs/workflow-details.md` — no output.
Run: `tail -c 1 docs/workflow-details.md | xxd` — `0a`.

### Step 7: Commit

```bash
git add docs/workflow-details.md
git commit -m "docs: split Phase 5 into Finalization + Archival for v3

Top intro updated from 5 phases / 9 steps to 6 phases / 10 steps;
at-a-glance table grows by one row. Step 7 convergence-loop diagram
and PASS termination bullet now reference finalize via /opsx:continue
instead of finishing-a-development-branch directly. Phase 5 splits
cleanly into Phase 5 Finalization (Step 9: finalize artifact) and
Phase 6 Archival (Step 10: /opsx:archive); Phase 6 documents the
canonical PR-review golden path with archive-before-merge ordering
and the local-merge variant callout."
```

---

## Task 8: Update `docs/project-layout.md` + root `README.md`

These two are small enough to bundle in one commit.

### Step 1: Update project-layout.md — templates tree

Use Edit to replace the templates tree block:

```text
        └── templates/             # markdown stubs per artifact
            ├── brainstorm.md
            ├── proposal.md
            ├── design.md
            ├── spec.md
            ├── tasks.md
            ├── plan.md
            ├── apply.md
            └── verify.md
```

with:

```text
        └── templates/             # markdown stubs per artifact
            ├── brainstorm.md
            ├── proposal.md
            ├── design.md
            ├── spec.md
            ├── tasks.md
            ├── plan.md
            ├── apply.md
            ├── verify.md
            └── finalize.md
```

### Step 2: Update project-layout.md — per-change-directory description

Use Edit to replace:

```markdown
Once you start working on changes, OpenSpec adds `openspec/changes/<change-name>/` directories at the same level as `schemas/`, holding the per-change `proposal.md`, `design.md`, `specs/`, `tasks.md`, `plan.md`, `apply.md`, and `verify.md` artifacts. After archival, completed changes move into `openspec/changes/archive/`.
```

with:

```markdown
Once you start working on changes, OpenSpec adds `openspec/changes/<change-name>/` directories at the same level as `schemas/`, holding the per-change `proposal.md`, `design.md`, `specs/`, `tasks.md`, `plan.md`, `apply.md`, `verify.md`, and `finalize.md` artifacts. After archival, completed changes move into `openspec/changes/archive/`.
```

### Step 3: Update root README.md — tagline

Use Edit to replace:

```markdown
  MIT licensed · Schema version 2 · Requires OpenSpec + Superpowers
```

with:

```markdown
  MIT licensed · Schema version 3 · Requires OpenSpec + Superpowers
```

### Step 4: Update root README.md — step-by-step Quick Start

Find the existing block in the step-by-step Quick Start section:

```markdown
/opsx:apply            # Worktree + Superpowers TDD loop — produces the actual code and writes apply.md (the v2 receipt)
/opsx:verify           # Validate implementation matches the delta specs and tasks (requires apply.md to exist)
/opsx:archive          # Sync the change's delta specs into project specs, then archive
```

Use Edit with `replace_all: true` to update both occurrences (this same block appears in both the step-by-step flow and the fast-forward flow). Replace with:

```markdown
/opsx:apply            # Worktree + Superpowers TDD loop — produces the actual code and writes apply.md (the v2 receipt)
/opsx:verify           # Validate implementation matches the delta specs and tasks (requires apply.md to exist)
/opsx:continue         # → finalize (invokes superpowers:finishing-a-development-branch, writes finalize.md; v3)
/opsx:archive          # Sync the change's delta specs into project specs, then archive
```

### Step 5: Verify edits

Run:

```bash
grep -n "Schema version 3\|finalize.md\|finishing-a-development-branch.*finalize.md" /Users/homer/dev/superspec/README.md /Users/homer/dev/superspec/docs/project-layout.md
```

Expected: matches in both files (project-layout.md mentions finalize.md, README.md mentions Schema version 3 + the /opsx:continue Quick Start line).

Run:

```bash
grep -n "Schema version 2" /Users/homer/dev/superspec/README.md
```

Expected: zero matches.

### Step 6: Trailing whitespace + final newline

Run:

```bash
grep -nE ' +$' /Users/homer/dev/superspec/README.md /Users/homer/dev/superspec/docs/project-layout.md
```

Expected: no output.

Run:

```bash
for f in /Users/homer/dev/superspec/README.md /Users/homer/dev/superspec/docs/project-layout.md; do printf '%s: ' "$f"; tail -c 1 "$f" | xxd | head -c 30; echo; done
```

Expected: each file ends with `0a`.

### Step 7: Commit

```bash
git add README.md docs/project-layout.md
git commit -m "docs: bump root README and project-layout for v3

Tagline 'Schema version 2' -> 'Schema version 3'. Quick Start blocks
add a /opsx:continue line between /opsx:verify and /opsx:archive
mentioning finalize. project-layout.md lists finalize.md in both the
templates tree and the per-change-directory description."
```

---

## Task 9: Repo-wide consistency sweep

**Files:** all repo markdown + the schema YAML

This is the final verification task. Read-only checks except for an optional pre-commit auto-fix commit.

### Step 1: Confirm no stale references to schema version 2 remain in current-state assertions

Run:

```bash
cd /Users/homer/dev/superspec
grep -rn "Schema version 2\|sdd-plus-superpowers\` v2\|version: 2" --include="*.md" --include="*.yaml" .
```

Expected: matches only in:

- `docs/superpowers/specs/2026-05-16-apply-artifact-and-verify-requires-design.md` (historical, the v2 design)
- `docs/superpowers/plans/2026-05-16-apply-artifact-and-verify-requires.md` (historical)
- `docs/superpowers/specs/2026-05-17-clarify-post-verify-workflow-design.md` (mentions v2 baseline in context)
- `docs/superpowers/plans/2026-05-17-clarify-post-verify-workflow.md` (this plan itself)
- `openspec/schemas/superspec/INTEGRATION.md` (Migration from schema v2 subsection — historical)
- `openspec/schemas/superspec/README.md` (decision log entry mentioning the v2 problem)

All other matches are missed edits. Specifically, none of these files should still claim the current schema is v2: `schema.yaml`, `INTEGRATION.md` header, `README.md` schema/root tagline, `workflow.md`, `workflow-details.md`.

If any match is in an unexpected location, stop and fix it before continuing.

### Step 2: Confirm verify.requires is consistently described as `[apply]` and finalize.requires as `[verify]`

Run:

```bash
grep -rn "finalize\.\?requires\|finalize.*requires" --include="*.md" --include="*.yaml" .
```

Expected: matches in `schema.yaml` (`requires:\n      - verify` under finalize), the v3 design spec, this plan, INTEGRATION.md, schema README, workflow-details.md, etc. Every match's description of finalize's requires should say `[verify]` or `verify` — never anything else.

Run:

```bash
grep -rn "verify.*finalize\|finalize.*verify" --include="*.md" .
```

Spot-check that wherever both terms appear together, the ordering is verify-then-finalize (because finalize requires verify, not the other way around).

### Step 3: Confirm finalize.md is referenced in all required doc surfaces

Run:

```bash
grep -rn "finalize\.md" --include="*.md" --include="*.yaml" .
```

Expected: matches in:

- `openspec/schemas/superspec/schema.yaml` (template field + instruction)
- `openspec/schemas/superspec/templates/finalize.md` (the template itself)
- `openspec/schemas/superspec/templates/verify.md` (convergence loop reminder PASS bullets)
- `openspec/schemas/superspec/INTEGRATION.md` (DAG diagram, walkthrough, design choice, migration)
- `openspec/schemas/superspec/README.md` (workflow, table, decision log)
- `docs/workflow.md` (phase summary)
- `docs/workflow-details.md` (Phase 5 + 6)
- `docs/project-layout.md` (templates tree + per-change description)
- `README.md` (Quick Start /opsx:continue line)
- `docs/superpowers/specs/2026-05-17-clarify-post-verify-workflow-design.md` (spec)
- `docs/superpowers/plans/2026-05-17-clarify-post-verify-workflow.md` (this plan)

If any required documentation file is missing a finalize.md reference, an edit was missed in the corresponding task.

### Step 4: Re-run the schema structural validator

Run:

```bash
cd /Users/homer/dev/superspec
python3 -c "import yaml; d=yaml.safe_load(open('openspec/schemas/superspec/schema.yaml')); ids=[a['id'] for a in d['artifacts']]; print(ids); assert d['version']==3; assert 'finalize' in ids; assert next(a['requires'] for a in d['artifacts'] if a['id']=='finalize')==['verify']; assert next(a['requires'] for a in d['artifacts'] if a['id']=='verify')==['apply']; assert d['apply']['requires']==['plan']; print('OK')"
```

Expected:

```
['brainstorm', 'proposal', 'design', 'specs', 'tasks', 'plan', 'apply', 'verify', 'finalize']
OK
```

### Step 5: Re-run the zod-equivalent hand-walk (cycle / dangling-requires check)

Run:

```bash
cd /Users/homer/dev/superspec
python3 << 'PY'
import yaml
d = yaml.safe_load(open('openspec/schemas/superspec/schema.yaml'))
ids = [a['id'] for a in d['artifacts']]
assert len(set(ids)) == len(ids), f"duplicate ids: {ids}"
for a in d['artifacts']:
    for k in ('id','generates','description','template'):
        assert a.get(k), f"missing {k} in {a.get('id')}"
    for r in a.get('requires', []):
        assert r in ids, f"unknown require: {r}"
def has_cycle(adj, start, seen, stack):
    seen.add(start); stack.add(start)
    for dep in adj.get(start, []):
        if dep not in seen:
            if has_cycle(adj, dep, seen, stack): return True
        elif dep in stack:
            return True
    stack.discard(start)
    return False
adj = {a['id']: a.get('requires', []) for a in d['artifacts']}
seen, stack = set(), set()
for n in ids:
    if n not in seen and has_cycle(adj, n, seen, stack):
        raise SystemExit(f"cycle from {n}")
print("OK")
PY
```

Expected: `OK`.

### Step 6: Run pre-commit on the whole tree

Run:

```bash
cd /Users/homer/dev/superspec
pre-commit run --all-files
```

Expected: all hooks pass. If pre-commit isn't installed, note it as a concern in the final report but do not fail this task — it is an optional local check.

If pre-commit auto-fixed any files, stage them and create a single cleanup commit:

```bash
git add -A
git commit -m "chore: pre-commit auto-fixes for v3 rollout"
```

Otherwise, no commit is needed at this step.

### Step 7: Confirm branch state

Run:

```bash
cd /Users/homer/dev/superspec
git log --oneline spec/apply-artifact-and-verify-requires..HEAD
```

Expected: roughly 9 commits added on this branch beyond the parent branch's tip — the spec commit (`011f8cd`) plus one commit per Task 1–8 plus an optional pre-commit cleanup commit. Order should follow the file-map progression:

1. `011f8cd docs: design spec for promoting finalize to a v3 DAG artifact`
2. Task 1: `feat(schema): add finalize.md template for v3 finalize artifact`
3. Task 2: `feat(schema): expand verify.md convergence reminder with PASS bullets`
4. Task 3: `feat(schema): promote finalize to v3 DAG artifact`
5. Task 4: `docs(schema): update INTEGRATION for v3 finalize artifact`
6. Task 5: `docs(schema): update schema README workflow + table for v3`
7. Task 6: `docs: workflow.md splits Phase 5 into Finalization + Archival`
8. Task 7: `docs: split Phase 5 into Finalization + Archival for v3`
9. Task 8: `docs: bump root README and project-layout for v3`
10. (Optional) `chore: pre-commit auto-fixes for v3 rollout`

If any commit is missing or out of order, surface it but do not attempt a rebase — the order is informational, not blocking.

### Step 8: Final report

The task is DONE if Steps 1–7 are clean. Report includes:

- Output of each grep / python step verbatim (or summarized if very long).
- The `git log --oneline` output.
- Any concerns about doc coverage, remaining drift, or pre-commit unavailability.

---

## Self-Review Notes

**Spec coverage:**

| Spec section | Implemented in |
|---|---|
| Schema changes (machine-readable diff sketch) | Task 3 (Steps 1–5) |
| New file `templates/finalize.md` | Task 1 |
| Modified file `templates/verify.md` | Task 2 |
| Doc updates table | Tasks 4–8 (one task per file group, except project-layout + root README bundled in Task 8) |
| Phase 5/6 split documentation | Task 6 (workflow.md) + Task 7 (workflow-details.md) |
| Canonical PR-review golden path | Task 4 Step 5 (INTEGRATION.md) + Task 7 Step 4 (workflow-details.md) |
| Local-merge variant callout | Task 4 Step 5 + Task 7 Step 4 |
| Migration from schema v2 | Task 4 Step 7 |
| Testing | Task 3 (Steps 6–7) + Task 9 (Steps 4–5) |
| Edge cases (verify FAIL with verify.md present) | Documented in spec; encoded in verify.md template update (Task 2) which keeps the loop reminder visible |
| Risks (drift between finalize artifact instruction and templates/finalize.md) | Implicit; finalize's instruction in Task 3 Step 2 is intentionally short and focused on the invocation + receipt fields |
| Out of scope (loop-runner command, /opsx:finish, upstream OpenSpec changes) | Documented in the spec; no tasks |

**Placeholder scan:** every code block above is concrete; no TBDs, no "TODO", no "implement similar to X". Every grep / python command states an exact expected outcome.

**Type consistency:** field names used in instructions match the templates (`Outcome`, `Branch state`, `Final state`, `PR URL`, `Workspace`, `Cleanup`, `Tests`, `Next step`). The artifact id `finalize` is used identically across the schema, all docs, and grep expectations. The phrase "DAG artifact" is consistent.

**Cross-version consistency:** v2 references that still belong (Migration from v2 in INTEGRATION.md; v2 mentions in the README decision log explaining the problem) are intentionally preserved; current-state v2 claims would be missed edits. Task 9 Step 1 distinguishes these.

---

## Manual follow-ups (outside this plan's automation)

These are recommended for the implementer to run by hand before opening the PR but are not automated steps because they require interactive tools:

1. `brew install openspec` (if not present), then `openspec validate --all --json` from the repo root — confirm the zod parser accepts schema v3.
2. Scaffold `openspec/changes/_test-finalize-artifact/` with stubs through `verify.md` (the v2 baseline). Confirm `openspec status --change _test-finalize-artifact --json` reports `finalize` as a `ready` node and that `/opsx:continue` surfaces the finalize artifact's instruction.
3. Delete the test change directory before pushing.
4. Regenerate `docs/assets/superspec-phases-flowchart.svg` for 6 phases if a regeneration pipeline exists in the repo or as a follow-up PR. The text references in `workflow.md` are updated by this plan; the SVG content is intentionally deferred.
