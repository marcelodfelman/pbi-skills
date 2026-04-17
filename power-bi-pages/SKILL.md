---
name: Power BI Pages
description: >
  Manage Power BI report pages and bookmarks -- add, remove, configure, and lay
  out pages in PBIR reports using pbi-cli. Invoke this skill whenever the user
  mentions "add page", "new page", "delete page", "page layout", "page size",
  "page background", "hide page", "show page", "drillthrough", "page order",
  "page visibility", "page settings", "page navigation", "bookmark", "create
  bookmark", "save bookmark", "delete bookmark", or wants to manage bookmarks
  that capture page-level state. Also invoke when the user asks about drillthrough
  configuration or pageBinding.
tools: pbi-cli
---

# Power BI Pages Skill

Manage pages in PBIR reports. Pages are folders inside `definition/pages/`
containing a `page.json` file and a `visuals/` directory. No Power BI Desktop
connection is needed.

## Listing and Inspecting Pages

```bash
# List all pages with display names, order, and visibility
pbi report list-pages

# Get full details of a specific page
pbi report get-page page_abc123
```

`get-page` returns:
- `name`, `display_name`, `ordinal` (sort order)
- `width`, `height` (canvas size in pixels)
- `display_option` (e.g. `"FitToPage"`)
- `visual_count` -- how many visuals on the page
- `is_hidden` -- whether the page is hidden in the navigation pane
- `page_type` -- `"Default"` or `"Drillthrough"`
- `filter_config` -- page-level filter configuration (if any)
- `visual_interactions` -- custom visual interaction rules (if any)
- `page_binding` -- drillthrough parameter definition (if drillthrough page)

## Adding Pages

```bash
# Add with display name (folder name auto-generated)
pbi report add-page --display-name "Executive Overview"

# Custom folder name and canvas size
pbi report add-page --display-name "Details" --name detail_page \
    --width 1920 --height 1080
```

Default canvas size is 1280x720 (standard 16:9). Common alternatives:
- 1920x1080 -- Full HD
- 1280x960 -- 4:3
- Custom dimensions for mobile or dashboard layouts

## Deleting Pages

```bash
# Delete a page and all its visuals
pbi report delete-page page_abc123
```

This removes the entire page folder including all visual subdirectories.

## Page Background

```bash
# Set a solid background colour
pbi report set-background page_abc123 --color "#F5F5F5"
```

## Page Visibility

Control whether a page appears in the report navigation pane:

```bash
# Hide a page (useful for drillthrough or tooltip pages)
pbi report set-visibility page_abc123 --hidden

# Show a hidden page
pbi report set-visibility page_abc123 --visible
```

## Bookmarks

Bookmarks capture page-level state (filters, visibility, scroll position).
They live in `definition/bookmarks/`:

```bash
# List all bookmarks in the report
pbi bookmarks list

# Get details of a specific bookmark
pbi bookmarks get "My Bookmark"

# Add a new bookmark
pbi bookmarks add "Executive View"

# Delete a bookmark
pbi bookmarks delete "Old Bookmark"

# Toggle bookmark visibility
pbi bookmarks set-visibility "Draft View" --hidden
```

## Drillthrough Pages

Drillthrough pages have a `pageBinding` field in `page.json` that defines the
drillthrough parameter. When you call `get-page` on a drillthrough page, the
`page_binding` field returns the full binding definition including parameter
name, bound filter, and field expression. Regular pages return `null`.

To create a drillthrough page, add a page and then configure it as drillthrough
in Power BI Desktop (PBIR drillthrough configuration is not yet supported via
CLI -- the CLI can read and report on drillthrough configuration).

## Workflow: Set Up Report Pages

```bash
# 1. Add pages in order
pbi report add-page --display-name "Overview" --name overview
pbi report add-page --display-name "Sales Detail" --name sales_detail
pbi report add-page --display-name "Regional Drillthrough" --name region_drill

# 2. Hide the drillthrough page from navigation
pbi report set-visibility region_drill --hidden

# 3. Set backgrounds
pbi report set-background overview --color "#FAFAFA"

# 4. Verify the setup
pbi report list-pages
```

## Path Resolution

Page commands inherit the report path from the parent `pbi report` group:

1. Explicit: `pbi report --path ./MyReport.Report list-pages`
2. Auto-detect: walks up from CWD looking for `*.Report/definition/`
3. From `.pbip`: finds sibling `.Report` folder from `.pbip` file

## JSON Output

```bash
pbi --json report list-pages
pbi --json report get-page page_abc123
pbi --json bookmarks list
```

## ⚠️ Drill-through: Desktop UI only — do NOT edit `page.json`

Adding a `"drillthrough"` property to `page.json` causes Power BI Desktop (March 2026) to reject the file:

> `Property 'drillthrough' has not been defined and the schema does not allow additional properties. Path 'drillthrough'`

The page schema (`page/2.1.0/schema.json`) does not expose drill-through configuration as a JSON property. Desktop manages drill-through internally in a format that is not publicly documented in the PBIR schema.

**Configure drill-through exclusively from Desktop UI:**
1. Open the `.pbip` in Power BI Desktop
2. Select the target page (e.g. "Customer 360")
3. Format pane → Page information → set as Drillthrough page
4. Drag the drill-through filter field (e.g. `DimCustomer[CustomerName]`) into the Drillthrough filters well
5. Save — Desktop writes the config in its own internal format

Do not attempt to:
- Add `"drillthrough": { ... }` to `page.json`
- Create hidden slicer visuals with `"isDrillthrough": true`
- Write any drill-through config by hand in PBIR files

This is a **Desktop UI-only operation** — no pbi-cli command or manual JSON editing can replicate it safely.

## Standard layout zones (1280×720 canvas)

| Zone        | x    | y   | w    | h   | Bottom |
|-------------|------|-----|------|-----|--------|
| Logo        | 30   | 2.6 | 99   | 48  | 50     |
| Slicers     | vary | 8   | 185  | 70  | 78     |
| KPI Cards   | vary | 84  | vary | 84  | 168    |
| Charts Row1 | vary | 175 | vary | 258 | 433    |
| Charts Row2 | vary | 438 | vary | 272 | 710    |

Rules:
- Minimum 7 px gap between zones. KPI cards bottom = 168; charts row1 must start at y ≥ 175.
- If a page has only one chart row, use y=175, h=535.
- Slicer x positions must align their right edge with the rightmost chart's right edge.
- Never use `drillthrough` property in page.json — Desktop UI only (schema 2.1.0 rejects it).
