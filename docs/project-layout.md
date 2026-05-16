# Project layout (after install)

After running through the [Installation](../README.md#installation) steps, your repository will contain the following Superspec-related files under `openspec/`:

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
            ├── apply.md
            └── verify.md
```

| Path | Purpose |
|---|---|
| `openspec/config.yaml` | Sets `schema: superspec` as the default for new changes. Override per-change with `/opsx:new <name> --schema <other>`. |
| `openspec/schemas/superspec/schema.yaml` | Machine-readable schema definition — the artifact pipeline, ordering, and per-artifact `instruction` fields that hand off to Superpowers skills. |
| `openspec/schemas/superspec/README.md` | Design motivation and rationale for this schema. |
| `openspec/schemas/superspec/INTEGRATION.md` | Full lifecycle, CLI cheat sheet, and design-choice rationale. |
| `openspec/schemas/superspec/templates/` | Markdown stubs copied into a new change directory when the matching artifact is generated. |

Once you start working on changes, OpenSpec adds `openspec/changes/<change-name>/` directories at the same level as `schemas/`, holding the per-change `proposal.md`, `design.md`, `specs/`, `tasks.md`, `plan.md`, `apply.md`, and `verify.md` artifacts. After archival, completed changes move into `openspec/changes/archive/`.

## See also

- [README → Installation](../README.md#installation) — how the layout above gets created
- [docs/workflow.md](workflow.md) — visual overview of the workflow
- [docs/workflow-details.md](workflow-details.md) — what happens inside `openspec/changes/<name>/` for each phase
- [`openspec/schemas/superspec/INTEGRATION.md`](../openspec/schemas/superspec/INTEGRATION.md) — schema-internal layout and CLI commands
