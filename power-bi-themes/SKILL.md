---
name: Power BI Themes
description: >
  Apply, inspect, and compare Power BI report themes and conditional formatting
  rules using pbi-cli. Invoke this skill whenever the user mentions "theme",
  "colours", "colors", "branding", "dark mode", "corporate theme", "styling",
  "conditional formatting", "colour scale", "gradient", "data bars",
  "background colour", "formatting rules", "visual formatting", or wants to
  change the overall look-and-feel of a report or apply data-driven formatting
  to specific visuals.
tools: pbi-cli
---

# Power BI Themes Skill

Manage report-wide themes and per-visual conditional formatting. No Power BI
Desktop connection is needed.

## Applying a Theme

Power BI themes are JSON files that define colours, fonts, and visual defaults
for the entire report. Apply one with:

```bash
pbi report set-theme --file corporate-theme.json
```

This copies the theme file into the report's `StaticResources/RegisteredResources/`
folder and updates `report.json` to reference it. The theme takes effect when
the report is opened in Power BI Desktop.

## Inspecting the Current Theme

```bash
pbi report get-theme
```

Returns:
- `base_theme` -- the built-in theme name (e.g. `"CY24SU06"`)
- `custom_theme` -- custom theme name if one is applied (or `null`)
- `theme_data` -- full JSON of the custom theme file (if it exists)

## Comparing Themes

Before applying a new theme, preview what would change:

```bash
pbi report diff-theme --file proposed-theme.json
```

Returns:
- `current` / `proposed` -- display names
- `added` -- keys in proposed but not current
- `removed` -- keys in current but not proposed
- `changed` -- keys present in both but with different values

This helps catch unintended colour changes before committing.

## Theme JSON Structure

A Power BI theme JSON file typically contains:

```json
{
  "name": "Corporate Brand",
  "dataColors": ["#0078D4", "#00BCF2", "#FFB900", "#D83B01", "#8661C5", "#00B294"],
  "background": "#FFFFFF",
  "foreground": "#252423",
  "tableAccent": "#0078D4",
  "visualStyles": { ... }
}
```

Key sections:
- `dataColors` -- palette for data series (6-12 colours recommended)
- `background` / `foreground` -- page and text defaults
- `tableAccent` -- header colour for tables and matrices
- `visualStyles` -- per-visual-type overrides (font sizes, padding, etc.)

See [Microsoft theme documentation](https://learn.microsoft.com/power-bi/create-reports/desktop-report-themes) for the full schema.

## ⚠️ Critical: Custom Theme Registration in PBIR

### 1 — The theme file must have NO file extension on disk

Power BI Service resolves `RegisteredResources` paths **without appending `.json`**. If your file is `deeply-theme.json` but the path in `report.json` references `"deeply-theme"`, the service throws:

> `"The following file was not found: 'StaticResources/RegisteredResources/deeply-theme'"`

**Convention**: keep **two copies** in `StaticResources/RegisteredResources/`:
- `deeply-theme` (no extension) — the file PBI Service actually loads
- `deeply-theme.json` — editable convenience copy

Always sync them after editing:
```bash
copy deeply-theme.json deeply-theme    # Windows
cp deeply-theme.json deeply-theme      # bash/WSL
```

In `report.json` the path must reference the no-extension filename:
```json
{
  "name": "deeply-theme",
  "path": "deeply-theme",
  "type": "CustomTheme"
}
```

### 2 — `customTheme` requires `reportVersionAtImport`

Both `baseTheme` and `customTheme` need the same three-field `reportVersionAtImport` object or publish fails with:

> `"Required properties are missing from object: reportVersionAtImport. Path 'themeCollection.customTheme'"`

```json
"themeCollection": {
  "baseTheme": {
    "name": "CY26SU02",
    "reportVersionAtImport": { "visual": "2.6.0", "report": "3.1.0", "page": "2.3.0" },
    "type": "SharedResources"
  },
  "customTheme": {
    "name": "deeply-theme",
    "reportVersionAtImport": { "visual": "2.6.0", "report": "3.1.0", "page": "2.3.0" },
    "type": "RegisteredResources"
  }
}
```

### 3 — Keep custom themes minimal (silent validation failures)

Unsupported `visualStyles` properties cause PBI to **silently fall back to the base theme** with no error message. Symptom: a single style override applies (e.g. header text colour from `visual.json`) but the rest of the theme is ignored (page stays white).

**Properties known to cause silent failures:**
- `textClasses` with deep overrides
- `lineChart.lineStyles`
- Granular `pivotTable` sub-property overrides
- Using `"*"` as a wildcard property name in nested `visualStyles`

**Safe starting point** — these properties reliably work:
```json
{
  "name": "my-theme",
  "dataColors": ["#4DD8E6", ...],
  "background": "#0F1B2D",
  "foreground": "#FFFFFF",
  "tableAccent": "#4DD8E6",
  "visualStyles": {
    "page": { "*": { "background": [{ "color": { "solid": { "color": "#0F1B2D" } }, "show": true }] } },
    "card": { "*": { "background": [{ "color": { "solid": { "color": "#17253D" } }, "show": true }] } }
  }
}
```

Grow from this baseline and test one section at a time.

## Conditional Formatting

Apply data-driven formatting to individual visuals:

```bash
# Gradient background (colour scale from min to max)
pbi format background-gradient visual_abc --page page1 \
    --table Sales --column Revenue \
    --min-color "#FFFFFF" --max-color "#0078D4"

# Rules-based background (specific value triggers a colour)
pbi format background-conditional visual_abc --page page1 \
    --table Sales --column Status --value "Critical" --color "#FF0000"

# Measure-driven background (a DAX measure returns the colour)
pbi format background-measure visual_abc --page page1 \
    --table Sales --measure "Status Color"

# Inspect current formatting rules
pbi format get visual_abc --page page1

# Clear all formatting rules on a visual
pbi format clear visual_abc --page page1
```

## Workflow: Brand a Report

```bash
# 1. Create the theme file
cat > brand-theme.json << 'EOF'
{
  "name": "Acme Corp",
  "dataColors": ["#1B365D", "#5B8DB8", "#E87722", "#00A3E0", "#6D2077", "#43B02A"],
  "background": "#F8F8F8",
  "foreground": "#1B365D",
  "tableAccent": "#1B365D"
}
EOF

# 2. Preview the diff against the current theme
pbi report diff-theme --file brand-theme.json

# 3. Apply it
pbi report set-theme --file brand-theme.json

# 4. Verify
pbi report get-theme
```

## JSON Output

```bash
pbi --json report get-theme
pbi --json report diff-theme --file proposed.json
pbi --json format get vis1 --page p1
```

## Brand Images (logos, icons) in `RegisteredResources`

Brand images live alongside themes in `StaticResources/RegisteredResources/`. The rules differ slightly from custom themes:

### Desktop renames images on import

When you add an image through Power BI Desktop UI (Insert → Image → upload), Desktop **hashes the filename** to avoid collisions across visuals that reference similarly-named files:

```
original:  Sugat.svg.png
on disk:   Sugat.svg2611373602373278.png   ← numeric hash suffix
```

The `report.json` resource entry uses the hashed name; the image visual's `sourceFile.image.name` keeps the ORIGINAL filename for display. Do NOT try to "clean up" the hashed name — Desktop regenerates it on every new import.

### Registering an image in `report.json`

```json
"resourcePackages": [{
  "name": "RegisteredResources",
  "type": "RegisteredResources",
  "items": [
    { "name": "sugat-theme", "path": "sugat-theme", "type": "CustomTheme" },
    { "name": "Sugat.svg2611373602373278.png",
      "path": "Sugat.svg2611373602373278.png",
      "type": "Image" }
  ]
}]
```

**Unlike themes**, image `path` INCLUDES the `.png` extension and the file on disk DOES carry the extension. This is the reverse of custom themes (which must be extension-less). The schema rule: `path` equals the on-disk filename verbatim for images.

### File placement

```
Retail.Report/StaticResources/RegisteredResources/
  sugat-theme                          ← theme, no extension (see above)
  sugat-theme.json                     ← editable copy (optional)
  Sugat.svg2611373602373278.png        ← image, WITH .png extension
```

### Referencing the image from a visual

See **power-bi-visuals** skill → "Image visuals". The visual's `url.ResourcePackageItem.ItemName` must match the `name` field in `resourcePackages.items` exactly (including the hash).

### Replicating a brand image to all pages

Write the image visual once on one page (inserting via Desktop is easiest — it handles the schema correctly), then copytree the visual folder to every other page:

```bash
python -c "import shutil
from pathlib import Path
src = Path('Retail.Report/definition/pages/exec_overview/visuals/sugat_logo')
for p in Path('Retail.Report/definition/pages').iterdir():
    if p.is_dir() and p.name != 'exec_overview':
        dst = p / 'visuals' / 'sugat_logo'
        if dst.exists(): shutil.rmtree(dst)
        shutil.copytree(src, dst)"
```

All pages then reference the same `resourcePackages` entry — one physical image file on disk.
