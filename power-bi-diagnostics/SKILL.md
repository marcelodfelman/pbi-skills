---
name: Power BI Diagnostics
description: Troubleshoot Power BI model performance, trace query execution, manage caches, and verify the pbi-cli environment using pbi-cli. Invoke this skill whenever the user says "pbi not working", "setup issues", "connection failed", "slow query", "performance", "profiling", "tracing", "health check", "model audit", "pbi setup", or encounters any pbi-cli error. This is the first skill to check when something goes wrong with pbi-cli.
tools: pbi-cli
---

# Power BI Diagnostics Skill

Troubleshoot performance, trace queries, and verify the pbi-cli environment.

## Prerequisites

```bash
pipx install pbi-cli-tool
pbi-cli skills install
pbi connect
```

## Environment Check

```bash
# Verify pythonnet and .NET DLLs are installed
pbi setup

# Show detailed environment info (version, DLL paths, pythonnet status)
pbi setup --info
pbi --json setup --info

# Check CLI version
pbi --version
```

## Quick Troubleshooting

If pbi-cli isn't working, run these checks in order:

```bash
# 1. Is pbi-cli installed correctly?
pbi --version
pbi setup --info

# 2. Is Power BI Desktop running with a model open?
pbi connect

# 3. Is the connection still alive?
pbi connections last

# 4. Can you query the model?
pbi dax execute "EVALUATE ROW(\"test\", 1)"
```

## Model Health Check

```bash
# Quick model overview
pbi --json model get

# Object counts (tables, columns, measures, relationships, partitions)
pbi --json model stats

# List all tables with column/measure counts
pbi --json table list
```

## Query Tracing

Capture diagnostic events during DAX query execution:

```bash
# Start a trace
pbi trace start

# Execute the query you want to profile
pbi dax execute "EVALUATE SUMMARIZECOLUMNS(Products[Category], \"Total\", SUM(Sales[Amount]))"

# Stop the trace
pbi trace stop

# Fetch captured trace events
pbi --json trace fetch

# Export trace events to a file
pbi trace export ./trace-output.json
```

## Cache Management

```bash
# Clear the formula engine cache (do this before benchmarking)
pbi dax clear-cache
```

## Connection Diagnostics

```bash
# List all saved connections
pbi connections list
pbi --json connections list

# Show the last-used connection
pbi connections last

# Reconnect to a specific data source
pbi connect -d localhost:54321

# Disconnect
pbi disconnect
```

## Workflow: Profile a Slow Query

```bash
# 1. Clear cache for a clean benchmark
pbi dax clear-cache

# 2. Start tracing
pbi trace start

# 3. Run the slow query
pbi dax execute "EVALUATE SUMMARIZECOLUMNS(Products[Category], \"Total\", SUM(Sales[Amount]))" --timeout 300

# 4. Stop tracing
pbi trace stop

# 5. Export trace for analysis
pbi trace export ./slow-query-trace.json

# 6. Review trace events
pbi --json trace fetch
```

## Workflow: Model Health Audit

```bash
# 1. Model overview
pbi --json model get
pbi --json model stats

# 2. Check table sizes and structure
pbi --json table list

# 3. Review relationships
pbi --json relationship list

# 4. Check security roles
pbi --json security-role list

# 5. Export full model for offline review
pbi database export-tmdl ./audit-export/
```

## Best Practices

- Clear cache before benchmarking: `pbi dax clear-cache`
- Use `--timeout` for long-running queries to avoid premature cancellation
- Export traces to files for sharing with teammates
- Run `pbi setup --info` first when troubleshooting environment issues
- Use `--json` output for automated monitoring scripts
- Use `pbi repl` for interactive debugging sessions with persistent connection

## Known Error Messages — Root Cause & Fix

### `The '<X>' measure cannot be created because a column with the same name already exists`

Thrown at import time against a live Fabric / AS server (offline build succeeds silently). Column and measure share a namespace within a table — cannot coexist with the same name.

```
# Diagnose
pbi column list --table FactProduction | grep -i OEE
pbi measure list --table FactProduction | grep -i OEE

# Fix: rename the column (keep the clean measure name)
pbi column rename OEE OEERow --table FactProduction
pbi measure update OEE -t FactProduction -e "AVERAGE(FactProduction[OEERow])"
pbi database export-tmdl ./Model.SemanticModel/definition/
```

See **power-bi-modeling** skill → "Column and measure names share a namespace".

### `DAX query operations are not supported on offline connections`

You are connected to a TMDL folder (`pbi connect ./Model.SemanticModel`) which has no running engine. Measures can be created but not executed.

```bash
pbi connections last         # confirm it says isOffline: true
# Fix: open the .pbip in Power BI Desktop, then reconnect live
pbi disconnect
pbi connect                  # auto-detects the Desktop port
pbi dax execute "..."        # now works
```

See **power-bi-dax** skill → "DAX execution is blocked on offline connections".

### `Column 'Date' not found in table 'DimDate'`

Typically appears right after creating a calculated table with `DaxExpression` offline, then calling `table mark-date`. The table has 0 columns because the DAX wasn't executed. Switch to M partition, verify the schema, then mark.

```bash
pbi table get DimDate        # inspect what columns actually exist
# → If empty, recreate using mExpression with explicit columns
pbi table mark-date DimDate --date-column Date
```

See **power-bi-deployment** skill → "Calculated tables materialise empty offline".

### `IsKey is only supported for DirectQuery tables`

You set `IsKey: true` on a column in an Import-mode table. Remove the flag — engine manages row identity automatically via `RowNumber`.

### `"ambiguous paths: TableA→TableB and TableA→TableB"`

Same relationship defined in both `model.tmdl` and `relationships.tmdl`. Only `relationships.tmdl` should contain `relationship` blocks. See **power-bi-deployment** skill.

### Measures return blank / error after column rename

`pbi column rename` does not propagate to measure DAX expressions. After any column rename, audit affected measures:

```bash
# Find measures still referencing the old column name
pbi --json measure list | jq -r '.[] | select(.expression | contains("OldColumnName")) | "\(.tableName).\(.name)"'

# Update each
pbi measure update "Net Sales" -t FactSales -e "<new expression>"
```

### Power BI Desktop rejects `.pbip` with: `Property 'layoutOptimization' has not been defined and the schema does not allow additional properties`

pbi-cli 3.10.5 `pbi report validate` reports `report.json` missing required `'layoutOptimization'` and may lead you to add it. Power BI Desktop (March 2026 build, 2.152.1279.0) uses the shipped report schema 3.2.0 which **does not** define this property, so it rejects the file at open time.

**Fix:** remove `"layoutOptimization"` from `report.json`. Desktop is the authoritative schema — ignore the `pbi report validate` warning for this specific key.

```bash
# Open report.json, delete the line `"layoutOptimization": "None"`
# pbi report validate will still complain — that is the known benign warning
pbi report info   # sanity check pages still list
# Double-click Retail.pbip — should open without the schema error
```

See **power-bi-report** skill → "layoutOptimization — validator vs Desktop schema conflict".

### Power BI Desktop rejects `.pbip` with: `Property 'drillthrough' has not been defined` in `page.json`

Adding `"drillthrough": { "type": "OnDrillthrough" }` to a `page.json` file causes Desktop to reject the entire report. The page schema `page/2.1.0/schema.json` does not define a `drillthrough` property.

**Fix:** remove the `"drillthrough"` key from the affected `page.json` file(s). Also delete any associated hidden drill-through filter visuals (`isDrillthrough: true`) you may have created.

```bash
# Find affected pages
grep -rl '"drillthrough"' Retail.Report/definition/pages/*/page.json

# Fix with Python
python -c "import json
from pathlib import Path
for pj in Path('Retail.Report/definition/pages').glob('*/page.json'):
    d = json.loads(pj.read_text(encoding='utf-8'))
    if 'drillthrough' in d:
        del d['drillthrough']
        pj.write_text(json.dumps(d, indent=2), encoding='utf-8')
        print(f'Fixed {pj}')"
```

**Drill-through is a Desktop UI-only operation.** Configure it from Format pane → Page → Drillthrough in Power BI Desktop. See **power-bi-pages** skill for details.

### Power BI Desktop rejects `.pbip` with: `Property 'Commands' has not been defined` in `visual.query.Commands`

`pbi visual bind` (and `bulk-bind`) always write a `Commands` array alongside `queryState.projections` in every `visual.json`. Power BI Desktop March 2026 rejects this — the visual 2.7.0 schema does not allow `Commands` under `visual.query`.

**Fix: strip `Commands` from every `visual.json` in the report as a mandatory post-bind step.** Runs in under a second for hundreds of visuals:

```bash
python -c "import json
from pathlib import Path
count = 0
for v in Path('./Retail.Report/definition/pages').rglob('visual.json'):
    d = json.loads(v.read_text(encoding='utf-8'))
    q = d.get('visual', {}).get('query', {})
    if 'Commands' in q:
        del q['Commands']
        v.write_text(json.dumps(d, indent=2), encoding='utf-8')
        count += 1
print(f'Cleaned {count} files')"
```

Treat this as part of the bind workflow: `bind` → `strip Commands` → `open in Desktop`. The fix is idempotent; re-running it on already-cleaned files is a no-op.

See **power-bi-visuals** skill → "Do NOT include `Commands` inside `visual.query`".

## BLANK KPI cards — diagnosis checklist

When a KPI card shows BLANK:

1. **YTD/MTD/QTD measure + no current-year data** → Most common. Replace TOTALYTD/SAMEPERIODLASTYEAR with DATESBETWEEN anchored to MAX(FactSales[SalesDate]).
2. **Ratio/% with no denominator** → DIVIDE returns BLANK if denominator = 0. Add `DIVIDE(num, denom, 0)` for counts; leave BLANK for ratios (acceptable).
3. **Year slicer set to year with no data** → Check if synthetic data covers the selected year.
4. **Measure references renamed column** → After any column rename, update all DAX that references it manually.

## Layout overlap — KPI cards and charts

**Symptom:** Charts appear on top of KPI cards.
**Cause:** Charts row1 set to y=155; KPI cards extend to y=168 (y=84+h=84).
**Fix:** Set charts row1 to y=175 (minimum). Standard layout:
- Cards: y=84, h=84, bottom=168
- Charts row1: y=175, h=258
- Charts row2: y=438, h=272
