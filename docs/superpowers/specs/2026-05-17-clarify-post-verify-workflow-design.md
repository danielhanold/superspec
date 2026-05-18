# Promote `finalize` to a Schema Artifact and Clarify the Post-Verify Workflow

**Date:** 2026-05-17
**Schema:** `superspec`
**Target version bump:** `2 → 3`
**Branch:** `spec/clarify-post-verify-workflow` (off `spec/apply-artifact-and-verify-requires`, which is in PR #3)
**Parent change:** PR #3 promoted `apply` from a phase-only block to a DAG artifact (schema v2). This change applies the same pattern one phase later.

## Problem

After v2, the DAG honestly expresses `plan → apply → verify`. The workflow stops there: nothing in the schema, templates, or convergence-loop documentation tells the agent what to invoke after verify reports PASS. The post-verify guidance that does exist lives in two places that the agent typically does not re-read after running `/opsx:verify`:

- `apply:` block step 5 (Completion) says "invoke `superpowers:finishing-a-development-branch`".
- `workflow-details.md` Phase 5 (Archival) documents the full `finishing-a-development-branch` → `/opsx:archive` ordering.

The verify artifact itself, the verify.md template's Next step block, and the convergence-loop reminder all treat PASS as "you're done" without naming the next call site. In practice this leads to agents (and humans) jumping directly to `/opsx:archive`, skipping the git-side closeout — no PR is created, no worktree is cleaned up, no branch is merged. The branch closeout has to be invoked manually before archive each time.

A second, related ambiguity: the schema does not capture the canonical post-finalize golden path that the project actually uses. In particular, `/opsx:archive` makes commits on the local branch (sync delta specs, move change directory) — so in the PR-review case, archive must run *before* the PR merges, with its commits pushed to update the PR, and the merge then lands everything in one move. This sequencing is not documented anywhere today.

## Goals

- The DAG honestly expresses `plan → apply → verify → finalize`. The schema names "finalize" as the post-verify step.
- `/opsx:continue` becomes the OPSX-vocabulary entry point that surfaces the finalize instruction; no new slash command needed, no upstream OpenSpec change required.
- finalize.md is a minimal receipt of the git-side closeout (option chosen, PR URL if any, final branch state, worktree cleanup status).
- The canonical PR-review golden path is documented end-to-end in workflow-details.md, including the `/opsx:archive`-before-merge ordering.
- The local-merge path is documented as a minor variant (acceptable for solo / local-only changes) without forcing developers to memorise a non-standard inline-archive step.

## Non-goals

- Modifying `/opsx:archive` itself or any other OpenSpec CLI command. Archive remains a pure spec-side operation (sync deltas + move change dir); we do not try to make it merge git branches.
- Adding a literal `/opsx:finish` workflow to OpenSpec. That would be an upstream OpenSpec change; `/opsx:continue` already gives us the entry point we need.
- Adding new artifacts beyond finalize. The retrospective recommendation is moved into finalize's instruction rather than promoted to its own artifact.
- Tightening verify's check #5 (the option discussed earlier in brainstorming). The DAG dependency on finalize handles the gate cleanly; an additional verify-side gate would be belt-and-suspenders without proportionate value.

## Approach

Add a new artifact `finalize` to the DAG, mirroring exactly what v2 did for `apply`. The DAG becomes:

```text
brainstorm → proposal → specs → tasks → plan → apply → verify → finalize
```

With `design` still an optional leaf off brainstorm, and `/opsx:archive` as the lifecycle close that follows finalize (not in the DAG; it is a CLI command).

**Why this works without upstream OpenSpec changes:**

- The OpenSpec CLI's `/opsx:continue` walks `schema.artifacts` and surfaces the next ready artifact's instruction. Once verify is done, `/opsx:continue` lands on finalize and tells the agent to invoke `superpowers:finishing-a-development-branch`. No new workflow command is needed.
- `openspec validate` only validates schema structure (parse, requires resolution, no cycles). It does NOT gate on artifact completeness. So adding finalize does not create a chicken-and-egg with verify check #1 (which runs `openspec validate --all`).
- `/opsx:archive` is unchanged; the workflow docs are the source of the "archive runs after finalize" sequence, and the schema gives the call site for finalize via `/opsx:continue`.

## CLI behavior dependencies (verified against Fission-AI/OpenSpec `main`)

Carried over from PR #3's spec, still accurate:

1. `validateRequiresReferences` rejects any `requires:` entry that does not match an `artifacts[*].id`. Therefore `finalize.requires: [verify]` is only valid because `verify` is in `artifacts:`.
2. `detectCompleted` walks `schema.artifacts` and checks `generates` file existence. `finalize` becomes a real DAG node by virtue of generating `finalize.md`.
3. `/opsx:continue` walks the DAG via `getNextArtifacts(completed)` and surfaces each artifact's own `instruction`. This is the CLI surface for finalize.
4. `/opsx:archive` is implemented in `src/commands/archive.ts` (not re-read for this design, but its observable behavior — validate + check task completion + sync delta specs + move change dir — has not changed) and does NOT consult the DAG for completeness. It will run even if finalize.md is missing. That is expected: the social/workflow convention is what enforces the ordering, not a CLI gate.

## Schema changes (machine-readable diff sketch)

```yaml
name: SuperSpec
version: 3              # was 2
description: |
  ... existing description ...
  v3: finalize is promoted from a manual post-verify step (documented
  inside the apply: block) to a real DAG artifact. /opsx:continue
  surfaces its instruction after verify completes, which tells the
  agent to invoke superpowers:finishing-a-development-branch and
  record the outcome in finalize.md. /opsx:archive is unchanged and
  runs after finalize.

artifacts:
  # ... brainstorm, proposal, design, specs, tasks, plan, apply unchanged ...

  - id: verify
    # ... 5 checks unchanged ...
    instruction: |
      ... preserved verbatim through the 5 checks ...

      Recommended convergence loop:
      - PASS → /opsx:continue advances to the finalize artifact; invoke
        superpowers:finishing-a-development-branch through it. /opsx:archive
        runs once finalize.md exists.
      - PASS_WITH_WARNINGS → same path as PASS; warnings are recorded
        but do not block.
      - FAIL with code-fixable items → return to apply, re-run, re-verify.
      - FAIL with artifact-level items (e.g. spec drift) → fix the
        offending artifact, then re-enter apply.
      - Iteration > 5 → stop looping; report to the user.

      See docs/workflow-details.md for the full convergence pattern and
      the canonical PR-review golden path.
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
  requires: [plan]    # unchanged
  tracks: tasks.md
  instruction: |
    # Steps 0-4 unchanged.
    # Step 5 (Completion) is REMOVED. finalize is the canonical home
    # for the finishing-a-development-branch invocation, and
    # /opsx:continue surfaces it after the convergence loop ends.
    #
    # The "Recommended — Retrospective before archive" section is
    # also MOVED out of this block into finalize's instruction
    # (above). The retrospective is logically pre-archive, not part
    # of apply.
```

The body of the existing `apply:` block instruction is preserved verbatim through step 4 (Verification). Step 5 (Completion) and the trailing "Recommended — Retrospective before archive (non-blocking)" subsection are removed from the apply: block.

## New file: `templates/finalize.md`

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

`<varies by outcome — see below>`

Outcome-specific Next step wording (the template's `<varies by outcome>`
placeholder is filled in by the agent at finalize time):

- `merge-locally`: "Run `/opsx:archive` on main to sync delta specs and
  move the change directory into the archive."
- `pr-created`: "Wait for PR review. After approval, run `/opsx:archive`
  on this feature branch (commits land here), push the archive commits
  to update the PR, then merge the PR (`gh pr merge --squash --delete-branch`
  or GitHub UI) — that single merge lands both the implementation and
  the archive into main."
- `kept-as-is`: "Resume later. Re-enter finalize when ready, or pick up
  where you left off; the worktree is preserved."
- `discarded`: "No further action; the change directory may also be
  discarded if appropriate."
```

## Modified file: `templates/verify.md`

Expand the existing Convergence loop reminder blockquote to include PASS bullets pointing at finalize:

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

The Iteration field directly under "Verified at" is unchanged.

## Workflow model — six phases (was five)

| Phase | # | Step | Brief why | Owner |
|---|---:|---|---|---|
| 1. Brainstorm | 1 | brainstorm | Avoid building the wrong thing. | Superpowers |
| 2. Artifact Creation | 2 | proposal | Define the change boundary. | OpenSpec |
| | 3 | design *(optional)* | Explain the chosen solution. | Hybrid |
| | 4 | specs | Create the testable contract. | OpenSpec |
| | 5 | tasks | Scope the work. | OpenSpec |
| | 6 | plan | Make the work executable. | Superpowers |
| 3. Code Implementation | 7 | apply | Change the system. | Superpowers |
| 4. Spec Validation | 8 | verify | Prove it matches intent. | OpenSpec |
| **5. Finalization (NEW)** | **9** | **finalize** | **Close out the git side cleanly before archive.** | **Superpowers** |
| **6. Archival** | **10** | **archive boundary** | **Sync deltas and freeze the change.** | **OpenSpec** |

## Canonical PR-review golden path (Phase 6 prose)

```text
1. verify (developer; verify.md committed on the feature branch)
   ↓
2. finalize (developer; superpowers:finishing-a-development-branch
   creates the PR; finalize.md says pr-open; worktree preserved)
   ↓
3. [PAUSE: human review on the PR; reviewer approves]
   ↓
4. /opsx:archive on the feature branch
   (syncs delta specs into openspec/specs/, moves the change directory
   into openspec/changes/archive/. These are NEW commits on the
   feature branch — not on main.)
   ↓
5. Push the archive commits to update the PR
   (git push — the PR now contains the implementation commits plus
   the archive commits, all reviewable together.)
   ↓
6. PR merge
   (gh pr merge --squash --delete-branch  — or GitHub UI with
   "delete branch on merge" enabled. Single merge lands implementation
   and archive together; branch is deleted.)
   ↓
7. Local worktree cleanup if still present
   (cd to main repo root; git worktree remove .worktrees/<name>;
   git worktree prune. Only needed if the worktree wasn't auto-cleaned.)
```

**Why archive must run before the PR merge** — `/opsx:archive` makes new commits on whichever branch it runs on (the delta-spec sync and change-directory move). If the PR is merged first, those commits have nowhere natural to land — they would have to be authored directly on main after the fact, breaking the "everything in the PR for review" invariant. Running archive on the feature branch and pushing the commits to the PR keeps the audit trail in one place: the PR's diff contains every commit that went into the change.

## Local-merge variant (acceptable for solo / local-only changes)

If finalize chose Option 1 ("Merge locally") from `superpowers:finishing-a-development-branch`, the skill performs the merge inline and removes the worktree. The archive step then runs on main:

```text
1. verify (on feature branch)
2. finalize chose merge-locally; finishing-a-development-branch merged
   the feature branch into main and removed the worktree.
   finalize.md says outcome: merge-locally; final state: merged.
3. /opsx:archive on main (syncs delta specs and moves change dir;
   commits land on main directly).
```

This inverts the archive/merge order vs. the canonical path. It is acceptable for solo or local-only changes where the PR audit trail is not relevant. Developers using this variant should be aware that the archive commits will appear on main without a corresponding PR diff for them.

## Doc updates

| File | What changes |
|---|---|
| `openspec/schemas/superspec/INTEGRATION.md` | Section 2 (7 touch-points table): row 7 (`finishing-a-development-branch`) hook becomes "finalize artifact" instead of "apply step 5". Section 3 (DAG diagram): add finalize node between verify and archive. Section 4 (walkthrough): split current Step 3 (Apply) and Step 4 (Archive) into Step 3 (Apply), Step 4 (Finalization — new), Step 5 (Archive with canonical PR golden path). Section 6: new design choice "Finalize is a real artifact, not a hidden phase (v3)". New "Migration from v2" subsection. |
| `openspec/schemas/superspec/README.md` | Workflow overview diagram extends through finalize. Differences table grows by one row: `verify → finalize` relationship. Decision log: new entry "Why finalize Is a DAG Artifact (v3)". |
| `docs/workflow.md` | At-a-glance table grows from 5 phases (9 steps) to 6 phases (10 steps). Phase summary subsections: edit Phase 5, add new Phase 6 (Archival). |
| `docs/workflow-details.md` | Phase 5 (Archival) is split into Phase 5 (Finalization, single step) and Phase 6 (Archival, single step). Phase 5 documents the finalize artifact + finalize.md receipt. Phase 6 documents the canonical PR golden path (7 steps) plus the local-merge variant callout. Update convergence-loop diagram so its PASS branch shows `→ finalize → archive` instead of `→ finishing-a-development-branch → archive`. |
| `docs/project-layout.md` | Add `finalize.md` to the templates tree listing and to the per-change-directory description. |
| `README.md` (root) | Tagline `Schema version 2 → Schema version 3`. Quick Start: between `/opsx:verify` and `/opsx:archive`, add a line for `/opsx:continue` (which now surfaces finalize). Brief mention that finalize creates the PR / does the branch closeout. |
| `docs/assets/superspec-phases-flowchart.svg` | Regen for 6 phases. If the SVG is hand-authored, update the Phase 5 block to split into "Finalization" + "Archival". If it is generated from a source file in the repo, regenerate. Marked as deferrable if the regen pipeline is uncertain — workflow.md table is the canonical source. |

## Testing

- `openspec validate` against the new schema parses cleanly (zod) and validates the DAG (no cycle; all `requires` resolve; new finalize.requires: [verify] points at an existing artifact).
- Python zod-equivalent hand-walk (the same script used in PR #3) confirms:
  - `version == 3`
  - artifact ids `[brainstorm, proposal, design, specs, tasks, plan, apply, verify, finalize]`
  - `verify.requires == ['apply']`
  - `finalize.requires == ['verify']`
  - `apply (artifact).requires == ['plan']`
  - `schema.apply.requires == ['plan']`
  - No duplicate ids, no cycle.
- Hand-walk: in a project using this schema, after `/opsx:verify` completes on a sample change, `openspec status` should show `finalize` as the next `ready` artifact. `/opsx:continue` surfaces finalize's instruction.
- Doc-DAG consistency: grep for `verify` → finalize wiring in INTEGRATION.md, schema README, workflow.md, workflow-details.md. Every doc surface must agree.
- Template-instruction alignment: finalize's instruction text references field names that appear in templates/finalize.md.

## Edge cases

- **finalize.md missing iteration semantic** — finalize.md does NOT track iterations the way apply.md does. If the user re-enters finalize (e.g., from `kept-as-is` to `pr-created` later), finalize.md is simply overwritten. Documented in the template's intro blockquote.
- **PR merge before /opsx:archive** — explicitly documented as a workflow anti-pattern in Phase 6 prose. If a developer does this anyway, the archive commits would have to be authored on main after the merge, which is acceptable but loses the "all in the PR" audit trail. Recoverable, not a hard failure.
- **finalize chose `discarded`** — finalize.md records outcome: discarded; the change directory may also be discarded. `/opsx:archive` is typically skipped in this case. Documented as the discard variant in Phase 6.
- **OpenSpec validates an incomplete change** — `openspec validate` does not gate on artifact completeness. So a change with no finalize.md still passes validate. The completeness check happens at `openspec status` time and via the `/opsx:continue` walker. No regression vs. v2.
- **Verify FAIL with verify.md still on disk** — `/opsx:continue` advances on file existence, not on file contents. If verify wrote FAIL into verify.md and stopped, `/opsx:continue` will still surface finalize as the next ready artifact. The verify.md template's Convergence loop reminder mitigates this by explicitly telling the agent to return to apply on FAIL (overwriting verify.md when verify is re-run). The schema cannot enforce the FAIL-loops-back rule; the agent must read and follow the convergence-loop reminder.

## Migration from schema v2

In-flight v2 changes that have verify.md but no finalize.md will not progress through `/opsx:continue` under v3 — the walker will report finalize as the next ready artifact. Migration:

1. Author a minimal finalize.md by hand from `openspec/schemas/superspec/templates/finalize.md`. Fill the fields from the actual branch state:
   - `Outcome`: pick the option that matches what was already done (`merge-locally` if the branch was merged inline; `pr-created` if a PR exists; etc.).
   - `Branch state`: copy from `git status` / `git log` / `gh pr view`.
   - `Workspace`: record current worktree state.
   - `Tests`: record the test-baseline status if known.
2. Re-run `/opsx:continue` — it should now advance past finalize and the change is ready for `/opsx:archive`.

The Superspec repository itself has no `openspec/changes/` directory, so there is no internal migration work.

## Risks

- **Drift between finalize artifact instruction and templates/finalize.md** — same risk as v2's apply artifact / apply: block split, mitigated the same way: keep finalize's instruction focused on *what to invoke* and *what to record*; let templates/finalize.md own the field structure. Reviewers should reject PRs that drift these.
- **Developers skipping `/opsx:archive` before merging the PR** — the schema cannot enforce this; only docs and convention can. Phase 6 prose makes the "archive before merge" ordering explicit and explains why.
- **External consumers using schema v2** — the migration section gives a path; pinning to v2 also remains an option for projects that do not want to adopt finalize yet.

## Open questions

None known after brainstorming. The local-merge variant is intentionally documented as a minor variant rather than blessed as a parallel golden path; if real usage shows it is more common than expected, a follow-up change could promote it.

## Out of scope (deferred or upstream)

- Modifying `/opsx:archive` to merge PRs or close worktrees. That is an upstream OpenSpec design decision; archive remains spec-side only.
- Adding a literal `/opsx:finish` workflow command. `/opsx:continue` already provides the CLI vocabulary entry point for finalize; no new command is needed.
- Adding lint/tooling to enforce that finalize.md exists before `/opsx:archive`. Out of scope for this change.
- Promoting `retrospective.md` to a DAG artifact in its own right. Out of scope; it stays as a recommendation in finalize's instruction.
