# Consolidation Table (Fixed Groups) — User Guide

A Power BI custom visual that renders a two-level-header consolidation table with collapsible Department/Cost Element rows, fully customizable Total rows, and dynamic (data-driven) labels — all without writing or editing any code.

---

## 1. What this visual does

It recreates a finance-style "consolidation table" report — the kind normally built with a huge HTML measure in DAX — as a real Power BI visual, driven entirely by drag-and-drop fields and Format pane settings:

- A **two-row header**: a top row of group names (e.g. "Act vs DB", "YTD vs DB") each spanning several sub-columns, and a bottom row listing the individual metric names (e.g. Perf, FX, ONEOFF, RD_USO, Total).
- **Department rows** that expand/collapse to reveal **Cost Element** rows underneath them.
- Automatic **bold + red-for-negative** number formatting, matching typical finance report conventions.
- Up to **3 independent Total rows**, each summing a custom subset of departments, with a custom label.
- **Renaming** of any group or sub-column name — either to fixed text, or to a value that comes live from your data (e.g. the current reporting month).

No unpivoting of your source data is required — you drag your existing wide columns (one physical column per metric) directly into the visual.

---

## 2. Installing the visual

1. In Power BI Desktop, open the **Visualizations pane**.
2. Click the **"..."** (More options) button below the built-in visual icons.
3. Choose **"Import a visual from a file"**.
4. Select the `.pbiviz` file you were given.
5. The visual now appears as a new icon at the bottom of the Visualizations pane — drag it onto the report canvas like any other visual.

This is a personal/organizational custom visual (not published to AppSource), so every report that needs it must import the same `.pbiviz` file once. If your organization has an admin portal for "Organizational visuals", it can be added there so users don't need to import it manually each time.

---

## 3. Data pane fields

| Field bucket | What to put here | Required? |
|---|---|---|
| **Department (main row)** | The column that defines your outer/parent rows (e.g. `DEPARTMENT`) | Yes |
| **Cost Element (sub-row, optional)** | The column that defines the child rows nested under each Department (e.g. `COST_ELEMENT`) | No — leave empty for a flat, non-expandable table |
| **Group 1 - Columns** through **Group 6 - Columns** | Drop any number of numeric fields into each bucket. All fields in the same bucket are shown together under one group header. Use as many buckets as you need (1 to 6); leave the rest empty. | At least one bucket needs at least one field |
| **Dynamic Labels (e.g. CurMonth)** | Optional. Drop any field here (e.g. a "current month" column) whose value you want to reuse as a live label elsewhere — see §7. | No |

**Example** — to recreate a table with two groups, "Act vs DB" (Performance/FX/One-off/RD_USO/Total) and "YTD vs DB" (same five sub-metrics):
- Group 1 → `DB_PERFORMANCE`, `DB_FX`, `DB_ONEOFF`, `DB_RD_USO`, `ACT_VS_DB`
- Group 2 → `YTD_DB_PERFORMANCE`, `YTD_DB_FX`, `YTD_DB_ONEOFF`, `YTD_DB_RD_USO`, `ACT_VS_DB_YTD`

The order you drop fields into a bucket is the order they're displayed left to right. The order you fill the buckets (Group 1, then Group 2, etc.) is the order the groups appear.

---

## 4. Row behavior

- Departments are listed in the order they first appear in your data (not alphabetically).
- Click anywhere on a Department row's label (or the **+ / −** icon, if shown) to expand or collapse its Cost Element rows.
- The Department row itself always shows the **sum across all its Cost Elements** for every column — you don't need a separate "subtotal" row.
- If you don't drag anything into "Cost Element", each Department is shown as a single flat row with no expand/collapse behavior.

---

## 5. Format pane reference

Open the Format pane (paint-roller icon) after selecting the visual on the canvas. Six sections are available:

### 5.1 Colors (`dataPoint`)
| Setting | Default | Effect |
|---|---|---|
| Negative number color | `#E20000` (red) | Font color applied to any cell whose value is negative |
| Header background color | `#E20000` (red) | Background of both header rows |
| Department row background color | `#7F7F7F` (gray) — *display only, currently unused in cell backgrounds* | Reserved for future use |
| Total rows background color | `#F0F0F0` (light gray) | Background of every enabled Total row |

### 5.2 Table format (`tableFormat`)
| Setting | Default | Effect |
|---|---|---|
| Font size | 13 | Applies to the entire table |
| Groups expanded by default | On | Whether Department rows start expanded or collapsed when the report first loads |
| Gap width between groups (px) | 8 | Width of the white vertical divider drawn to the left of the first column of every group (both header and data rows) |
| Show +/- expand icon | On | Toggle the visibility of the +/− icon on Department rows. The row stays clickable either way. |
| Bold text for Department/Total rows | On | Whether Department and Total rows render in bold or regular weight |

### 5.3 Group names — top header (`groupLabels`)
Six text boxes: **Group 1 name** through **Group 6 name**. Leave any of them blank to fall back to the default "Group 1", "Group 2", etc. Type any text you want (e.g. "Act vs DB"), or a dynamic reference (see §7).

### 5.4 Column labels (`columnLabels`)
One text box, **Rename columns**, holding a `key=value` list separated by semicolons:
```
Sum of ACTUALS=Actuals;Sum of DB_PERFORMANCE=Performance
```
- The **key** must match the field's display name exactly as Power BI shows it (usually `Sum of <FieldName>`, unless you changed the default aggregation).
- The **value** is the text to display instead. It can also be a dynamic reference (see §7).
- Separate multiple pairs with `;` — no extra spaces are needed but are tolerated.

### 5.5 Total rows (`totalRows`)
Three independent blocks (Total row 1, 2, 3), each with:
| Setting | Effect |
|---|---|
| Enable Total row N | Turns this Total row on/off |
| Total row N label | The text shown in the row's left-hand cell |
| Total row N departments | Comma-separated list of Department names to sum into this row (e.g. `Bad Debt, Technology, CVM & NBA`) |

Matching is **case-insensitive** and ignores leading/trailing spaces, but the names must otherwise match your `DEPARTMENT` column's values exactly (spelling, punctuation, "&" vs "and", etc.).

**Example** — to reproduce "Total Direct Cost" and "Total Direct Cost (Exc Reg)" from the original report:
- Total row 1: label `Total Direct Cost`, departments `Bad Debt, Regulatory, Technology, CVM & NBA`
- Total row 2: label `Total Direct Cost (Exc Reg)`, departments `Bad Debt, Technology, CVM & NBA`

### 5.6 Description header cell (`descriptionHeader`)
Controls the single top-left header cell (the one above the row labels, spanning both header rows):
| Setting | Effect |
|---|---|
| Split into two lines | Off = a single line of text. On = two lines stacked inside the same cell, separated by a thin divider. |
| Line 1 text | The (only, or top) line of text. Default: "Description". |
| Line 2 text | The bottom line — only shown when "Split into two lines" is on. |
| Separator thickness between lines (px) | Thickness of the divider line between Line 1 and Line 2. Only visible when two lines is on. |

Both Line 1 and Line 2 accept dynamic references (see §7).

---

## 6. Negative numbers and formatting

Numbers are formatted with thousands separators and no decimals, matching the classic accounting convention: negative values are shown in parentheses and colored (e.g. `(1,234)` instead of `-1,234`), using the color set in **Negative number color**.

---

## 7. Dynamic labels — pulling a name live from your data

Sometimes you don't want a fixed label like "Actuals" — you want it to always show the current reporting month (or any other value that changes with your data), without editing the Format pane every time the data refreshes.

**Steps:**
1. Drag the field holding that value (e.g. `CurMonth`, a text or date column) into the **Dynamic Labels** bucket in the Data pane.
   - If Power BI asks you to choose a summarization for that field, pick **First** (or "Don't summarize") — you generally don't want it to be Summed or Counted.
2. Anywhere you can type a label — **group names**, **Rename columns** values, or **Description header** line text — write a dollar sign `$` followed by the field's name **exactly as it appears in the Data pane**:
   ```
   Sum of ACTUALS=$CurMonth
   ```
3. Whenever your underlying data changes (e.g. a new month loads), the label updates automatically — no manual edits needed.

If the name after `$` doesn't match any field currently in the Dynamic Labels bucket, the visual just displays the raw text (including the `$`) as a fallback, so a typo is easy to spot.

You can combine multiple dynamic references, each targeting a different sub-column or group:
```
Sum of ACTUALS=$CurMonth;Sum of PREV_ACTUALS=$PrevMonth
```

---

## 8. Troubleshooting

| Symptom | Likely cause |
|---|---|
| Visual shows the placeholder message about adding fields | Nothing dropped in Department, or no field in any Group 1–6 bucket |
| A group header shows "Group 3" instead of your custom name | The Group 3 name text box in the Format pane is empty, or you're looking at the wrong Group number |
| Renaming doesn't take effect | The key in "Rename columns" doesn't exactly match the field's displayed name (check capitalization, spaces, and whether it's prefixed with "Sum of ") |
| Dynamic label shows literally `$CurMonth` instead of a value | The field name after `$` doesn't match what's shown in the Data pane under Dynamic Labels — copy it exactly, or the field wasn't actually dropped into that bucket |
| A Total row doesn't appear | Its "Enable" toggle is off, or its department list doesn't match any value in your Department column |
| Expand/collapse doesn't do anything | No field was dropped into "Cost Element" — with no sub-rows, there's nothing to expand |

---

## 9. Quick reference — recreating the original finance table

| Original DAX table element | This visual's equivalent |
|---|---|
| Two-level colored header with grouped columns | Group 1–6 buckets + Group name text boxes |
| Department rows with click-to-expand Cost Elements | Category / Subcategory fields, always on by default |
| Red parentheses for negative numbers | Negative number color setting (automatic) |
| "Total Direct Cost" / "Total Direct Cost (Exc Reg)" rows | Total row 1 / Total row 2, each with its own department list |
| `Description<br>EGP'000` two-line header cell | Description header cell, "Split into two lines" on |
| White separators between column groups (`sec-gap`) | Gap width between groups setting |
| Dynamic month in the header (`MAX(YEARMO)`) | Dynamic Labels bucket + `$FieldName` syntax |
