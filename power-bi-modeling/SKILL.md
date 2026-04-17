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
