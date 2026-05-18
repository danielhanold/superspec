<p align="center">
  <img src="docs/assets/superspec-logo.png" alt="SuperSpec — Spec-driven development with superpowers" width="640">
</p>

<p align="center">
  Spec-driven workflow that connects <a href="https://github.com/Fission-AI/OpenSpec">OpenSpec</a> governance with <a href="https://github.com/obra/superpowers">Superpowers</a> execution discipline so a single change is fully traceable from idea → spec → TDD-verified code.
</p>

<p align="center">
  MIT licensed · Schema version 3 · Requires OpenSpec + Superpowers
</p>

---

## About Superspec

**Superspec is an opinionated integration of OpenSpec and Superpowers**, with OpenSpec as the orchestrator. The artifact pipeline drives the workflow; each phase invokes the right Superpowers skill at the right time so the path from idea to TDD-verified code is traceable, reproducible, and auditable.

**[OpenSpec](https://github.com/Fission-AI/OpenSpec)** turns feature ideas into versioned, reviewable specs — proposals, capability deltas, and tasks that live in the repo alongside the code.

**[Superpowers](https://github.com/obra/superpowers)** is a set of execution skills for coding agents — brainstorming, plan-writing, TDD, subagent dispatch, and code review — that enforce discipline during implementation.

The two overlap (both produce design and task artifacts) but focus on different domains: OpenSpec governs **spec-driven planning**, Superpowers governs **spec-driven development and implementation**. Used independently, you end up with duplicate documents, parallel task lists, and manual decisions about which skill to invoke at each step.

OpenSpec supports custom schemas, and Superspec is exactly that — a drop-in schema that picks the best of both frameworks and wires them together for a fully integrated workflow combining spec-driven and test-driven development. No fork of OpenSpec, no modification to Superpowers skills.

---

## Concepts

> **Looking for the workflow map?** See **[docs/workflow.md](docs/workflow.md)** for the visual overview, or **[docs/workflow-details.md](docs/workflow-details.md)** for the full ten-step breakdown.

### The six phases of a Superspec change

Every change moves through the same six phases, in order:

1. **Brainstorm** — nail down the idea for the change through a guided conversation.
2. **Artifact creation** — produce the proposal, optional design, [delta specs](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md#delta-specs), tasks, and the micro-task plan.
3. **Code implementation** — write the code in an isolated worktree using subagent-driven TDD.
4. **Spec validation** — verify the implementation matches the delta specs and tasks.
5. **Finalization** — close out the git side (create PR / merge / clean up the worktree) and record the outcome in `finalize.md`.
6. **Archival** — merge the change's delta specs into the project's living specs and archive the change directory.

Each phase produces concrete artifacts in the change directory and (where applicable) hands off to a Superpowers skill.

---

## Installation

**Prerequisites**
- [Homebrew](https://brew.sh/) strongly preferred.

### Install OpenSpec and SuperPowers

#### Install OpenSpec
```bash
brew install openspec
```

If Homebrew is unavailable, [install OpenSpec via a different package manager](https://github.com/Fission-AI/OpenSpec/blob/main/docs/installation.md).

#### Install Superpowers
Install globally for your agent harness. Follow the [instructions in the Superpowers docs](https://github.com/obra/superpowers#installation).

### Configure OpenSpec

#### OpenSpec Global configuration (one-time only)

Requires [jq](https://github.com/jqlang/jq) to be installed for full automation.
Copy commands below and run.

```bash
# Backup and remove any existing global OpenSpec configurations.
OPENSPEC_CONFIG_PATH="$(openspec config path)"
OPENSPEC_CONFIG_PATH_BACKUP="${OPENSPEC_CONFIG_PATH}.$(date +%s).bkp"
if [[ -f "${OPENSPEC_CONFIG_PATH}" ]]; then
  printf "Found existing OpenSpec config: %s\n" "${OPENSPEC_CONFIG_PATH}"
  printf "Will back up existing config (%s), then delete.\n" "${OPENSPEC_CONFIG_PATH_BACKUP}"
  cp "${OPENSPEC_CONFIG_PATH}" "${OPENSPEC_CONFIG_PATH_BACKUP}"
  rm "${OPENSPEC_CONFIG_PATH}"
fi

# Uncomment to disable telemtry
# OPENSPEC_TELEMETRY=0

# Create global OpenSpec config with core profile.
openspec config profile core

# Set correct profile and delivery (commands and skills)
openspec config set profile custom
openspec config set delivery both

# Enable all required workflows.
# Requires jq to be installed, otherwise enable all workflows by running this interactive mode: `openspec config profile`
if command -v jq >/dev/null 2>&1; then
  OPENSPEC_CONFIG_PATH="$(openspec config path)"
  OPENSPEC_TMP_FILE="$(mktemp)"
  jq '.workflows = ["propose", "explore", "new", "continue", "apply", "ff", "sync", "archive", "bulk-archive", "verify"]' "$OPENSPEC_CONFIG_PATH" > "$OPENSPEC_TMP_FILE" && mv "$OPENSPEC_TMP_FILE" "$OPENSPEC_CONFIG_PATH"
  printf "\nSuccess\!\n\nOpenSpec is correctly configured for SuperSpec and has the following workflows enabled: \n%s\n\n" "$(openspec config get workflows)"
else
  printf "\nWarning: jq was not found on your system.\n\nRun the following command manually:\nopenspec config profile\n\n"
  printf "During interactive process:\n  Select: Change \"Workflows Only\"\n  Enable all workflows\n\n"
  printf "Once completed, run this command to confirm all workflows are available:\nopenspec config get workflows\n"
fi

```

#### OpenSpec Repository Setup

Execute steps below from the root of your local git repository.

##### 1. Initialize OpenSpec for your harness

The example below initializes for **Cursor**. For other harnesses (Claude Code, Copilot, Codex, Gemini), run `openspec init --help` to see the supported `--tools` values and follow the guided prompts.

```bash
openspec init --tools cursor --profile custom
```

##### 2. Copy the SuperSpec schema into `openspec/schemas`

```bash
git clone --depth 1 --filter=blob:none --sparse \
  https://github.com/danielhanold/superspec.git /tmp/superspec-tmp && \
  cd /tmp/superspec-tmp && \
  git sparse-checkout set openspec/schemas/superspec && \
  cd - && \
  mkdir -p openspec/schemas/superspec && \
  cp -r /tmp/superspec-tmp/openspec/schemas/superspec/. openspec/schemas/superspec/ && \
  rm -rf /tmp/superspec-tmp
```

##### 3. Set SuperSpec as the default schema

```bash
echo "schema: superspec" > openspec/config.yaml
```

##### 4. Verify the install

```bash
openspec schemas      # superspec should appear in the list
openspec validate     # should pass with no errors
```

---

## Quick start

These commands are run **inside your agent harness** (e.g., Cursor's agent mode, Claude Code, Copilot, Codex, or Gemini) — not in a plain shell. The `/opsx:` slash commands are registered by your harness's OpenSpec integration, so type them directly into the agent prompt.

OpenSpec installs both slash commands as well as agent skills for your harness. Agent harness often auto-complete
slash commands and skills. Use the slash commands (starting with `/opsx:...`) instead of the skills.

Example: Archiving a change.
```
/opsx-archive             ==> slash commands (use this)
/openspec-archive-change  ==> skill name (do not use this - slash command will use this implicitly)
```

Once installed, you have two flows.

### Step-by-step flow (recommended)

Stop at each artifact, review it, give feedback, and only continue when you're satisfied. This is the default flow for any non-trivial change — human-in-the-loop at every checkpoint.

```bash
/opsx:new my-feature   # → starts a new change (use change name or description of what you want to build)
/opsx:continue         # → brainstorm (interactive conversation)
/opsx:continue         # → proposal
/opsx:continue         # → design (optional, only when technical decisions need explanation)
/opsx:continue         # → specs (creates delta specs: ADDED / MODIFIED / REMOVED / RENAMED)
/opsx:continue         # → tasks
/opsx:continue         # → plan
/opsx:apply            # Worktree + Superpowers TDD loop — produces the actual code and writes apply.md (the v2 receipt)
/opsx:verify           # Validate implementation matches the delta specs and tasks (requires apply.md to exist)
/opsx:continue         # → finalize (invokes superpowers:finishing-a-development-branch, writes finalize.md; v3)
/opsx:archive          # Sync the change's delta specs into project specs, then archive
```

### Quick flow (fast-forward)

For small, well-understood changes where you trust the agent to produce every artifact without per-step review. `/opsx:ff` runs the full artifact-creation pipeline end-to-end with no checkpoints.

```bash
/opsx:ff my-feature    # End-to-end: brainstorm + proposal + design + specs + tasks + plan
/opsx:apply            # Worktree + Superpowers TDD loop — produces the actual code and writes apply.md (the v2 receipt)
/opsx:verify           # Validate implementation matches the delta specs and tasks (requires apply.md to exist)
/opsx:continue         # → finalize (invokes superpowers:finishing-a-development-branch, writes finalize.md; v3)
/opsx:archive          # Sync the change's delta specs into project specs, then archive
```

The `/opsx:` slash commands ship with your harness's OpenSpec integration, not with this schema. If your harness uses different command names, check its OpenSpec docs.

To skip Superspec for a single change and use the upstream schema instead:

```bash
/opsx:new my-simple-fix --schema spec-driven
```

---

## Further reading

- [`docs/workflow.md`](docs/workflow.md) — visual overview and quick mental model for the six-phase workflow
- [`docs/workflow-details.md`](docs/workflow-details.md) — full ten-step walkthrough with per-step rationale, owner, and fallback notes
- [`docs/project-layout.md`](docs/project-layout.md) — files and directories Superspec adds under `openspec/` after install
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
