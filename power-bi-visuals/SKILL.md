---
name: Power BI Visuals
description: >
  Add, configure, bind data to, and bulk-manage visuals on Power BI PBIR report
  pages using pbi-cli. Invoke this skill whenever the user mentions "add a chart",
  "bar chart", "line chart", "card", "KPI", "gauge", "scatter", "table visual",
  "matrix", "slicer", "combo chart", "bind data", "visual type", "visual layout",
  "resize visuals", "bulk update visuals", "bulk delete", "visual calculations",
  or wants to place, move, bind, or remove any visual on a report page. Also invoke
  when the user asks what visual types are supported or how to connect a visual to
  their data model.
tools: pbi-cli
---

# Power BI Visuals Skill

Create and manage visuals on PBIR report pages. No Power BI Desktop connection
is needed -- these commands operate directly on JSON files.

## Adding Visuals

```bash
# Add by alias (pbi-cli resolves to the PBIR type)
pbi visual add --page page_abc123 --type bar
pbi visual add --page page_abc123 --type card --name "Revenue Card"

# Custom position and size (pixels)
pbi visual add --page page_abc123 --type scatter \
    --x 50 --y 400 --width 600 --height 350

# Named visual for easy reference
pbi visual add --page page_abc123 --type combo --name sales_combo
```

Each visual is created as a folder with a `visual.json` file inside the page's
`visuals/` directory. The template includes the correct schema URL and queryState
roles for the chosen type.

## Binding Data

Visuals start empty. Use `visual bind` with `Table[Column]` notation to connect
them to your semantic model. The bind options vary by visual type -- see the
type table below.

```bash
# Bar chart: category axis + value
pbi visual bind mybar --page p1 \
    --category "Geography[Region]" --value "Sales[Revenue]"

# Card: single field
pbi visual bind mycard --page p1 --field "Sales[Total Revenue]"

# Matrix: rows + values + optional column
pbi visual bind mymatrix --page p1 \
    --row "Product[Category]" --value "Sales[Amount]" --value "Sales[Quantity]"

# Scatter: X, Y, detail, optional size and legend
pbi visual bind myscatter --page p1 \
    --x "Sales[Quantity]" --y "Sales[Revenue]" --detail "Product[Name]"

# Combo chart: category + column series + line series
pbi visual bind mycombo --page p1 \
    --category "Calendar[Month]" --column "Sales[Revenue]" --line "Sales[Margin]"

# KPI: indicator + goal + trend axis
pbi visual bind mykpi --page p1 \
    --indicator "Sales[Revenue]" --goal "Sales[Target]" --trend "Calendar[Date]"

# Gauge: value + max/target
pbi visual bind mygauge --page p1 \
    --value "Sales[Revenue]" --max "Sales[Target]"
```

Binding uses ROLE_ALIASES to translate friendly names like `--value` into the PBIR
role name (e.g. `Y`, `Values`, `Data`). Measure vs Column is inferred from the role:
value/indicator/goal/max roles create Measure references, category/row/detail roles
create Column references. Override with `--measure` flag if needed.

## Inspecting and Updating

```bash
# List all visuals on a page
pbi visual list --page page_abc123

# Get full details of one visual
pbi visual get visual_def456 --page page_abc123

# Move, resize, or toggle visibility
pbi visual update vis1 --page p1 --width 600 --height 400
pbi visual update vis1 --page p1 --x 100 --y 200
pbi visual update vis1 --page p1 --hidden
pbi visual update vis1 --page p1 --visible

# Delete a visual
pbi visual delete visual_def456 --page page_abc123
```

## Container Properties

Set border, background, or title on the visual container itself:

```bash
pbi visual set-container vis1 --page p1 --background "#F0F0F0"
pbi visual set-container vis1 --page p1 --border-color "#CCCCCC" --border-width 2
pbi visual set-container vis1 --page p1 --title "Sales by Region"
```

## Visual Calculations

Add DAX calculations that run inside the visual scope:

```bash
pbi visual calc-add vis1 --page p1 --role Values \
    --name "RunningTotal" --expression "RUNNINGSUM([Revenue])"

pbi visual calc-list vis1 --page p1
pbi visual calc-delete vis1 --page p1 --name "RunningTotal"
```

## Bulk Operations

Operate on many visuals at once by filtering with `--type` or `--name-pattern`:

```bash
# Find visuals matching criteria
pbi visual where --page overview --type barChart
pbi visual where --page overview --type kpi --y-min 300

# Bind the same field to ALL bar charts on a page
pbi visual bulk-bind --page overview --type barChart \
    --category "Date[Month]" --value "Sales[Revenue]"

# Resize all KPI cards
pbi visual bulk-update --page overview --type kpi --width 250 --height 120

# Hide all visuals matching a pattern
pbi visual bulk-update --page overview --name-pattern "Temp_*" --hidden

# Delete all placeholders
pbi visual bulk-delete --page overview --name-pattern "Placeholder_*"
```

Filter options for `where`, `bulk-bind`, `bulk-update`, `bulk-delete`:
- `--type` -- PBIR visual type or alias (e.g. `barChart`, `bar`)
- `--name-pattern` -- fnmatch glob on visual name (e.g. `Chart_*`)
- `--x-min`, `--x-max`, `--y-min`, `--y-max` -- position bounds (pixels)

All bulk commands require at least `--type` or `--name-pattern` to prevent
accidental mass operations.

## Supported Visual Types (32)

### Charts

| Alias              | PBIR Type                    | Bind Options                                  |
|--------------------|------------------------------|-----------------------------------------------|
| bar                | barChart                     | --category, --value, --legend                 |
| line               | lineChart                    | --category, --value, --legend                 |
| column             | columnChart                  | --category, --value, --legend                 |
| area               | areaChart                    | --category, --value, --legend                 |
| ribbon             | ribbonChart                  | --category, --value, --legend                 |
| waterfall          | waterfallChart               | --category, --value, --breakdown              |
| stacked_bar        | stackedBarChart              | --category, --value, --legend                 |
| clustered_bar      | clusteredBarChart            | --category, --value, --legend                 |
| clustered_column   | clusteredColumnChart         | --category, --value, --legend                 |
| scatter            | scatterChart                 | --x, --y, --detail, --size, --legend          |
| funnel             | funnelChart                  | --category, --value                           |
| combo              | lineStackedColumnComboChart  | --category, --column, --line, --legend        |
| donut / pie        | donutChart                   | --category, --value, --legend                 |
| treemap            | treemap                      | --category, --value                           |

### Cards and KPIs

| Alias              | PBIR Type                    | Bind Options                                  |
|--------------------|------------------------------|-----------------------------------------------|
| card               | card                         | --field                                       |
| card_visual        | cardVisual                   | --field (modern card)                         |
| card_new           | cardNew                      | --field                                       |
| multi_row_card     | multiRowCard                 | --field                                       |
| kpi                | kpi                          | --indicator, --goal, --trend                  |
| gauge              | gauge                        | --value, --max / --target                     |

### Tables

| Alias              | PBIR Type                    | Bind Options                                  |
|--------------------|------------------------------|-----------------------------------------------|
| table              | tableEx                      | --value                                       |
| matrix             | pivotTable                   | --row, --value, --column                      |

### Slicers

| Alias              | PBIR Type                    | Bind Options                                  |
|--------------------|------------------------------|-----------------------------------------------|
| slicer             | slicer                       | --field                                       |
| text_slicer        | textSlicer                   | --field                                       |
| list_slicer        | listSlicer                   | --field                                       |
| advanced_slicer    | advancedSlicerVisual         | --field (tile/image slicer)                   |

### Maps

| Alias              | PBIR Type                    | Bind Options                                  |
|--------------------|------------------------------|-----------------------------------------------|
| azure_map / map    | azureMap                     | --category, --size                            |

### Decorative and Navigation

| Alias              | PBIR Type                    | Bind Options                                  |
|--------------------|------------------------------|-----------------------------------------------|
| action_button      | actionButton                 | (no data binding)                             |
| image              | image                        | (no data binding)                             |
| shape              | shape                        | (no data binding)                             |
| textbox            | textbox                      | (no data binding)                             |
| page_navigator     | pageNavigator                | (no data binding)                             |

## JSON Output

All commands support `--json` for agent consumption:

```bash
pbi --json visual list --page overview
pbi --json visual get vis1 --page overview
pbi --json visual where --page overview --type barChart
```

## ⚠️ Critical: visual.json Gotchas

### Scatter chart: use `Category` + `Series` roles, NOT `Details`

The `Details` role puts all points in a single group (one dot). To get one point per dimension member (e.g. one dot per month), use:
- `Category` → the label field (e.g. `MonthName`)
- `Series` → a split-by field (e.g. `Year`) to produce separate series

In `visual.json` `queryState`:
```json
"Category": { "projections": [{ "field": { "Column": { ... "Property": "MonthName" }}, "queryRef": "Dim_Date.MonthName" }] },
"Series":   { "projections": [{ "field": { "Column": { ... "Property": "Year" }},     "queryRef": "Dim_Date.Year" }] }
```

### Slicer field bindings: always use `"Column"` field type

A slicer bound with `"Measure"` type will display wrong/blank values. For any dimension column (even one that looks like a measure), use `"Column"`:

```json
{ "field": { "Column": { "Expression": { "SourceRef": { "Entity": "Dim_Date" }}, "Property": "Year" }}}
```

### Slicer default value: Categorical filter with `L` int64 suffix

To default a slicer to a specific value (e.g. Year = 2026), add a `filterConfig` entry of type `"Categorical"`:

```json
"filterConfig": {
  "filters": [{
    "type": "Categorical",
    "filter": {
      "Version": 2,
      "From": [{ "Name": "d", "Entity": "Dim_Date", "Type": 0 }],
      "Where": [{
        "Condition": { "In": {
          "Expressions": [{ "Column": { "Expression": { "SourceRef": { "Source": "d" }}, "Property": "Year" }}],
          "Values": [[{ "Literal": { "Value": "2026L" }}]]
        }}
      }]
    }
  }]
}
```

Note: integer literals require the `L` suffix (`"2026L"`) — without it PBI may ignore the filter.

### Do NOT include `Commands` inside `visual.query`

The 2.7.0 visual schema does not support a `Commands` property in `visual.query`. Its presence causes schema validation failure with Power BI Desktop (March 2026, 2.152.1279.0):

> `'pages/.../visual.json': Property 'Commands' has not been defined and the schema does not allow additional properties. Path 'visual.query.Commands'`

**`pbi visual bind` (and `bulk-bind`) emit this block.** Every bind call writes both `queryState.projections` AND a legacy `Commands: [ { SemanticQueryDataShapeCommand: ... } ]` array. Desktop only accepts `queryState` — `Commands` is redundant because Desktop re-derives the query plan from projections at render time.

**Fix: strip `Commands` from all `visual.json` files as a post-processing step** after any binding operation. One-liner using Python (works cross-platform):

```bash
python -c "import json
from pathlib import Path
for v in Path('Retail.Report/definition/pages').rglob('visual.json'):
    d = json.loads(v.read_text(encoding='utf-8'))
    q = d.get('visual', {}).get('query', {})
    if 'Commands' in q:
        del q['Commands']
        v.write_text(json.dumps(d, indent=2), encoding='utf-8')"
```

Run it after every batch of `pbi visual bind` calls, before opening the `.pbip` in Desktop. This affects **100% of bound visuals**, not just ones using `--legend`. If Desktop rejects a `.pbip` with the `Commands` schema error, run the one-liner and retry — the fix is idempotent.

### Textbox alignment: set `horizontalTextAlignment` on paragraphs

Textbox visuals default to centered text, which can cause overlap when placed next to slicers. Set alignment explicitly in the paragraph definition:

```json
{ "horizontalTextAlignment": "left", "textRuns": [ ... ] }
```

Also position headers carefully relative to slicer bounding boxes — check the `x + width` of any slicer on the same row before placing a textbox.

### Image visuals (`visualType: "image"`) — use `objects.image[].sourceFile`, NOT `general.imageUrl`

`pbi visual add` supports only `bar_chart`, `line_chart`, `card`, `table`, `matrix` — **image visuals must be written by hand**. The correct PBIR 2.7.0 schema for an image visual:

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/visualContainer/2.7.0/schema.json",
  "name": "company_logo",
  "position": {
    "x": 30.0, "y": 2.5, "z": 1000,
    "height": 32.0, "width": 62.0,
    "tabOrder": 1000
  },
  "visual": {
    "visualType": "image",
    "objects": {
      "general": [{ "properties": {} }],
      "image": [{
        "properties": {
          "sourceFile": {
            "image": {
              "name":    { "expr": { "Literal": { "Value": "'company-logo.png'" } } },
              "url":     { "expr": { "ResourcePackageItem": {
                             "PackageName": "RegisteredResources",
                             "PackageType": 1,
                             "ItemName": "company-logo-<hashed-name>.png"
                           } } },
              "scaling": { "expr": { "Literal": { "Value": "'Normal'" } } }
            }
          }
        }
      }]
    },
    "visualContainerObjects": {
      "title": [{ "properties": { "show": { "expr": { "Literal": { "Value": "false" } } } } }]
    },
    "drillFilterOtherVisuals": true
  }
}
```

**Common mistakes (what Desktop rejects):**
- `objects.general[0].properties.imageUrl` — WRONG key. It is `objects.image[0].properties.sourceFile`.
- `objects.general` missing entirely — must be present as `[{ "properties": {} }]`, even empty.
- `imageScalingType` as a sibling of `imageUrl` — WRONG. `scaling` goes INSIDE `sourceFile.image.scaling`.
- Missing `visualContainerObjects.title.show=false` — without it, Desktop renders an auto-generated title above the logo.
- `ItemName` pointing to the pre-rename filename — when Desktop imports an image manually, it hashes the filename (`Sugat.svg.png` → `Sugat.svg2611373602373278.png`). The `name` field shows the original name; the `url.ItemName` must match the hashed name registered in `report.json` `resourcePackages`.

**Scaling values**: `'Normal'` (display 1:1 until clipped), `'Fit'` (letterbox — preserve aspect, fit inside container), `'Fill'` (crop — preserve aspect, fill container).

**Registering the image in `report.json`** (separate step from creating the visual):

```json
"resourcePackages": [{
  "name": "RegisteredResources",
  "type": "RegisteredResources",
  "items": [{
    "name": "company-logo-<hashed-name>.png",
    "path": "company-logo-<hashed-name>.png",
    "type": "Image"
  }]
}]
```

File on disk goes in `Retail.Report/StaticResources/RegisteredResources/<hashed-name>.png`.

### Replicating the same image to many pages

For a logo or brand mark on every page, write the visual once, then copytree to every page's `visuals/` folder:

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

## Data labels on bar charts

All `barChart` / `clusteredBarChart` / `stackedBarChart` visuals must have data labels enabled. Add to `visual.objects`:

```json
"labels": [{
  "properties": {
    "show": {"expr": {"Literal": {"Value": "true"}}},
    "color": {"solid": {"color": {"expr": {"Literal": {"Value": "'#262329'"}}}}},
    "fontFamily": {"expr": {"Literal": {"Value": "'Segoe UI'"}}},
    "fontSize": {"expr": {"Literal": {"Value": "'9'"}}}
  }
}]
```

## Standard visual container formatting

Every chart visual (barChart, lineChart, pivotTable, tableEx) must have in `visual.visualContainerObjects`:

```json
"background":   [{"properties": {"show": true, "color": "#FFFFFF", "transparency": "0D"}}],
"border":       [{"properties": {"show": true, "color": "#DFDDD6", "radius": "4D", "width": "1D"}}],
"visualHeader": [{"properties": {"show": false}}],
"title":        [{"properties": {"show": true/false, "fontFamily": "'Segoe UI'", "fontSize": "'12'", "bold": true, "alignment": "'center'", "fontColor": "#262329"}}]
```

Title is `show: true` for line/bar charts, `show: false` for cards, matrix, slicers.

## Time-axis hierarchy

Any line chart with a date field on the category axis must use the two-level hierarchy:
- `DimDate[Year]` — active: true (default level shown)
- `DimDate[MonthName]` — active: false (drill-down available)

Never use YearQuarter or YearMonth on the category axis.

## `tableEx` binding — all fields in `Values`, dims first

Power BI Desktop renders `tableEx` visuals by reading **only `queryState.Values.projections`**. A sibling well like `queryState.category` or `queryState.Category` (whether created manually or left over from another visual type) is silently ignored — columns bound there never appear in the rendered table even though they show up fine in the field list. Users see a table with only the measures and assume the dim binding was lost.

**Correct shape for tableEx:**
```json
"queryState": {
  "Values": {
    "projections": [
      // Dim columns FIRST, in desired display order
      {"field": {"Column": {"Expression": {"SourceRef": {"Entity": "DimCustomer"}}, "Property": "Customer"}},
       "queryRef": "DimCustomer.Customer", "nativeQueryRef": "Customer", "active": true},
      {"field": {"Column": {"Expression": {"SourceRef": {"Entity": "DimCustomer"}}, "Property": "Chain"}},
       "queryRef": "DimCustomer.Chain", "nativeQueryRef": "Chain", "active": true},
      // Then measures
      {"field": {"Measure": {"Expression": {"SourceRef": {"Entity": "FactSales"}}, "Property": "Net Sales"}},
       "queryRef": "FactSales.Net Sales", "nativeQueryRef": "Net Sales", "active": true}
    ]
  }
}
```

Rules:
- **One well only.** Never add `category` / `Category` to a `tableEx` queryState — Desktop ignores it.
- **Column order = display order.** Projections render left-to-right in the order they appear in the `projections` array.
- **Mix column and measure types freely** — both use the same `Values` well; the `field` discriminator (`Column` vs `Measure`) is what the engine uses to decide grouping vs aggregation.
- **Contrast with charts.** Line/bar charts *do* use separate wells (`Category`, `Values`, `Series`). That's why this mistake happens — developers copy the chart pattern into tables. For `tableEx` and `pivotTable` rows, flatten everything into `Values`.

## Table bindings: dim column that explodes rows with repeating measure values

When a table binds a dim column whose values don't exist in the fact (e.g. `Customer × Channel` where AR is a customer-level measure), each customer gets one row *per channel*, and all channels show the same AR value — repeated identically — while sales/GM only populate for the channel actually used. Users read 7× rows of blank cells and identical AR numbers.

**Rule:** never add a dim column to a table's Values projections that is *one-to-many* with the fact being shown. Either:
- Remove the offending dim column (if it's not essential to the grouping), OR
- Switch to a different fact table / measure that joins meaningfully to that dim, OR
- Use a matrix with the second dim on Columns (so blanks collapse visually).

Symptom to watch for: measure values repeating identically across groups + many blank cells in a table.

## Bar chart data labels must sit `OutsideEnd`

Default label position on bars overlaps the bar fill — values become unreadable against the colored bar. Always set:
```json
"labels": [{"properties": {
  "show": {"expr": {"Literal": {"Value": "true"}}},
  "labelPosition": {"expr": {"Literal": {"Value": "'OutsideEnd'"}}},
  "color": {"solid": {"color": {"expr": {"Literal": {"Value": "'#262329'"}}}}},
  "fontFamily": {"expr": {"Literal": {"Value": "'Segoe UI'"}}},
  "fontSize": {"expr": {"Literal": {"Value": "'9'"}}}
}}]
```

## Line chart with only 2 points = diagonal straight line

Symptom: a `by Year` line chart on 2 years of data shows a single diagonal straight line — no signal, just an endpoint-to-endpoint diagonal. The visual is bound to `DimDate[Year]` only.

**Fix:** add `DimDate[MonthName]` as a second level in the Categories projections, with BOTH levels `active: true` so the chart renders monthly points that aggregate into the Year header. (Setting MonthName `active: false` means drill-down-available-but-not-expanded — which leaves the chart at the Year level, i.e. still two points.)

## Unicode escape literal in visual.json binding strings

If a measure name contains `³`, `°`, `₪`, `µ` (Water Intensity `(m³/T)`, Temperature `(°C)`, etc.) and the `visual.json` binding was copy-pasted from a terminal that escaped the character, you'll see the literal 6-character sequence `\u00b3` in `Property`, `queryRef`, and `nativeQueryRef`. The measure lookup fails and Desktop renders the KPI card as a visual-level error ("See details" with X icon).

**Fix:** replace the literal escape with the actual unicode character. `\u00b3` → `³`, `\u00b0` → `°`, `\u20aa` → `₪`, `\u00b5` → `µ`. Desktop always writes the real character, never the escape. If you edit bindings by hand, verify the JSON contains `³` directly.
