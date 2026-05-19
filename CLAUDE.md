# CLAUDE.md

## Repository Overview

This is a collection of Microsoft Fabric automation tools — reusable Jupyter notebooks and toolkit modules for administering Power BI / Microsoft Fabric workspaces. Everything runs inside Microsoft Fabric Spark notebooks.

## Repository Structure

```
microsoft_fabric_mods/
├── code_snippets/
│   ├── ADM/
│   │   └── Schedule_Scanner.ipynb       # Scans and catalogs scheduled items across workspaces
│   └── PBI/
│       └── RLS_Synchronizer.ipynb       # Synchronizes Row-Level Security between semantic models
└── data_storage_observability_toolkit/
    └── OBS_01_StorageProfile_Scanning/
        └── StorageProfile_Scanning.txt  # Stub / placeholder for storage profile scanning
```

### Directory Conventions

- `code_snippets/ADM/` — Administration utilities (scheduling, monitoring, workspace ops)
- `code_snippets/PBI/` — Power BI / semantic model utilities (RLS, dataset management)
- `data_storage_observability_toolkit/` — AI-ready data observability modules prefixed `OBS_XX_`

## Notebook Structure Convention

Every notebook follows this cell-section order:

1. **Markdown header** — title, what it does, limitations, notes, version, last updated
2. **PARAMETERS** — user-facing knobs at the top; always the first code cell
3. **IMPORTS** / **Libraries** — `%pip install` then imports
4. **CONFIGURATION** — dataclasses or lookup tables derived from parameters
5. **FUNCTIONS** — all helper logic, grouped under markdown sub-headers (Read Operations, Audit & Compare, Modify Operations, Backup Operations, Main function)
6. **MAIN** — single call to the top-level orchestrator function
7. **Output** / **Evaluation** / **Export** — display results or write to Delta

## Key Libraries

| Library | Purpose |
|---|---|
| `sempy.fabric` | Fabric REST client, workspace/dataset resolution |
| `sempy_labs` | Semantic model BIM export/import, report cloning, TOM access |
| `notebookutils` (`nbutl`) | Notebook exit, runtime context detection |
| `polars` | Lazy DataFrame operations; preferred for large scan outputs |
| `pandas` | Used in RLS utilities; returned from TOM-based functions |
| `pytz` | Timezone-aware timestamps on backup names and audit columns |

## Code Conventions

### Parameters block
- Always at the top, clearly separated with a `# PARAMETERS` markdown cell.
- Provide sensible placeholder strings (`"TARGET_MODEL_NAME"`) so the notebook is self-documenting.
- Include a `DRY_RUN = True` safety switch for any notebook that writes data.

### Dry-run pattern
All functions that modify Fabric resources accept `dry_run: bool = True`. When `dry_run=True`:
- Print what *would* happen without executing anything.
- Return the same type as the live path so callers can inspect planned changes.
- Cap preview output at 10 items and print a count for the remainder.

### Function signatures
- Positional: `model`, `workspace` first, then action-specific params, then `dry_run` last.
- Return types are always annotated.
- Docstrings follow Google style: Args, Returns, Raises (when applicable), Example.

### Read vs. write separation
- TOM connections for read operations use `readonly=True`.
- Write connections use `readonly=False` and are only opened inside live (non-dry-run) branches.

### Polars LazyFrame pattern (scanner notebooks)
- Build the full pipeline as a `LazyFrame` before calling `.collect()`.
- Collect exactly once at the end (or in the Main section).
- Use `.filter()`, `.with_columns()`, `.select()`, `.sort()` — in that order — for clarity.

### Naming
- Functions and variables: `snake_case`
- Dataclasses and class names: `PascalCase`
- Constants / config lookups: `UPPER_SNAKE_CASE`
- Backup items: `{prefix}_{model_name}_{YYYYMMDDTHHmmSS}` timestamp format

### Step-numbered orchestrators
Multi-step workflow functions (like `sync_rls`) print numbered steps separated by `"="*70` dividers so notebook output is easy to scan.

### Exit behavior
Notebooks detect interactive vs. pipeline runs via `nbutl.runtime.context["isForInteractive"]`:
- Interactive: `print(result)`
- Pipeline / scheduled: `nbutl.notebook.exit(result)`

## Development Workflow

### Adding a new snippet
1. Create a notebook under the appropriate subfolder (`ADM/`, `PBI/`, or a new category).
2. Follow the cell-section order above.
3. Start with `DRY_RUN = True` and validate output before enabling writes.
4. Pin the `semantic-link-labs` version in the `%pip install` cell.

### Adding a new observability module
- Name it `OBS_XX_<DescriptiveName>/` inside `data_storage_observability_toolkit/`.
- Include a `README.txt` in the folder describing what the module scans.

### Testing
There is no automated test suite. Validation is done by running notebooks in dry-run mode against real Fabric workspaces and inspecting printed output before setting `DRY_RUN = False`.

## Important Constraints

- Mixed/composite storage models (Import + DirectQuery in the same model) are **not supported** by the RLS Synchronizer.
- `Schedule_Scanner` only captures *enabled* schedules by default (filtered with `.filter(pl.col('ScheduleEnabled') == True)`); remove that filter to include disabled schedules.
- Supported schedulable item types: `DataPipeline`, `CopyJob`, `Notebook`, `SparkJobDefinition`, `Dataflow` (Gen2). Any new Fabric item type must be added to `ITEM_KEYS` in `Schedule_Scanner`.
- `backup_model` clones both the semantic model and its default report for Import/DirectQuery storage modes; DirectLake models get a model-only backup.
