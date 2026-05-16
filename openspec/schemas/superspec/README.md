# sdd-plus-superpowers Schema

Integrates OpenSpec's artifact governance workflow with Superpowers' execution skills into a single unified workflow.

## What This Schema Solves

OpenSpec manages the "what" (proposal → specs → design → tasks), while Superpowers manages the "how" (brainstorming, writing-plans, subagent-driven-development). Both are excellent on their own, but three structural problems emerge when alternating between them during real development:

1. **Duplicate outputs** — Brainstorming produces design documents in the Superpowers directory (`docs/superpowers/specs/`), and OpenSpec writes proposal/design again in the change directory, with highly overlapping content.
2. **Task fragmentation** — OpenSpec's `tasks.md` (coarse-grained checkboxes) and Superpowers' plan (micro TDD steps) describe the same work, but their format, location, and status tracking are each independent.
3. **Manual orchestration** — Users must decide on their own "which skill should I use now"; there is no automatic handoff between the two systems.

### Why a Custom Schema Instead of Modifying Existing Skills

Two alternatives were considered:

- **Adding custom fields to config.yaml** (e.g., `skill_bindings`): The OpenSpec CLI doesn't recognize these fields — no validation, no discoverability, and multiple SKILL.md files would need modification to read them.
- **Directly modifying opsx skill files**: Highly invasive, affects all changes, and gets overwritten on SKILL.md upgrades.

A custom schema leverages OpenSpec's **natively supported project-level schema mechanism**:
- The CLI validates the schema structure
- `openspec schemas` lists it automatically
- Each change can independently select a schema (`--schema spec-driven` or `--schema sdd-plus-superpowers`)
- No existing SKILL.md or command files are modified

---

## Workflow Overview

```text
brainstorm ──→ proposal ──→ specs ──→ tasks ──→ plan ──→ apply ──→ verify
                  │                     ↑
                  └──→ design ──────────┘
```

Differences from `spec-driven`:

| | spec-driven | sdd-plus-superpowers (v2) |
|---|---|---|
| Starting point | proposal (written manually) | **brainstorm** (invokes brainstorming skill) |
| Endpoint | tasks (coarse-grained) | **verify** (post-implementation report; apply is the executable midpoint) |
| apply requires | tasks | **plan** |
| apply method | Standard task-by-task | **worktree + subagent-driven-development** |
| Additional artifacts | — | brainstorm, plan, **apply (receipt)** |
| verify requires | tasks | **apply** (v2 — was plan in v1) |

---

## Integrated Superpowers Skills

| Schema Phase | Superpowers Skill Invoked | Trigger |
|------------|------------------------|---------|
| brainstorm artifact | `superpowers:brainstorming` | artifact instruction |
| plan artifact | `superpowers:writing-plans` | artifact instruction |
| apply phase | `superpowers:using-git-worktrees` | apply instruction |
| apply phase | `superpowers:subagent-driven-development` | apply instruction |
| post-apply | `superpowers:finishing-a-development-branch` | apply instruction |

All integrations are achieved through the `instruction` field in schema.yaml — directing the AI to invoke the corresponding skill at the appropriate time via the Skill tool. No Superpowers skill files are modified.

### Output Redirection

Superpowers skills have default output paths (e.g., brainstorming writes to `docs/superpowers/specs/`). In this schema, artifact instructions include redirection directives that tell the invoked skill to write output into the change directory:

- brainstorming → `openspec/changes/<name>/brainstorm.md` (+ optional `design.md`)
- writing-plans → `openspec/changes/<name>/plan.md`

This is achieved through context injection (appending directives when invoking the skill), not by modifying skill code.

---

## Usage

### Quick Flow (Recommended)
```bash
/opsx:ff my-feature    # End-to-end: create directory + brainstorm + proposal + design + specs + tasks + plan
/opsx:apply            # worktree + subagent-driven-development
/opsx:archive          # archive
```

### Step-by-Step Flow
```bash
/opsx:new my-feature --schema sdd-plus-superpowers
/opsx:continue         # → brainstorm (interactive conversation)
/opsx:continue         # → proposal
/opsx:continue         # → design (optional, only when technical decisions need explanation)
/opsx:continue         # → specs
/opsx:continue         # → tasks
/opsx:continue         # → plan
/opsx:apply
/opsx:archive
```

### Switching Back to spec-driven
```bash
# Use a different schema for a single change
/opsx:new my-simple-fix --schema spec-driven

# Or change the project default
# openspec/config.yaml: schema: spec-driven
```

---

## Design Decision Log

### Why brainstorm Is an Artifact, Not a Hook

Brainstorming is an interactive multi-turn conversation that requires user participation. Making it the first artifact rather than a schema-level hook has two benefits:
1. **Skippable** — If the user already knows what to build, they can create brainstorm.md directly without invoking the skill
2. **Trackable** — `openspec status` can show whether brainstorm is complete, and subsequent artifacts have explicit dependency relationships

### Why plan Is Separate from tasks

`tasks.md` contains coarse-grained checkboxes ("Add PdfServiceTest"), while `plan.md` contains micro steps ("Create test skeleton → Write downloadPdf test → Run → Commit"). The two differ in granularity and purpose:
- `tasks.md` → Tracks overall progress (the apply phase's `tracks` field parses checkboxes)
- `plan.md` → Guides subagents through step-by-step implementation (input for the executor)

The apply phase requires `plan` rather than `tasks` because the executor needs micro steps to work effectively. However, `tracks: tasks.md` ensures progress is still tracked via coarse-grained checkboxes.

### Why apply Is Both an Artifact and a Phase Block (v2)

The OpenSpec CLI builds its DAG strictly from the `artifacts:` list, and the `ApplyPhase` zod schema has no `generates` field — so the apply phase alone cannot mark itself complete or be referenced by another artifact's `requires`. In v1 this manifested as `verify.requires: [plan]` plus an inline comment saying "actually verify needs apply, but the schema can't say so." Agents predictably ran verify before apply.

v2 solves this by representing apply twice: a real artifact (`generates: apply.md`, `requires: [plan]`) so `verify.requires: [apply]` is honest, *and* the existing `apply:` top-level block so `/opsx:apply` CLI behavior is unchanged (the handler reads `schema.apply.instruction` and would not see the artifact's instruction). To avoid drift, the canonical body lives in the `apply:` block; the apply artifact's instruction is a short redirect.

### Fallback Strategy

If Superpowers skills are unavailable (not installed, version incompatible, etc.), each instruction includes a fallback path:
- brainstorm → Manually write brainstorm.md
- plan → Manually write plan.md
- apply → Standard task-by-task manual implementation
