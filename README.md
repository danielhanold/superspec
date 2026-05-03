<p align="center">
  <img src="docs/assets/superspec-logo.png" alt="SuperSpec — Spec-driven development with superpowers" width="640">
</p>

<p align="center">
  Spec-driven workflow that connects <a href="https://github.com/Fission-AI/OpenSpec">OpenSpec</a> governance with <a href="https://github.com/obra/superpowers">Superpowers</a> execution discipline so a single change is fully traceable from idea → spec → TDD-verified code.
</p>

<p align="center">
  MIT licensed · Schema version 1 · Requires OpenSpec + Superpowers
</p>

---

## Why Superspec

**[OpenSpec](https://github.com/Fission-AI/OpenSpec)** turns feature ideas into versioned, reviewable specs — proposals, capability deltas, and tasks that live in the repo alongside the code.

**[Superpowers](https://github.com/obra/superpowers)** is a set of execution skills for coding agents — brainstorming, plan-writing, TDD, subagent dispatch, and code review — that enforce discipline during implementation.

The two overlap (both produce design and task artifacts) but focus on different domains: OpenSpec governs **spec-driven planning**, Superpowers governs **spec-driven development and implementation**. Used independently, you end up with duplicate documents, parallel task lists, and manual decisions about which skill to invoke at each step.

**Superspec is an opinionated integration of both**, with OpenSpec as the orchestrator. The artifact pipeline drives the workflow; each phase invokes the right Superpowers skill at the right time so the path from idea to TDD-verified code is traceable, reproducible, and auditable.

OpenSpec supports custom schemas, and Superspec is exactly that — a drop-in schema that picks the best of both frameworks and wires them together for a fully integrated workflow combining spec-driven and test-driven development. No fork of OpenSpec, no modification to Superpowers skills.

---

## Concepts

> **Looking for a phase-by-phase walkthrough?** See **[docs/workflow.md](docs/workflow.md)** — it covers what each of the nine phases does, why each one is required, which source system (OpenSpec or Superpowers) owns it, and which step from the other source it replaces.

### The five phases of a Superspec change

Every change moves through the same five phases, in order:

1. **Brainstorm** — nail down the idea for the change through a guided conversation.
2. **Artifact creation** — produce the proposal, optional design, [delta specs](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md#delta-specs), tasks, and the micro-task plan.
3. **Code implementation** — write the code in an isolated worktree using subagent-driven TDD.
4. **Spec validation** — verify the implementation matches the delta specs and tasks.
5. **Archival** — merge the change's delta specs into the project's living specs.

Each phase produces concrete artifacts in the change directory and (where applicable) hands off to a Superpowers skill. The full artifact-by-artifact mapping is below.

### Artifact pipeline

```
brainstorm  ──►  proposal  ──►  specs  ──►  tasks  ──►  plan  ──►  apply  ──►  verify  ──►  archive
                    │                                                              ▲
                    └─►  design (optional, may be pre-filled by brainstorm)  ──────┘
```

| Artifact | Output | Superpowers skill invoked | What it produces |
|---|---|---|---|
| `brainstorm` | `brainstorm.md` | `superpowers:brainstorming` | Validated design from a multi-turn conversation; may pre-fill `design.md` |
| `proposal` | `proposal.md` | — | Why / What Changes / Capabilities / Impact (50–1000 chars on Why) |
| `design` *(optional)* | `design.md` | — | Technical decisions, only when non-trivial |
| `specs` | `specs/<capability>/spec.md` | — | Delta specs (ADDED / MODIFIED / REMOVED / RENAMED) with `SHALL`/`MUST` requirements and `#### Scenario:` blocks |
| `tasks` | `tasks.md` | — | Coarse `- [ ] X.Y` checkboxes tracked during apply |
| `plan` | `plan.md` | `superpowers:writing-plans` | Micro-task breakdown (2–5 minute steps) |
| `apply` *(phase, not artifact)* | code + commits | `using-git-worktrees`, `subagent-driven-development` (which transitively triggers `test-driven-development` and `requesting-code-review`), `finishing-a-development-branch` | The implementation itself |
| `verify` | `verify.md` | — | Post-apply checks: structural validation, task completion, delta sync, design/spec coherence, no unstaged files |

### The 7 Superpowers touch points

| # | Skill | Hook | Trigger |
|---|---|---|---|
| 1 | `superpowers:brainstorming` | `brainstorm` artifact instruction | Direct |
| 2 | `superpowers:writing-plans` | `plan` artifact instruction | Direct |
| 3 | `superpowers:using-git-worktrees` | apply step 1 | Direct |
| 4 | `superpowers:subagent-driven-development` | apply step 2a | Direct |
| 5 | `superpowers:test-driven-development` | inside #4 | **Transitive** |
| 6 | `superpowers:requesting-code-review` | inside #4 | **Transitive** |
| 7 | `superpowers:finishing-a-development-branch` | apply step 4 | Direct |

`subagent-driven-development` is the workhorse: the main agent reads `plan.md` and dispatches a fresh subagent for each micro-task. Each subagent enforces TDD (write failing test → minimum code to pass → refactor) and runs spec-compliance + code-quality review per task. A final review runs over the whole implementation before apply concludes.

**Fallback path:** `superpowers:executing-plans` is documented for harnesses without subagent support. On Claude Code, subagents *are* available, so 2a is the default. If you're forced to fall back, you must manually maintain TDD discipline and invoke `superpowers:requesting-code-review` yourself.

### Fallback strategy

If a Superpowers skill is unavailable (not installed, version mismatch), each artifact instruction includes a manual fallback:

- `brainstorm` → manually write `brainstorm.md`
- `plan` → manually write `plan.md`
- `apply` → standard task-by-task manual implementation

Superspec degrades gracefully to plain OpenSpec when the execution layer is missing.

---

## Installation

**Prerequisites**

- A git repository (run `git init` if you don't have one yet).
- An agent harness — Claude Code, Cursor, GitHub Copilot, OpenAI Codex, or Gemini.
- Homebrew on macOS for the `brew install openspec` step. On other platforms, install OpenSpec from its [release page](https://github.com/Fission-AI/OpenSpec/releases).

### 1. Install Superpowers

Install for your agent harness — globally, or scoped to the current repository. Follow the instructions in the Superpowers README:

```bash
open https://github.com/obra/superpowers
```

### 2. Install OpenSpec

```bash
brew install openspec
```

### 3. Initialize OpenSpec for your harness

The example below initializes for **Cursor**. For other harnesses (Claude Code, Copilot, Codex, Gemini), run `openspec init --help` to see the supported `--tools` values and follow the guided prompts.

```bash
openspec init --tools cursor --profile custom
```

### 4. Copy the SuperSpec schema into `openspec/schemas`

Run from the root of your local git repository:

```bash
git clone --depth 1 --filter=blob:none --sparse \
  https://github.com/danielhanold/superspec.git /tmp/superspec-tmp && \
  cd /tmp/superspec-tmp && \
  git sparse-checkout set openspec/schemas/superspec && \
  cd - && \
  mkdir -p openspec/schemas && \
  mkdir -p openspec/schemas/superspec && \
  cp -r /tmp/superspec-tmp/openspec/schemas/superspec/. openspec/schemas/superspec/ && \
  rm -rf /tmp/superspec-tmp
```

The trailing `/.` in `superspec/.` copies the directory's *contents* into the existing `openspec/schemas/superspec/`, avoiding a nested `superspec/superspec/`.

### 5. Set SuperSpec as the default schema

```bash
echo "schema: superspec" > openspec/config.yaml
```

### 6. Verify the install

```bash
openspec schemas      # superspec should appear in the list
openspec validate     # should pass with no errors
```

---

## Quick start

Once installed, you have two flows.

### Quick flow (recommended)

```bash
/opsx:ff my-feature    # End-to-end: brainstorm + proposal + design + specs + tasks + plan
/opsx:apply            # worktree + subagent-driven-development
/opsx:archive          # archive
```

### Step-by-step flow

```bash
/opsx:new my-feature --schema superspec
/opsx:continue         # → brainstorm (interactive conversation)
/opsx:continue         # → proposal
/opsx:continue         # → design (optional, only when technical decisions need explanation)
/opsx:continue         # → specs
/opsx:continue         # → tasks
/opsx:continue         # → plan
/opsx:apply
/opsx:archive
```

The `/opsx:` slash commands ship with your harness's OpenSpec integration, not with this schema. If your harness uses different command names, check its OpenSpec docs.

To skip Superspec for a single change and use the upstream schema instead:

```bash
/opsx:new my-simple-fix --schema spec-driven
```

---

## Project layout (after install)

```
openspec/
├── config.yaml                    # schema: superspec
└── schemas/
    └── superspec/
        ├── schema.yaml            # artifact pipeline + instructions
        ├── README.md              # design rationale (this schema)
        ├── INTEGRATION.md         # lifecycle + CLI cheat sheet
        └── templates/             # markdown stubs per artifact
            ├── brainstorm.md
            ├── proposal.md
            ├── design.md
            ├── spec.md
            ├── tasks.md
            ├── plan.md
            └── verify.md
```

---

## Further reading

- [`docs/workflow.md`](docs/workflow.md) — phase-by-phase walkthrough of the nine Superspec phases (mental model, why required, source phase, replaced step)
- [`openspec/schemas/superspec/README.md`](openspec/schemas/superspec/README.md) — design motivation and rationale
- [`openspec/schemas/superspec/INTEGRATION.md`](openspec/schemas/superspec/INTEGRATION.md) — full lifecycle, CLI cheat sheet, and design choices
- [`openspec/schemas/superspec/schema.yaml`](openspec/schemas/superspec/schema.yaml) — machine-readable schema
- [Fission-AI/OpenSpec](https://github.com/Fission-AI/OpenSpec) — upstream OpenSpec
- [obra/superpowers](https://github.com/obra/superpowers) — Superpowers skills

---

## Credits

Superspec is based on [JiangWay/OpenSpec — `schemas/sdd-plus-superpowers`](https://github.com/JiangWay/OpenSpec/tree/main/schemas/sdd-plus-superpowers), which originated the integration of OpenSpec's spec-driven workflow with Superpowers execution skills. This repository repackages that schema as a standalone, drop-in addition for any OpenSpec project.

---

## License

[MIT](LICENSE) © 2026 Daniel Hanold.
