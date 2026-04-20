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

## Composite "hero" KPI tile (mockup-style card)

Mockup-style KPI cards (title + big value + colored delta pill + last-year value + sparkline, all in one rounded tile) cannot be built with a single Power BI visual. The classic `card` shows one number; `cardVisual` / `cardNew` are closer but still constrained. The reliable way is to **overlay 5–6 visuals inside one square frame**.

**Rules learned from production refactor — apply by default, do not deviate:**

1. **Square aspect ratio.** Frame is **230×230 px** (not 240×340 or other tall rectangles). Tall composites look unbalanced; the square reads as a "dashboard tile" in modern SaaS style. Place at any `(x, y)` you want, but keep `width == height == 230`.

2. **Frame = invisible card.** The bottom layer (`z=0`) is a `card` visual with a real measure projection (any measure works) but rendered invisibly:
   ```json
   "objects": {
     "labels":         [{"properties": {"color": "#FFFFFF", "fontSize": 8D}}],
     "categoryLabels": [{"properties": {"show": false}}]
   }
   ```
   Its only role is to render the white background + rounded border (`radius: 16D`). Everything else stacks above with higher `z`.

3. **Sparkline = `areaChart`, never `lineChart`.** The filled area reads better at small sizes (~80px tall) — a stroke-only line is too thin to convey trend at thumbnail scale. Always include an explicit `query.sortDefinition` sorting `DimDate[Year]` then `DimDate[MonthName]` ascending. Without it, points may render in alphabetical month order ("April, August, December…") and the line zigzags.

4. **Strip card padding for micro-elements.** Pills, last-year values, and any card embedded inside a parent tile need `padding` set to zero on all four sides:
   ```json
   "objects": {
     "padding": [{"properties": {
       "top":    {"expr": {"Literal": {"Value": "0D"}}},
       "bottom": {"expr": {"Literal": {"Value": "0D"}}},
       "left":   {"expr": {"Literal": {"Value": "0D"}}},
       "right":  {"expr": {"Literal": {"Value": "0D"}}}
     }}]
   }
   ```
   Default card padding wastes ~10 px on each side — half the height of a 28 px pill. Without `padding=0` the value gets clipped or appears in tiny font.

5. **Hide `categoryLabels` on every overlaid card.** The measure name is redundant inside a composite tile — context (title, position, color) tells the user what the number means. `categoryLabels.show = false` on all child cards.

6. **No backgrounds, no borders on overlays.** Every overlaid card/lineChart/areaChart needs `visualContainerObjects.background.show=false` and `border.show=false`. Only the frame (z=0) draws the visible tile boundary.

**Reference layout zones inside the 230×230 frame at `(520, 180)`:**

| Element       | x      | y      | w   | h    | z | Notes |
|---------------|--------|--------|-----|------|---|-------|
| Frame (bg)    | 520    | 180    | 230 | 230  | 0 | invisible card |
| Title         | 540    | 190.83 | 200 | 31.7 | 1 | container `title.show=true`, value invisible |
| Value         | 540    | 216.67 | 200 | 50   | 2 | fontSize 24, displayUnits Auto or 1M, precision 1 |
| Delta pill    | 592.5  | 266.67 | 95  | 30.8 | 3 | bg `#2D7D6E` (green) or `#B33A3A` (red), radius 14D, fontSize 13 white bold |
| Last Year     | 627.5  | 315    | 132.5 | 27.5 | 4 | small, no categoryLabel |
| Sparkline     | 540    | 326.78 | 199 | 80   | 5 | `areaChart` with sortDefinition |

**When to use this pattern:** "hero" KPIs — one large decorative tile per page, intended as the main visual element.

### Rectangular variant (for the standard 5-card KPI row, ~220×84)

When upgrading the dense KPI row at the top of analytical pages — typical dimensions 190–240 px wide × 84 px tall — apply a **two-column split** instead of stacking:

```
┌───────────────────────────────┬──────────────┐
│ Title (10pt bold, gray-blue)  │              │
│ $242K (18pt bold, navy)       │   ╱╲    ╱╲   │  ← areaChart
│ ▲10.9%  LY: $218K             │  ╱  ╲__╱  ╲  │   right column
└───────────────────────────────┴──────────────┘
   ~60% w (text stack)             ~38% w (spark)
```

Layout zones for a `(x, y, w, 84)` rectangle:
- `spark_w = max(60, int(w * 0.30))` — keep sparkline narrow, ~30% of width. Wider eats space the texts need.
- `text_w = w - spark_w - 14`, gap of 8px
- Title:     `(x+8, y+6, text_w, 16)` — fontSize 10, bold, left-align
- Value:     `(x+8, y+22, text_w, 30)` — fontSize 18, bold, displayUnits Auto
- Pill:      `(x+8, y+h-26, 70, 22)` — fontSize 11, white on green. Make it big enough to read the % comfortably; small pills (e.g. 56×18) clip the value at small widths. **Critical: set `padding: [{...all zero}]` on BOTH `objects.padding` AND `visualContainerObjects.padding`** — the inner card padding clips the % text inside the pill bounds.
- LY label (separate visual): `(x+8+76, y+h-24, 22, 18)` — card with value invisible, container title shown as `'LY:'` 9pt gray. **Do NOT use the title of the LY value card itself for the "LY:" label** — the title slot occupies the full container width even when the text is short, pushing the value down or clipping it. Render label and value as two separate visuals side-by-side.
- LY value (separate visual): `(x+8+76+24, y+h-24, text_w-100, 18)` — card with PY measure, no title, fontSize 10 bold dark
- Sparkline: `(x+w-spark_w-6, y+6, spark_w, h-12)` — fills right column

**Tier classification — not every measure makes sense for pill+LY+sparkline:**
- **Lite** (frame+title+value only): measures that are point-in-time states with no FactDate relationship (`DimX[IsActive]=TRUE` style — same value in CY and PY → meaningless YoY, flat sparkline). Examples: Active Certifications, Expiring Soon, Audit Readiness Score.
- **No-pill** (frame+title+value+sparkline): measures that vary over time but where YoY is meta or already-a-comparison. Examples: `Net Sales YoY %` (YoY of YoY), `Net Sales vs Budget %` (already a variance), `Headcount Variance` (already a variance — YoY of variance is volatile noise).
- **Full** (frame+title+value+pill+LY+sparkline): everything else — requires the model to have `<Measure> PY` and `<Measure> YoY %` companion measures.

**Generating PY+YoY% companion measures in bulk:** when the model has dozens of measures without time-intelligence companions, generate them following the project's MaxDate pattern (NEVER use `SAMEPERIODLASTYEAR` or `DATEADD` directly — they return BLANK when filter context is empty, which breaks the card display). Pattern, parameterized by fact-table date column:

```dax
<Measure> PY =
    VAR MaxDate = CALCULATE(MAX(<Fact>[<DateCol>]), ALL())
    RETURN
        CALCULATE([<Measure>], YEAR(DimDate[Date]) = YEAR(MaxDate) - 1)

<Measure> YoY % =
    VAR MaxDate = CALCULATE(MAX(<Fact>[<DateCol>]), ALL())
    VAR CY = CALCULATE([<Measure>], YEAR(DimDate[Date]) = YEAR(MaxDate))
    VAR PY = CALCULATE([<Measure>], YEAR(DimDate[Date]) = YEAR(MaxDate) - 1)
    RETURN IF(ISBLANK(PY), BLANK(), DIVIDE(CY - PY, PY))
```

The `IF(ISBLANK(PY), BLANK(), ...)` guard hides the pill/LY for the earliest year where there is no comparable (data start year). Without it the pill shows `-100%` falsely.

Inherit the base measure's `formatString` and `displayFolder` for the PY measure so it sorts next to its base; use `0.0%` format for the YoY % measure.

### Pill conditional formatting — green/red by business polarity

A pill that is **always green** is misleading. Half the measures in a typical FMCG / retail / supply-chain model are **lower-is-better** (DSO, DIO, AR, defect rate, downtime, scrap, complaints, freight cost) — for those, a +5% YoY is BAD and must render red.

**Pattern: one `<Measure> YoY Color` companion per base measure.** Returns a hex string the pill background+border bind to via field-value conditional formatting.

```dax
<Measure> YoY Color =
    VAR V = [<Measure> YoY %]
    RETURN
        IF(
            ISBLANK(V),
            "#9E9E9E",   -- gray for first-year edge case (no comparable)
            IF(<cond>, "#2D7D6E", "#B33A3A")
        )
```

Where `<cond>` flips by polarity:
- **Higher-is-better** (default — sales, margin, OTIF, market share, ROI, throughput, OEE, headcount, training completion, MTBF): `V >= 0`
- **Lower-is-better** (costs, balances, failures): `V <= 0`

Mark the color measure `isHidden` and place in a `99 Conditional Formatting` displayFolder so it doesn't pollute the field list.

**Lower-is-better classification cheat sheet** (memorize the categories, not the individual measures — apply judgment to new ones):
- **Working capital balances**: DSO, DIO, AR Outstanding/Disputed/Past Due/High Risk, Cash Conversion Cycle, Inventory Value, % Aged, Slow-Moving SKUs, Stockout Rate
- **Costs**: Freight Cost (any variant), Total Spend, Avg Price (raw materials), Campaign Spend, Trade Spend, Lead Time
- **Quality / safety failures**: Complaint Rate, NCR Count, Critical/Hold Events, Open/Overdue CAPAs, LTI, LTIFR, Days Lost, Near-Miss
- **Operations waste**: Downtime (events/duration), MTTR, Scrap %, Unplanned Downtime %, Energy/Water Intensity
- **HR attrition / strain**: Turnover %, Absenteeism %, Overtime %
- **Supplier risk**: Defect Rate (ppm), HHI (concentration), PPV (variance vs std)
- **Competitor share**: Competitor Value (we want competitor's share to drop)

**PBIR binding** — the pill's `visualContainerObjects.background.color` and `border.color` both point to the color measure as field value:

```json
"color": {
  "solid": {
    "color": {
      "expr": {
        "Measure": {
          "Expression": {"SourceRef": {"Entity": "<Fact>"}},
          "Property": "<Measure> YoY Color"
        }
      }
    }
  }
}
```

This is "field value" conditional formatting — Desktop reads the measure result (a hex string) and uses it directly as the color. No `FillRule` / `linearGradient` envelope needed for this pattern.

**Why the gray BLANK branch matters:** without it, the pill renders as the chart palette default (often white/transparent) when YoY is BLANK, making the pill disappear into the card. Gray `#9E9E9E` keeps the pill visible while signaling "no comparable" honestly.

**Title text via container header:** since cards can't show static text, the "Sales" title is rendered as the visual container's title:
```json
"visualContainerObjects": {
  "title": [{
    "properties": {
      "show":      {"expr": {"Literal": {"Value": "true"}}},
      "text":      {"expr": {"Literal": {"Value": "'Sales'"}}},
      "fontSize":  {"expr": {"Literal": {"Value": "'18'"}}},
      "bold":      {"expr": {"Literal": {"Value": "true"}}},
      "alignment": {"expr": {"Literal": {"Value": "'left'"}}}
    }
  }]
}
```

## Bilingual / RTL pages — what works in PBIR 2.7.0 and what doesn't

**Don't try to build an in-page language toggle from scratch.** PBIR 2.7.0 supports `actionButton` visuals + bookmark JSON at `Retail.Report/definition/bookmarks/<id>.bookmark.json`, but the schemas for `actionButton.objects.{shape,fill,outline,text}` properties and for `visualContainerObjects.visualLink` are sparsely documented. `pbi report validate` accepts hand-written guesses; Desktop rejects them. Concrete failures observed:
- `visualContainerObjects.general[0].properties.show = false` → `"Property 'show' has not been defined and the schema does not allow additional properties"` from Desktop. There is no per-visual visibility property in PBIR 2.7.0 visual.json.
- `visualContainerObjects.visualLink[0].properties.{type,bookmark}` → unverified; Desktop may reject.
- Bookmark JSON `explorationState.sections.<page>.visualContainers.<vis>.singleVisual.display.mode = "hidden"` → unverified format.

**Robust path: two pages instead of one toggle.** Copy the source page, mirror coords for RTL (`x' = 1280 - x - width` applied to every visual's `position.x`), translate titles in-place via `visualContainerObjects.title.text` Literals, and rebind columns to language-specific companions. The page navigator becomes the switch. To add a real button toggle later, generate bookmarks + buttons via Desktop UI (Insert → Buttons → Blank, Action → Bookmark) — Desktop emits valid PBIR JSON that you can then version-control.

### Mirror-for-RTL coordinate flip

```python
new_x = canvas_width - old_x - width
```

Apply to `position.x` of every visual (including slicers and logo) on the mirrored page. Don't touch `y`, `width`, `height`, or `tabOrder`. Also align titles to the right: `visualContainerObjects.title[0].properties.alignment = "'right'"` so multi-word titles read as RTL.

### HE companion columns for translated axes/categories/slicer values

Hebrew (or any non-EN) text in chart axes, slicer dropdowns, or categorical bars requires a sibling column with pre-translated data. The column display name doesn't help — that only renames the field's heading, not the row values. Add to the dim's M partition:

```m
HebMonths = {"ינואר","פברואר","מרץ","אפריל","מאי","יוני","יולי","אוגוסט","ספטמבר","אוקטובר","נובמבר","דצמבר"},
WithMoH   = Table.AddColumn(WithMoN, "MonthNameHE", each HebMonths{Date.Month([Date]) - 1}, type text),
```

Plus a column block in the table TMDL with the same `sortByColumn` as the EN sibling:

```tmdl
column MonthNameHE
    dataType: string
    lineageTag: <new-uuid>
    sourceColumn: MonthNameHE
    sortByColumn: MonthNum
```

Then rebind in the HE visuals: `Property: "MonthName"` → `"MonthNameHE"`, plus update `queryRef` and `nativeQueryRef` accordingly. Same pattern works for ProductNameHE, ChannelHE, BrandHE, etc. Build only the columns needed for the visuals on the bilingual page — don't blanket-create HE for every dim.

### Slicer "field label" override (Hebrew or any custom text)

A slicer's default header renders the bound column's name ("Year", "Channel"). To show a custom label without changing the column's model-wide display name:

```json
"objects": {
  "header": [{"properties": {"show": {"expr": {"Literal": {"Value": "false"}}}}}]
},
"visualContainerObjects": {
  "title": [{
    "properties": {
      "show": {"expr": {"Literal": {"Value": "true"}}},
      "text": {"expr": {"Literal": {"Value": "'שנה'"}}},
      "alignment": {"expr": {"Literal": {"Value": "'right'"}}},
      "fontSize": {"expr": {"Literal": {"Value": "'11'"}}},
      "bold": {"expr": {"Literal": {"Value": "true"}}},
      "fontFamily": {"expr": {"Literal": {"Value": "'Segoe UI'"}}}
    }
  }]
}
```

The container title sits above the slicer where the auto-header used to be. Use `alignment: 'right'` for RTL languages.

## Unicode escape literal in visual.json binding strings

If a measure name contains `³`, `°`, `₪`, `µ` (Water Intensity `(m³/T)`, Temperature `(°C)`, etc.) and the `visual.json` binding was copy-pasted from a terminal that escaped the character, you'll see the literal 6-character sequence `\u00b3` in `Property`, `queryRef`, and `nativeQueryRef`. The measure lookup fails and Desktop renders the KPI card as a visual-level error ("See details" with X icon).

**Fix:** replace the literal escape with the actual unicode character. `\u00b3` → `³`, `\u00b0` → `°`, `\u20aa` → `₪`, `\u00b5` → `µ`. Desktop always writes the real character, never the escape. If you edit bindings by hand, verify the JSON contains `³` directly.
