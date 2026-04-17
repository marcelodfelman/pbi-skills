---
name: Power BI Deployment
description: Import and export TMDL/TMSL formats, manage model lifecycle with transactions, and version-control Power BI semantic models using pbi-cli. Invoke this skill whenever the user mentions "deploy", "export", "import", "TMDL", "TMSL", "version control", "git", "backup", "migrate", "transaction", "commit changes", "rollback", or wants to save/restore model state.
tools: pbi-cli
---

# Power BI Deployment Skill

Manage model lifecycle with TMDL export/import, transactions, and version control.

## Prerequisites

```bash
pipx install pbi-cli-tool
pbi-cli skills install
pbi connect
```

## Connecting to Targets

```bash
# Local Power BI Desktop (auto-detects port)
pbi connect

# Local with explicit port
pbi connect -d localhost:54321

# Named connections for switching
pbi connect -d localhost:54321 --name dev
pbi connections list
pbi connections last
pbi disconnect
```

## TMDL Export and Import

TMDL (Tabular Model Definition Language) is the text-based format for version-controlling Power BI models.

```bash
# Export entire model to TMDL folder
pbi database export-tmdl ./model-tmdl/

# Import TMDL folder into connected model
pbi database import-tmdl ./model-tmdl/
```

## TMSL Export

```bash
# Export as TMSL JSON (for SSAS/AAS compatibility)
pbi database export-tmsl
```

## TMDL Diff (Compare Snapshots)

Compare two TMDL export folders to see what changed between snapshots.
Useful for CI/CD pipelines ("what did this PR change in the model?").

```bash
# Compare two exports
pbi database diff-tmdl ./model-before/ ./model-after/

# JSON output for CI/CD scripting
pbi --json database diff-tmdl ./baseline/ ./current/
```

Returns a structured summary:
- **tables**: added, removed, and changed tables with per-table entity diffs
  (measures, columns, partitions, hierarchies added/removed/changed)
- **relationships**: added, removed, and changed relationships
- **model**: changed model-level properties (e.g. culture, default power bi dataset version)
- **summary**: total counts of all changes

LineageTag-only changes (GUID regeneration without real edits) are automatically
filtered out to avoid false positives.

No connection to Power BI Desktop is needed -- works on exported folders.

## Database Operations

```bash
# List databases on the connected server
pbi database list
```

## Transaction Management

Use transactions for atomic multi-step changes:

```bash
# Begin a transaction
pbi transaction begin

# Make changes
pbi measure create "New KPI" -e "SUM(Sales[Amount])" -t Sales
pbi measure create "Another KPI" -e "COUNT(Sales[OrderID])" -t Sales

# Commit all changes atomically
pbi transaction commit

# Or rollback if something went wrong
pbi transaction rollback
```

## Table Refresh

```bash
# Refresh individual tables
pbi table refresh Sales --type Full
pbi table refresh Sales --type Automatic
pbi table refresh Sales --type Calculate
pbi table refresh Sales --type DataOnly
```

## Workflow: Version Control with Git

```bash
# 1. Export model to TMDL
pbi database export-tmdl ./model/

# 2. Commit to git
cd model/
git add .
git commit -m "feat: add new revenue measures"

# 3. Later, import back into Power BI Desktop
pbi connect
pbi database import-tmdl ./model/
```

## Workflow: Inspect Model Before Deploy

```bash
# Get model metadata
pbi --json model get

# Check model statistics
pbi --json model stats

# List all objects
pbi --json table list
pbi --json measure list
pbi --json relationship list
```

## Best Practices

- Always export TMDL before making changes (backup)
- Use transactions for multi-object changes
- Test changes in dev before deploying to production
- Use `--json` for scripted deployments
- Store TMDL in git for version history
- Use named connections (`--name`) to avoid accidental changes to wrong environment

## ⚠️ Critical: TMDL Relationship File Rules

### Relationships belong ONLY in `relationships.tmdl`

PBIP projects with a `definition/relationships.tmdl` file store **all** relationship definitions there. Never add relationship blocks to `model.tmdl` as well — the PBI engine reads both files and treats the entries as duplicates, throwing:

> `"ambiguous paths: TableA→TableB and TableA→TableB"` (same path listed twice)

**`model.tmdl`** should contain only `ref table`, `ref cultureInfo`, and model-level settings — no `relationship` blocks.

To check for accidental duplicates:
```bash
grep -r "^relationship" Report.SemanticModel/definition/
```

All matches should be in `relationships.tmdl` only.

### Data refresh required after publishing new relationships

Adding or changing relationships defines them structurally but does **not** compute them. After publishing:

1. Open the `.pbip` in Power BI Desktop → **Home → Refresh**
2. Or trigger a dataset refresh in Power BI Service

Symptom when skipped:
> `"Error fetching data for this visual — relationship … does not hold any data because it needs to be recalculated"`

Visuals also show `(Blank)` for all measures that cross the new relationship boundary.

## ⚠️ Offline / TMDL-folder mode constraints

When connecting to a TMDL folder instead of a live Power BI Desktop / AAS / Fabric instance (`pbi connect <folder>` or `connection_operations.ConnectFolder`), several operations behave differently because there is no running engine.

### In-memory changes are NOT persisted until export

Every `table create`, `measure create`, `relationship create`, `column rename`, etc., lives **only in memory** for the duration of the offline session. The `.tmdl` files on disk are not touched automatically. The session returning `success: true` does NOT mean the file was written. Running again in a new session will lose the work.

**Always finish an offline session with an explicit export:**
```bash
pbi database export-tmdl ./Retail.SemanticModel/definition/
```

After export, verify the files changed:
```bash
ls -la ./Retail.SemanticModel/definition/tables/
git status    # if version-controlled — should show modified .tmdl files
```

### Calculated tables (DaxExpression) materialise empty offline

When creating a table with `DaxExpression` (a calculated table like `ADDCOLUMNS(CALENDAR(...), ...)`), the offline TMDL engine stores the DAX string but cannot execute it — there is no VertiPaq engine to run `CALENDAR` or `ADDCOLUMNS`. The table is created with **0 columns**. Subsequent operations like `MarkAsDateTable` then fail because the `Date` column does not yet exist:

> `"Column 'Date' not found in table 'DimDate'"`

**Prefer M partitions for every table when building offline**, including DimDate. M expressions with `#table()`, `Table.FromRecords()`, or `List.Dates()` are parsed and the column schema is materialised without a running VertiPaq engine:

```m
let
    Dates = List.Dates(#date(2023,1,1), 1461, #duration(1,0,0,0)),
    ToTable = Table.FromList(Dates, Splitter.SplitByNothing(), {"Date"}),
    Typed = Table.TransformColumnTypes(ToTable, {{"Date", type date}}),
    WithYear = Table.AddColumn(Typed, "Year", each Date.Year([Date]), Int64.Type)
in
    WithYear
```

Reserve `DaxExpression` tables for connections to a **live** instance that can actually execute DAX.

### DAX query execution is blocked offline

`pbi dax run` / `dax_query_operations.Execute`/`.Validate` return:

> `"DAX query operations are not supported on offline connections"`

To validate measures or run `EVALUATE`, either open the `.pbip` in Power BI Desktop (live engine) or connect to a Fabric workspace / local AS instance. Offline sessions are for structural changes only — no data-driven checks are possible. See **power-bi-dax** skill for the mitigation pattern.

### Deployment loop from TMDL folder

Typical flow when the `.pbip` must end up back in Power BI Desktop:

```bash
# Start — build structure offline (fast, no Desktop required)
pbi connect ./Retail.SemanticModel
pbi table create ...
pbi measure create ...
pbi database export-tmdl ./Retail.SemanticModel/definition/

# Finish — open in Desktop, refresh data, validate DAX
open Retail.pbip       # macOS — or double-click on Windows
# → Desktop reads the updated TMDL, runs M queries, populates data
# → now DAX queries and visual rendering work
```
