---
name: Power BI Report
description: >
  Scaffold, validate, preview, and manage Power BI PBIR report projects using
  pbi-cli. Invoke this skill whenever the user mentions "create report", "new
  report", "PBIR", "scaffold", "validate report", "report structure", "preview
  report", "report info", "reload Desktop", "convert report", ".pbip project",
  "report project", or wants to understand the PBIR folder format, set up a new
  report from scratch, or work with the report as a whole. For specific tasks,
  see also: power-bi-visuals (charts, binding), power-bi-pages (page management),
  power-bi-themes (themes, formatting), power-bi-filters (page/visual filters).
tools: pbi-cli
---

# Power BI Report Skill

Manage Power BI PBIR report projects at the top level -- scaffolding, validation,
preview, and Desktop integration. No connection to Power BI Desktop is needed
for most operations.

## PBIR Format

PBIR (Enhanced Report Format) stores reports as a folder of JSON files:

```
MyReport.Report/
  definition.pbir            # dataset reference
  definition/
    version.json             # PBIR version
    report.json              # report settings, theme
    pages/
      pages.json             # page order
      page_abc123/
        page.json            # page settings
        visuals/
          visual_def456/
            visual.json      # visual type, position, bindings
```

Each file has a public JSON schema from Microsoft for validation.
PBIR is GA as of January 2026 and the default format in Desktop since March 2026.

## Creating a Report

```bash
# Scaffold a new report project
pbi report create ./MyProject --name "Sales Report"

# With dataset reference
pbi report create ./MyProject --name "Sales" --dataset-path "../Sales.Dataset"
```

This creates the full folder structure with `definition.pbir`, `report.json`,
`version.json`, and an empty `pages/` directory.

## Report Info and Validation

```bash
# Show report metadata summary (pages, theme, dataset)
pbi report info
pbi report info --path ./MyReport.Report

# Validate report structure and JSON files
pbi report validate
```

Validation checks:
- Required files exist (`definition.pbir`, `report.json`, `version.json`)
- All JSON files parse without errors
- Schema URLs are present and consistent
- Page references in `pages.json` match actual page folders

## Preview

Start a live HTML preview of the report layout:

```bash
pbi report preview
```

Opens a browser showing all pages with visual placeholders, types, positions,
and data bindings. The preview auto-refreshes when files change.

Requires the `preview` optional dependency: `pip install pbi-cli-tool[preview]`

## Desktop Integration

```bash
# Trigger Power BI Desktop to reload the current report
pbi report reload
```

Power BI Desktop's Developer Mode auto-detects TMDL changes but not PBIR
changes. This command sends a keyboard shortcut to the Desktop window to
trigger a reload. Requires the `reload` optional dependency: `pip install pbi-cli-tool[reload]`

## Convert

```bash
# Convert a .Report folder into a distributable .pbip project
pbi report convert ./MyReport.Report --output ./distributable/
```

## Path Resolution

All report commands auto-detect the `.Report` folder:

1. Explicit: `pbi report --path ./MyReport.Report info`
2. Auto-detect: walks up from CWD looking for `*.Report/definition/`
3. From `.pbip`: finds sibling `.Report` folder from `.pbip` file

## Workflow: Build a Complete Report

This workflow uses commands from multiple skills:

```bash
# 1. Scaffold report (this skill)
pbi report create . --name "SalesDashboard" --dataset-path "../SalesModel.Dataset"

# 2. Add pages (power-bi-pages skill)
pbi report add-page --display-name "Overview" --name overview
pbi report add-page --display-name "Details" --name details

# 3. Add visuals (power-bi-visuals skill)
pbi visual add --page overview --type card --name revenue_card
pbi visual add --page overview --type bar --name sales_by_region

# 4. Bind data (power-bi-visuals skill)
pbi visual bind revenue_card --page overview --field "Sales[Total Revenue]"
pbi visual bind sales_by_region --page overview \
    --category "Geo[Region]" --value "Sales[Amount]"

# 5. Apply theme (power-bi-themes skill)
pbi report set-theme --file brand-colors.json

# 6. Validate (this skill)
pbi report validate
```

## Combining Model and Report

pbi-cli covers both the semantic model layer and the report layer:

```bash
# Model layer (requires pbi connect)
pbi connect
pbi measure create Sales "Total Revenue" "SUM(Sales[Amount])"

# Report layer (no connection needed)
pbi report create . --name "Sales"
pbi visual add --page overview --type card --name rev_card
pbi visual bind rev_card --page overview --field "Sales[Total Revenue]"
```

## Related Skills

| Skill | When to use |
|-------|-------------|
| **power-bi-visuals** | Add, bind, update, delete visuals |
| **power-bi-pages** | Add, remove, configure pages and bookmarks |
| **power-bi-themes** | Themes, conditional formatting |
| **power-bi-filters** | Page and visual filters |

## JSON Output

```bash
pbi --json report info
pbi --json report validate
```

## ⚠️ Critical: report.json Schema Rules (3.2.0)

### Do NOT include `layoutOptimization` at the report level

`layoutOptimization` is not a valid property in the 3.2.0 report schema. Adding it causes publish to fail with a schema validation error.

### `customTheme` requires `reportVersionAtImport`

Both `baseTheme` and `customTheme` must include `reportVersionAtImport`. Without it publish fails:

> `"Required properties are missing from object: reportVersionAtImport. Path 'themeCollection.customTheme'"`

Minimal correct `themeCollection`:
```json
"themeCollection": {
  "baseTheme": {
    "name": "CY26SU02",
    "reportVersionAtImport": { "visual": "2.6.0", "report": "3.1.0", "page": "2.3.0" },
    "type": "SharedResources"
  },
  "customTheme": {
    "name": "my-theme",
    "reportVersionAtImport": { "visual": "2.6.0", "report": "3.1.0", "page": "2.3.0" },
    "type": "RegisteredResources"
  }
}
```

And in `resourcePackages`, the `path` must match the filename without extension (see **power-bi-themes** for the full custom theme file rules).

### `resourcePackages` entry for custom themes

```json
{
  "name": "RegisteredResources",
  "type": "RegisteredResources",
  "items": [{
    "name": "my-theme",
    "path": "my-theme",
    "type": "CustomTheme"
  }]
}
```

The `path` value is resolved as `StaticResources/RegisteredResources/{path}` — **no `.json` appended**.

### `layoutOptimization` — validator vs Desktop schema conflict

`pbi report validate` in pbi-cli 3.10.5 flags `report.json` as invalid with:

> `"report.json missing required 'layoutOptimization'"`

Adding the property satisfies the CLI but makes **Power BI Desktop (March 2026, build 2.152.1279.0) refuse to open the file** with:

> `'report.json': Property 'layoutOptimization' has not been defined and the schema does not allow additional properties. Path 'layoutOptimization', line 50, position 23.`

The two tools validate against different schema versions. The CLI uses a newer schema; Desktop uses the shipped 3.2.0 which does not include the property.

**Resolution: omit `layoutOptimization` from `report.json`**. Desktop is the ground truth — if the file does not open, nothing else matters. The `pbi report validate` warning is safe to ignore for this key; all other validation errors are still actionable. Never add the property even when the CLI suggests it.

If you encounter the same conflict on a different key, the same rule applies: open the `.pbip` in Desktop; whatever Desktop accepts is the source of truth.
