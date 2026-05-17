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
