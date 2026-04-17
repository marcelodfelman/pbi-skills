---
name: Power BI Partitions & Expressions
description: Manage Power BI table partitions, named expressions (M/Power Query data sources), and calendar table configuration using pbi-cli. Invoke this skill whenever the user mentions "partitions", "data sources", "M expressions", "Power Query", "incremental refresh", "named expressions", "connection parameters", or wants to configure how tables load data. For broader modeling tasks (measures, relationships, hierarchies), see power-bi-modeling instead.
tools: pbi-cli
---

# Power BI Partitions & Expressions Skill

Manage table partitions, named expressions (M queries), and calendar tables.

## Prerequisites

```bash
pipx install pbi-cli-tool
pbi-cli skills install
pbi connect
```

## Partitions

Partitions define how data is loaded into a table. Each table has at least one partition.

```bash
# List partitions in a table
pbi partition list --table Sales
pbi --json partition list --table Sales

# Create a partition with an M expression
pbi partition create "Sales_2024" --table Sales \
  --expression "let Source = Sql.Database(\"server\", \"db\"), Sales = Source{[Schema=\"dbo\",Item=\"Sales\"]}[Data], Filtered = Table.SelectRows(Sales, each [Year] = 2024) in Filtered" \
  --mode Import

# Create a partition with DirectQuery mode
pbi partition create "Sales_Live" --table Sales --mode DirectQuery

# Delete a partition
pbi partition delete "Sales_Old" --table Sales

# Refresh a specific partition
pbi partition refresh "Sales_2024" --table Sales
```

## Named Expressions

Named expressions are shared M/Power Query definitions used as data sources or reusable query logic.

```bash
# List all named expressions
pbi expression list
pbi --json expression list

# Get a specific expression
pbi expression get "ServerURL"
pbi --json expression get "ServerURL"

# Create a named expression (M query)
pbi expression create "ServerURL" \
  --expression '"https://api.example.com/data"' \
  --description "API endpoint for data refresh"

# Create a parameterized data source
pbi expression create "DatabaseServer" \
  --expression '"sqlserver.company.com"' \
  --description "Production database server name"

# Delete a named expression
pbi expression delete "OldSource"
```

## Calendar Tables

Calendar/date tables enable time intelligence in DAX. Mark a table as a date table to unlock functions like TOTALYTD, SAMEPERIODLASTYEAR, etc.

```bash
# List all calendar/date tables
pbi calendar list
pbi --json calendar list

# Mark a table as a calendar table
pbi calendar mark Calendar --date-column Date

# Alternative: use the table command
pbi table mark-date Calendar --date-column Date
```

## Workflow: Set Up Partitioned Table

```bash
# 1. Create a table
pbi table create Sales --mode Import

# 2. Create partitions for different date ranges
pbi partition create "Sales_2023" --table Sales \
  --expression "let Source = ... in Filtered2023" \
  --mode Import

pbi partition create "Sales_2024" --table Sales \
  --expression "let Source = ... in Filtered2024" \
  --mode Import

# 3. Refresh specific partitions
pbi partition refresh "Sales_2024" --table Sales

# 4. Verify partitions
pbi --json partition list --table Sales
```

## Workflow: Manage Data Sources

```bash
# 1. List current data source expressions
pbi --json expression list

# 2. Create shared connection parameters
pbi expression create "ServerName" \
  --expression '"prod-sql-01.company.com"' \
  --description "Production SQL Server"

pbi expression create "DatabaseName" \
  --expression '"SalesDB"' \
  --description "Production database"

# 3. Verify
pbi --json expression list
```

## Best Practices

- Use partitions for large tables to enable incremental refresh
- Refresh only the partitions that have new data (`pbi partition refresh`)
- Use named expressions for shared connection parameters (server names, URLs)
- Always mark calendar tables with `pbi calendar mark` for time intelligence
- Use `--json` output for scripted partition management
- Export model as TMDL to version-control partition definitions: `pbi database export-tmdl ./model/`

## ⚠️ M Expression Gotcha: `date - date` returns `Duration`, not a number

Subtracting two `date` values in M produces a `Duration` value, not a day count. Any subsequent numeric comparison fails at refresh time with:

> `Expression.Error: We cannot apply operator < to types Number and Duration. Details: Left=30, Right=45.00:00:00`

This breaks typical aging-bucket logic and day-count derivations. **Always wrap date subtraction with `Duration.Days(...)`** to get an `Int64` day count you can compare numerically:

```m
// WRONG — returns Duration, comparisons fail
daysOutstanding = Date.From(AsOfDate) - Date.From(invoiceDate),
agingBucket = if daysOutstanding <= 30 then "0-30" else "31+"

// CORRECT — returns Int64 day count
daysOutstanding = Duration.Days(AsOfDate - invoiceDate),
agingBucket = if daysOutstanding <= 30 then "0-30" else "31+"
```

Notes:
- `Date.From(x) - Date.From(y)` and plain `x - y` (both dates) return the same `Duration` — wrapping in `Date.From` does not coerce to number.
- `Duration.Days(d)` returns an integer. `Duration.TotalDays(d)` returns a float (matters only when subtracting `datetime` with sub-day precision).
- The error surfaces only at **refresh time in Desktop**, not at M expression validation. Offline TMDL schema checks do not catch it. Always refresh once after generating synthetic-data M to verify.
- Affects any aging / elapsed-days logic: AR aging, inventory days-on-hand, time-to-fill, lead time, traceability windows.

## `Number.RandomBetween` is volatile — bind once, clamp defensively

In synthetic-data M partitions, `Number.RandomBetween(min, max)` can **re-evaluate on every reference**, even inside a `let` binding. The engine treats it as non-deterministic and does not memoize across reads. This silently breaks any logic that uses the same variable in both a branch condition AND an arithmetic expression:

```m
// WRONG — r5 can resample between the `if` and the `*`
let
    r5 = Number.RandomBetween(0, 1),
    DiscountPct = if r5 < 0.3 then 0.05 + r5 * 0.5 else 0,
    //                              ^^  this r5 may be a DIFFERENT draw than the one in the if
    NetUnitPrice = UnitList * (1 - DiscountPct),
    NetAmountILS = Quantity * NetUnitPrice
```

The reader thinks `DiscountPct` is bounded at `0.05 + 0.3*0.5 = 0.20` (20%), so `NetUnitPrice` stays positive. But if the second `r5` resamples and lands at, say, `1.9`, then `DiscountPct = 0.05 + 1.9*0.5 = 1.00` → `NetUnitPrice <= 0` → **`NetAmountILS` goes negative** for a small fraction of rows. Users see sporadic negative Net Sales / Net Amount rows and cannot reproduce them from a fixed seed.

**Fix pattern — dedicated draw variable + `List.Max({..., 0})` safety clamp:**

```m
// CORRECT
let
    DiscountDraw = Number.RandomBetween(0, 1),       // named draw, single conceptual sample
    DiscountPct  = if DiscountDraw < 0.3 then 0.05 + DiscountDraw * 0.5 else 0,
    NetUnitPrice = UnitList * (1 - DiscountPct),
    NetAmountILS = List.Max({Quantity * NetUnitPrice, 0})   // belt-and-braces clamp
```

> ⚠️ **`Number.Max(a, b)` does NOT exist in M** for two scalars — it only accepts a list. Writing `Number.Max(x, 0)` parses at TMDL-load time but **throws at refresh** (`Expression.Error: The name 'Number.Max' was not matched with a function of 2 arguments`). Use `List.Max({a, b})` or `if a < b then b else a`. Same rule for `Number.Min` — use `List.Min({a, b})`.

Two rules:
1. **Bind once per conceptual draw.** Use a distinct `XxxDraw` / `XxxRand` name for each random sample. Never reuse `r1..r9` across a condition *and* arithmetic — even if the naming suggests it's "the same draw."
2. **Clamp any derived quantity that must be non-negative.** Wrap with `Number.Max(expr, 0)`. This catches both the resampling bug and legitimate edge cases (e.g., subtracting aging buckets where the residual could go slightly negative).

### Where to audit

Any `Fact*.tmdl` partition that generates synthetic data. The vulnerable pattern is any `rN = Number.RandomBetween(...)` referenced in both a branch condition and an arithmetic formula. Known-clean patterns:
- `rN` used once only (inline or in a single expression).
- `rN` in condition that returns a boolean/flag (not a number fed to arithmetic).
- Already-clamped derived columns (`Number.Max(..., 0)` or explicit `if x < 0 then 0 else x`).

### Detection heuristic

Grep each `Fact*.tmdl` for `Number.RandomBetween`, then check every `rN =` for dual use. Fixed across this project in FactSales, FactAR, FactPurchases, FactShipment, FactInventory.
