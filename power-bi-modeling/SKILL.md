---
name: Power BI Modeling
description: Create and manage Power BI semantic model structure using pbi-cli -- tables, columns, measures, relationships, hierarchies, calculation groups, and date/calendar tables. Invoke this skill whenever the user says "create table", "add measure", "add column", "create relationship", "date table", "calendar table", "star schema", "mark as date table", "add hierarchy", "calculation group", or any model-building task. Also invoke when creating multiple measures at once -- the skill contains critical guidance on multi-line DAX expression handling.
tools: pbi-cli
---

# Power BI Modeling Skill

Use pbi-cli to manage semantic model structure. Requires `pipx install pbi-cli-tool`, `pbi-cli skills install`, and `pbi connect`.

## Prerequisites

```bash
pipx install pbi-cli-tool
pbi-cli skills install
pbi connect
```

## Tables

```bash
pbi table list                                    # List all tables
pbi table get Sales                               # Get table details
pbi table create Sales --mode Import              # Create table
pbi table delete OldTable                         # Delete table
pbi table rename OldName NewName                  # Rename table
pbi table refresh Sales --type Full               # Refresh table data
pbi table schema Sales                            # Get table schema
pbi table mark-date Calendar --date-column Date   # Mark as date table
```

## Columns

```bash
pbi column list --table Sales                                       # List columns
pbi column get Amount --table Sales                                 # Get column details
pbi column create Revenue --table Sales --data-type double --source-column Revenue  # Data column
pbi column create Profit --table Sales --expression "[Revenue]-[Cost]"              # Calculated
pbi column delete OldCol --table Sales                              # Delete column
pbi column rename OldName NewName --table Sales                     # Rename column
```

## Measures

```bash
pbi measure list                                                    # List all measures
pbi measure list --table Sales                                      # Filter by table
pbi measure get "Total Revenue" --table Sales                       # Get details
pbi measure create "Total Revenue" -e "SUM(Sales[Revenue])" -t Sales                        # Basic
pbi measure create "Revenue $" -e "SUM(Sales[Revenue])" -t Sales --format-string "\$#,##0"  # Formatted
pbi measure create "KPI" -e "..." -t Sales --folder "Key Measures" --description "Main KPI" # With metadata
pbi measure update "Total Revenue" -t Sales -e "SUMX(Sales, Sales[Qty]*Sales[Price])"       # Update expression
pbi measure delete "Old Measure" -t Sales                           # Delete
pbi measure rename "Old" "New" -t Sales                             # Rename
pbi measure move "Revenue" -t Sales --to-table Finance              # Move to another table
```

**Multi-line DAX in measure expressions:** The `-e` flag passes DAX as a shell argument, which collapses newlines. For simple expressions like `SUM(Sales[Amount])` or `DIVIDE([A] - [B], [B])` this works fine. For complex expressions using VAR/RETURN, pipe from stdin instead:

```bash
echo 'VAR TotalSales = SUM(Sales[Amount])
VAR TotalCost = SUM(Sales[Cost])
RETURN TotalSales - TotalCost' | pbi measure create "Profit" -e - -t Sales
```

See the **power-bi-dax** skill for the full explanation and more workarounds.

## Relationships

```bash
pbi relationship list                              # List all relationships
pbi relationship get RelName                       # Get details
pbi relationship create \
  --from-table Sales --from-column ProductKey \
  --to-table Products --to-column ProductKey       # Create relationship
pbi relationship delete RelName                    # Delete
pbi relationship find --table Sales                # Find relationships for a table
pbi relationship activate RelName                  # Activate
pbi relationship deactivate RelName                # Deactivate
```

## Hierarchies

```bash
pbi hierarchy list --table Date                    # List hierarchies
pbi hierarchy get "Calendar" --table Date          # Get details
pbi hierarchy create "Calendar" --table Date       # Create
pbi hierarchy delete "Calendar" --table Date       # Delete
```

## Calculation Groups

```bash
pbi calc-group list                                # List calculation groups
pbi calc-group create "Time Intelligence" --description "Time calcs"  # Create group
pbi calc-group items "Time Intelligence"           # List items
pbi calc-group create-item "YTD" \
  --group "Time Intelligence" \
  --expression "CALCULATE(SELECTEDMEASURE(), DATESYTD(Calendar[Date]))"  # Add item
pbi calc-group delete "Time Intelligence"          # Delete group
```

## Creating a Date/Calendar Table

Date tables are essential for time intelligence functions (TOTALYTD, SAMEPERIODLASTYEAR, DATEADD, etc.).

```bash
# Create a calculated date table with DAX (covers full calendar years)
pbi table create Calendar \
  --dax-expression "ADDCOLUMNS(CALENDAR(DATE(2023,1,1), DATE(2024,12,31)), \"Year\", YEAR([Date]), \"MonthNumber\", MONTH([Date]), \"MonthName\", FORMAT([Date], \"MMMM\"), \"Quarter\", \"Q\" & FORMAT([Date], \"Q\"))"

# Mark it as a date table (required for time intelligence)
pbi table mark-date Calendar --date-column Date

# Verify it's recognized as a date table
pbi calendar list
```

## Workflow: Create a Star Schema

```bash
# 1. Create fact table
pbi table create Sales --mode Import

# 2. Create dimension tables
pbi table create Products --mode Import
pbi table create Calendar --mode Import

# 3. Create relationships
pbi relationship create --from-table Sales --from-column ProductKey --to-table Products --to-column ProductKey
pbi relationship create --from-table Sales --from-column DateKey --to-table Calendar --to-column DateKey

# 4. Mark date table
pbi table mark-date Calendar --date-column Date

# 5. Add measures
pbi measure create "Total Revenue" -e "SUM(Sales[Revenue])" -t Sales --format-string "\$#,##0"
pbi measure create "Total Qty" -e "SUM(Sales[Quantity])" -t Sales --format-string "#,##0"
pbi measure create "Avg Price" -e "AVERAGE(Sales[UnitPrice])" -t Sales --format-string "\$#,##0.00"

# 6. Verify
pbi table list
pbi measure list
pbi relationship list
```

## Best Practices

- Use format strings for currency (`$#,##0`), percentage (`0.0%`), and integer (`#,##0`) measures
- Organize measures into display folders by business domain
- Always mark calendar tables with `mark-date` for time intelligence
- Use `--json` flag when scripting: `pbi --json measure list`
- Export TMDL for version control: `pbi database export-tmdl ./model/`

## ⚠️ Critical: TMDL Gotchas

### Multi-line DAX indentation in TMDL files

When hand-editing `.tmdl` files, multi-line DAX expressions (VAR/RETURN) must be indented at **3 tabs** — one level deeper than measure properties (2 tabs). The `-e` flag collapses newlines, so always use `--file` or stdin piping for VAR/RETURN expressions:

```bash
cat measure.dax | pbi measure create "MyMeasure" -e - -t MyTable
```

### Sorting text columns (MonthName, DayName)

Without a sort override, text columns sort alphabetically. A MonthName column in a chart shows Apr, Aug, Dec ... instead of Jan, Feb, Mar.

Add `sortByColumn` in the TMDL column definition:
```tmdl
column MonthName
    dataType: string
    sourceColumn: MonthName
    sortByColumn: Month    ← references the numeric Month column
```

Or via pbi-cli:
```bash
pbi column update MonthName --table Dim_Date --sort-by-column Month
```

### Relationships: location in TMDL

All relationships must go in `relationships.tmdl` only — **never** in `model.tmdl`. Defining a relationship in both files produces:

> `"ambiguous paths: X→Y and X→Y"` (same path twice)

See the **power-bi-deployment** skill for the full explanation.

### Fact table design: when SUM returns equal values for all dimension members

If all dimension members show identical values (e.g. each sales channel = same total), the fact table likely lacks the FK column for that dimension. Example: `Fact_Revenue` with no `ChannelID` — every filter on `Dim_Channel` returns the full sum unchanged.

Fix: use a different fact table that has the FK, and create dedicated measures over it:
```bash
pbi measure create "Revenue by Channel" \
    -e "SUM(Fact_Reservations[GrossRevenue])" \
    -t _Measures --folder "Channels"
```

### Column and measure names share a namespace within a table

You **cannot** have a column and a measure with the same name in the same table. The Fabric / Analysis Services server rejects the import with:

> `The '<oii>OEE</oii>' measure cannot be created because a column with the same name already exists.`

This fails at deploy time even when the offline TMDL build succeeds, because the engine validates the namespace on import. Common trap: a row-level numeric column (e.g. `FactProduction[OEE]` per shift) and an aggregate measure over it (e.g. `[OEE] = AVERAGE(FactProduction[OEE])`).

**Convention:** when both the row-level value and an aggregate measure are needed, suffix the column with `Row` and keep the measure name clean. Users see `[OEE]`, `[GRPs]`, `[Impressions]`, `[Clicks]` in the Fields pane without collision.

```bash
pbi column rename OEE OEERow --table FactProduction
pbi measure update OEE -t FactProduction -e "AVERAGE(FactProduction[OEERow])"
```

Before creating an aggregate measure, list the target table's columns and verify no collision:
```bash
pbi column list --table FactProduction | grep -i "^<proposed-measure-name>$"
```

### Column rename does NOT update DAX references in measures

`pbi column rename` updates the column definition but **does not** repoint measure expressions that reference it. Measures silently break (referencing a non-existent column). Always verify and patch:

```bash
pbi column rename OEE OEERow --table FactProduction
pbi measure get OEE -t FactProduction          # check expression still reads FactProduction[OEE]
pbi measure update OEE -t FactProduction \
    -e "AVERAGE(FactProduction[OEERow])"       # repoint manually
```

Table renames, relationship renames, and measure renames DO propagate automatically — only column renames require the manual follow-up.

### IsKey on Import-mode columns fails

When creating a table with `mode: Import`, setting `IsKey: true` on a column returns:

> `IsKey is only supported for DirectQuery tables. In non-DirectQuery models, key columns are managed automatically by the engine via the RowNumber column.`

Omit `IsKey` on Import tables. The engine builds its own row identity. Only set `IsKey` on `mode: DirectQuery` or DirectLake tables.

### MarkAsDateTable requires the date column to exist and be DateTime

`table mark-date` fails with `"Column 'Date' not found in table 'DimDate'"` if the table was just created offline and columns were not materialized yet (common with `DaxExpression`-based calculated tables — see power-bi-deployment skill for why). Order of operations:

1. Create the table with an `mExpression` that explicitly defines all columns including `Date` (type `DateTime`)
2. `pbi table get DimDate` or `pbi table schema DimDate` — verify `Date` appears in the column list
3. `pbi table mark-date DimDate --date-column Date`

## Time-intelligence patterns for synthetic/demo data

When data ends before today (e.g., synthetic data ending Dec 2025), standard DAX time-intelligence functions (TOTALYTD, TOTALMTD, SAMEPERIODLASTYEAR) return BLANK because they anchor to TODAY().

**Pattern: anchor to last available data date**

```dax
-- YTD
Net Sales YTD =
VAR MaxDate = CALCULATE(MAX(FactSales[SalesDate]), ALL())
RETURN CALCULATE([Net Sales],
    DATESBETWEEN(DimDate[Date], DATE(YEAR(MaxDate), 1, 1), MaxDate))

-- MTD
Net Sales MTD =
VAR MaxDate = CALCULATE(MAX(FactSales[SalesDate]), ALL())
RETURN CALCULATE([Net Sales],
    DATESBETWEEN(DimDate[Date], DATE(YEAR(MaxDate), MONTH(MaxDate), 1), MaxDate))

-- YoY %
Net Sales YoY % =
VAR MaxDate = CALCULATE(MAX(FactSales[SalesDate]), ALL())
VAR CY = CALCULATE([Net Sales], YEAR(DimDate[Date]) = YEAR(MaxDate))
VAR PY = CALCULATE([Net Sales], YEAR(DimDate[Date]) = YEAR(MaxDate) - 1)
RETURN DIVIDE(CY - PY, PY)
```

Never use: `TOTALYTD`, `TOTALMTD`, `TOTALQTD`, `SAMEPERIODLASTYEAR` — all anchor to TODAY().

## Period-normalized rate measures

Any `rate %` measure that counts events (Turnover, Training Completion, Defect %, Attendance Incident Rate) must scale its denominator to the filter-context period length. Without this, a quarterly slice shows 4× the annual rate — stakeholders see "Turnover 116%" and lose trust.

```dax
Turnover % =
VAR DaysInPeriod = 1 + DATEDIFF(MIN(DimDate[Date]), MAX(DimDate[Date]), DAY)
VAR MonthsInPeriod = DaysInPeriod / 30.44
VAR Departures = [Terminations Count]
VAR AnnualizedRate = DIVIDE(Departures, [Headcount]) * (12 / MonthsInPeriod)
RETURN AnnualizedRate
```

Same pattern for denominator-based completion rates:
```dax
Mandatory Training Completion % =
VAR MonthsInPeriod = (1 + DATEDIFF(MIN(DimDate[Date]), MAX(DimDate[Date]), DAY)) / 30.44
VAR Required = [Headcount] * (MonthsInPeriod / 12)   -- scales with period
VAR Completed = [Mandatory Training Events Count]
RETURN DIVIDE(Completed, Required)
```

## BLANK-safe future-date measures

Measures that compute distance to a future event (`Days to Next Expiry`, `Days Until Next Audit`, `Time to Next Maintenance`) must guard `DATEDIFF` against a BLANK anchor. Otherwise the card renders a visual-level error ("See details" with X icon) instead of gracefully showing BLANK.

```dax
// WRONG — DATEDIFF(TODAY(), BLANK(), DAY) errors the visual
Days to Next Expiry = DATEDIFF(TODAY(), MINX(FILTER(...), [ExpiryDate]), DAY)

// CORRECT
Days to Next Expiry =
VAR NextExp = CALCULATE(
    MIN(DimCertification[ExpiryDate]),
    DimCertification[IsActive] = TRUE,
    DimCertification[ExpiryDate] >= TODAY()
)
RETURN IF(ISBLANK(NextExp), BLANK(), DATEDIFF(TODAY(), NextExp, DAY))
```

## Comparison-period alignment (vs Budget, vs PY, vs Plan)

A measure like `vs Budget %` must compare matched periods. When the filter context is "All Years" and `Budget` is a single year's plan × 1.08, the naive `(NS − Budget) / Budget` explodes to 160%+. Always anchor both numerator and denominator to the same year window:

```dax
Net Sales vs Budget % =
VAR MaxDate = CALCULATE(MAX(FactSales[SalesDate]), ALL())
VAR CY_NS      = CALCULATE([Net Sales],        YEAR(DimDate[Date]) = YEAR(MaxDate))
VAR CY_Budget  = CALCULATE([Budget Net Sales], YEAR(DimDate[Date]) = YEAR(MaxDate))
RETURN DIVIDE(CY_NS - CY_Budget, CY_Budget)
```

## Contribution Margin vs Gross Margin

`Contribution Margin % = Gross Margin %` is a bug. Contribution Margin = Net Sales − COGS − Variable Selling Costs (freight, commissions, trade spend), so it must be ~6–10 percentage points below GM%. If a model doesn't have a separate variable-selling column, use a proxy:

```dax
Contribution Margin = [Net Sales] - [COGS] - [Net Sales] * 0.08
Contribution Margin % = DIVIDE([Contribution Margin], [Net Sales])
```

## Subset-count measures — Overdue / Overdue% rule

Any "bad subset of universe" measure (Overdue CAPAs, Rejected Submissions, Failed Audits) must be a strict subset of its parent universe. If Overdue is defined independently of Open, you can see Overdue% > 100% — logically impossible.

```dax
// WRONG — over-counts
Overdue CAPAs = CALCULATE(COUNTROWS(FactQualityEvent), FactQualityEvent[IsOverdue] = TRUE)

// CORRECT — subset of Open
Overdue CAPAs =
CALCULATE(
    COUNTROWS(FactQualityEvent),
    FactQualityEvent[IsOpen] = TRUE,
    FactQualityEvent[IsOverdue] = TRUE
)

// Denominator includes both open-not-overdue and open-overdue:
Overdue CAPA % = DIVIDE([Overdue CAPAs], [Open CAPAs])  -- where Open CAPAs already includes overdue ones
```

Verify in the underlying M partition too: `isOverdue` must be `isOpen AND past_due`, not standalone.

## BLANK-safe count/percentage measures (force 0 instead of BLANK)

`CALCULATE(COUNTROWS(...), filter)` returns **BLANK** (not 0) when the filter yields zero rows. When such a count feeds a `DIVIDE`, the whole percentage shows as (Blank) on the card — users read it as "measure broken" when actually the filter just didn't match anything in current context.

Force 0 with the `+ 0` idiom or a third `DIVIDE` argument:

```dax
// WRONG — returns BLANK when no rows match IsOverdue=TRUE
Overdue CAPA % = DIVIDE([Overdue CAPAs], [Open CAPAs])

// CORRECT — explicit 0 default, BLANK replaced with 0 at numerator AND denominator
Overdue CAPA % = DIVIDE([Overdue CAPAs] + 0, [Open CAPAs], 0)
```

The `+ 0` coerces the numerator: `BLANK + 0 = 0`, so the division evaluates to 0 instead of propagating BLANK. The third `DIVIDE` arg handles the zero-denominator case.

Apply this pattern to every count-based rate: defect %, rejection %, overdue %, compliance %, etc.

## Strict-filter measures that reference rare flag values

A measure like `CALCULATE(SUM(...), SomeColumn = "High")` returns BLANK if the data generator never emits `"High"` (or emits it very rarely). The card reads (Blank), the user asks "why", and the answer is "that value never exists in this data."

Before shipping any `column = "specific_value"` filter:
1. Verify the partition M actually assigns that value (grep the M for the string literal).
2. If the data is seeded deterministically and the rare category stays empty, either adjust the M threshold to guarantee some matches, or switch to a numeric threshold on a raw column (e.g. `DaysPastDue > 60` instead of `RiskTag = "High"`).

Rule: never build a KPI around a categorical column value that your data generator doesn't guarantee to produce.
