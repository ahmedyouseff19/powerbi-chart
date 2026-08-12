# Consolidation Table (Fixed Groups) — Custom Visual Documentation

This document explains, piece by piece, how the Power BI custom visual was built: what each file does, why the code is structured the way it is, and how the different Format-pane options connect to the rendered HTML table.

---

## 1. The big picture

A Power BI custom visual is just three things working together:

| File | Role |
|---|---|
| `capabilities.json` | Declares what fields can be dropped onto the visual (Data pane roles) and what formatting options appear in the Format pane. Power BI reads this file to build both panes automatically — you never draw the Data pane or Format pane yourself. |
| `src/visual.ts` | Runs every time the data or a format setting changes. It reads the `DataView` object Power BI hands it, and manually builds an HTML `<table>` using plain DOM calls (`document.createElement`, etc.). |
| `style/visual.less` | Plain CSS (compiled from LESS) that styles the table. Classes are added by `visual.ts`; the LESS file just defines what those classes look like. |

The visual has no "memory" of its own between renders — every time `update()` runs, it wipes the container (`this.target.innerHTML = ""`) and rebuilds the whole table from scratch based on the current data + current settings. The only thing that *does* persist across renders is `this.expandedState`, a plain JS object living on the class instance, which remembers which Department rows are expanded/collapsed.

---

## 2. `capabilities.json` — declaring the Data pane

### 2.1 Data roles

```json
"dataRoles": [
  { "displayName": "Department (main row)", "name": "category", "kind": "Grouping" },
  { "displayName": "Cost Element (sub-row, optional)", "name": "subcategory", "kind": "Grouping" },
  { "displayName": "Group 1 - Columns", "name": "group1", "kind": "Measure" },
  { "displayName": "Group 2 - Columns", "name": "group2", "kind": "Measure" }
]
```

- `category` and `subcategory` are `Grouping` roles — they become the **row hierarchy** (Department, then optionally Cost Element underneath it).
- `group1` .. `group6` are `Measure` roles. Each one is a "bucket" you can drop *multiple* numeric fields into. This is the trick that avoids needing to unpivot your source table: instead of one measure column that has to carry every metric, each bucket accepts a whole set of columns (e.g. `DB_PERFORMANCE`, `DB_FX`, `DB_ONEOFF`...) that belong together under one header group.

### 2.2 `dataViewMappings` — how those roles become an actual DataView

```json
"categorical": {
  "categories": {
    "select": [
      { "for": { "in": "category" } },
      { "for": { "in": "subcategory" } }
    ]
  },
  "values": {
    "select": [
      { "for": { "in": "group1" } }, { "for": { "in": "group2" } }
    ]
  }
}
```

- `categories.select` with two entries produces **two parallel arrays** (`categorical.categories[0]` and `[1]`), one row per unique (Department, Cost Element) combination — this is what lets `visual.ts` read `deptColumn.values[i]` and `ceColumn.values[i]` for the same row `i`.
- `values.select` listing all six group roles means `categorical.values` ends up as one **flat array of columns**, where every column still remembers *which role it came from* (via `column.source.roles`). That's the mechanism `visual.ts` uses to sort each dragged field back into its Group 1–6 bucket (see 3.4).

### 2.3 `objects` — the Format pane sections

Every top-level key under `objects` becomes a **section** in the Format pane; every key under `properties` becomes a **control** inside that section. The `type` decides which control renders:

| `type` | Control rendered |
|---|---|
| `{ "fill": { "solid": { "color": true } } }` | Color picker |
| `{ "text": true }` | Text input |
| `{ "bool": true }` | Toggle switch |
| `{ "numeric": true }` | Number input |
| `{ "formatting": { "fontSize": true } }` | Font-size input |

Five sections were defined:
- **`dataPoint`** — the four colors (negative numbers, header background, department-row background, total-row background).
- **`tableFormat`** — font size, default expand state, gap width between groups, show/hide the +/- icon, bold toggle for Department/Total rows.
- **`groupLabels`** — one text box per bucket (`group1Label` .. `group6Label`) so you can rename "Group 1" to "Act vs DB" etc.
- **`columnLabels`** — a single text box (`relabelMap`) holding a `key=value;key2=value2` string used to rename individual sub-columns without touching the source data.
- **`totalRows`** — three independent Total-row configs, each with an enable toggle, a label, and a comma-separated department list.
- **`descriptionHeader`** — controls for the top-left "Description" cell: one line vs two lines, the text of each line, and the separator thickness between them.

Nothing here draws the actual pane — Power BI's shell does that from this JSON. What `visual.ts` has to do is (a) **read back** whatever the user typed/toggled, and (b) **tell Power BI what to show** when the pane opens (via `enumerateObjectInstances`, see 3.7).

---

## 3. `src/visual.ts` — line-by-line walkthrough

### 3.1 Imports and constants

```ts
import "../style/visual.less";
import powerbi from "powerbi-visuals-api";
```
The LESS import is how webpack knows to bundle the stylesheet into the visual package. The `powerbi-visuals-api` import brings in TypeScript types only (no runtime code) — it's how `DataView`, `VisualUpdateOptions`, etc. get type-checked.

```ts
const GROUP_ROLES = ["group1", "group2", "group3", "group4", "group5", "group6"];
```
A single source of truth for the six role names, reused everywhere instead of typing the strings six times.

### 3.2 The settings model

```ts
interface ITotalRowConfig {
    enabled: boolean;
    label: string;
    departments: string[];
}
```
Represents one configurable Total row after parsing. `departments` is already split, trimmed, and lower-cased at this point (see `parseDeptList`, 3.3).

```ts
interface ISettings { ... }
function getDefaultSettings(): ISettings { ... }
```
`ISettings` is the visual's entire configuration in one flat object — every value the Format pane can influence, with the defaults the visual should use the very first time it renders (before the user has touched anything). Keeping every setting in one object, rebuilt fresh on every `update()` call, avoids any stale-state bugs.

### 3.3 Small pure helper functions

```ts
function parseRelabelMap(text: string): Record<string, string> {
    const map: Record<string, string> = {};
    if (!text) return map;
    text.split(";").forEach(pair => {
        const idx = pair.indexOf("=");
        if (idx === -1) return;
        const key = pair.substring(0, idx).trim();
        const val = pair.substring(idx + 1).trim();
        if (key) map[key] = val;
    });
    return map;
}
```
Turns the raw string `"DB_PERFORMANCE=Perf;DB_FX=FX"` into `{ DB_PERFORMANCE: "Perf", DB_FX: "FX" }`. Splitting on the *first* `=` (via `indexOf`, not `split("=")`) means a value that itself contains `=` wouldn't break the key.

```ts
function parseDeptList(text: string): string[] {
    if (!text) return [];
    return text.split(",").map(s => s.trim().toLowerCase()).filter(s => s.length > 0);
}
```
Turns `"Bad Debt, Technology, CVM & NBA"` into `["bad debt", "technology", "cvm & nba"]`. Lower-casing here (rather than at comparison time) means the comparison later is a simple `indexOf` instead of a case-insensitive comparison function.

```ts
function relabel(map: Record<string, string>, key: string): string {
    return map[key] !== undefined ? map[key] : key;
}
```
If the user renamed this key, use the new name; otherwise fall back to the original field name unchanged.

```ts
function formatNumber(value: number, decimals: number = 0): string {
    if (value === null || value === undefined || isNaN(value)) return "";
    const abs = Math.abs(value).toLocaleString("en-US", {
        minimumFractionDigits: decimals,
        maximumFractionDigits: decimals
    });
    return value < 0 ? `(${abs})` : abs;
}
```
Reproduces the original DAX `FORMAT(value, "#,##0;(#,##0)")` behaviour: thousands separator via `toLocaleString`, negative numbers shown as `(1,234)` instead of `-1,234`.

### 3.4 `IColumnInfo` and reading the DataView

```ts
interface IColumnInfo {
    column: DataViewValueColumn;
    groupIndex: number;
    subLabel: string;
    isGroupStart: boolean;
}
```
This is the visual's internal, already-processed representation of one rendered column — built once per `update()` call and then passed around to every row-building function so nothing has to re-derive it.

Inside `update()`:

```ts
const rawColumns: { column: DataViewValueColumn; groupIndex: number; subLabel: string }[] = [];
categorical.values.forEach((col: DataViewValueColumn) => {
    const roles = col.source.roles || {};
    let groupIndex = -1;
    for (let g = 0; g < GROUP_ROLES.length; g++) {
        if (roles[GROUP_ROLES[g]]) { groupIndex = g; break; }
    }
    if (groupIndex === -1) return;
    rawColumns.push({ column: col, groupIndex, subLabel: relabel(this.settings.relabelMap, col.source.displayName) });
});
```
This is the key trick that makes the "6 fixed buckets" design work without ever asking the user to concatenate text fields together: every value column Power BI hands back carries `source.roles`, a small object like `{ group2: true }`, telling us which capability role it was dragged into. Looping over `GROUP_ROLES` and checking each one tells us the bucket index (0–5) for that specific column.

```ts
const columns: IColumnInfo[] = rawColumns.map((c, idx) => ({
    ...c,
    isGroupStart: idx === 0 || rawColumns[idx - 1].groupIndex !== c.groupIndex
}));
```
`isGroupStart` is `true` for the first column of every new bucket — that flag is what later triggers the white "gap" border (`ct-group-gap` class) to visually separate groups, and it's computed once here instead of re-checking neighbours every time a cell is drawn.

### 3.5 Row iteration and aggregation

```ts
const deptOrder: string[] = [];
deptValues.forEach(d => { if (deptOrder.indexOf(d) === -1) deptOrder.push(d); });
```
Departments are rendered in **first-appearance order** in the data, not alphabetically — this keeps the table's row order predictable and matching whatever order the underlying query returns.

```ts
deptOrder.forEach(dept => {
    if (this.expandedState[dept] === undefined) {
        this.expandedState[dept] = this.settings.defaultExpanded;
    }
    const deptIndices = allIndices.filter(i => deptValues[i] === dept);
    const hasChildren = !!ceValues;

    const deptTotals = columns.map(c => this.sumIndices(c.column, deptIndices));
    tbody.appendChild(this.buildDataRow(dept, deptTotals, columns, true, hasChildren, dept));
```
For every unique Department, `deptIndices` is the list of row-indices in the underlying flat DataView that belong to it. `sumIndices` (3.5.1) adds up the values across *all* those indices for one column — this is what produces the Department-level aggregate row, even when there are several Cost Elements underneath contributing to it.

```ts
    if (hasChildren) {
        const ceOrder: string[] = [];
        deptIndices.forEach(i => { const c = ceValues![i]; if (ceOrder.indexOf(c) === -1) ceOrder.push(c); });
        ceOrder.forEach(ce => {
            const ceIndices = deptIndices.filter(i => ceValues![i] === ce);
            const ceTotals = columns.map(c => this.sumIndices(c.column, ceIndices));
            const row = this.buildDataRow(ce, ceTotals, columns, false, false, dept);
            row.dataset.parent = dept;
            row.style.display = this.expandedState[dept] ? "" : "none";
            tbody.appendChild(row);
        });
    }
});
```
Same logic one level down: unique Cost Elements *within this Department*, each one summed over its own (usually single) row index. `row.dataset.parent = dept` tags the `<tr>` with a plain HTML `data-parent` attribute — this is how the click handler later finds "all rows belonging to this Department" using a CSS attribute selector, with no need to keep a separate JS array of child elements.

```ts
private sumIndices(col: DataViewValueColumn, indices: number[]): number {
    let sum = 0;
    indices.forEach(i => {
        const v = col.values[i] as number;
        if (v !== null && v !== undefined && !isNaN(v)) sum += v;
    });
    return sum;
}
```
A defensive sum: blanks/nulls are treated as 0 and skipped rather than turning the whole sum into `NaN`.

```ts
this.settings.totalRows.forEach(tr => {
    if (!tr.enabled || tr.departments.length === 0) return;
    const matchedIndices = allIndices.filter(i => tr.departments.indexOf(deptValues[i].trim().toLowerCase()) !== -1);
    const totals = columns.map(c => this.sumIndices(c.column, matchedIndices));
    tbody.appendChild(this.buildTotalRow(tr.label, totals, columns));
});
```
Same aggregation pattern a third time, but the row-index filter comes from the user's typed department list instead of a single department name — this is what lets you build a "Total Direct Cost (excl. Regulatory)" row purely from Format-pane text, no code change required.

### 3.6 Building the HTML

**`buildDescriptionCell()`** — the top-left header cell:
```ts
if (this.settings.descTwoLines) {
    descTh.appendChild(document.createTextNode(this.settings.descLine1));
    const sep = document.createElement("div");
    sep.className = "ct-desc-separator";
    sep.style.borderTopWidth = `${this.settings.descSeparatorSize}px`;
    descTh.appendChild(sep);
    descTh.appendChild(document.createTextNode(this.settings.descLine2));
} else {
    descTh.textContent = this.settings.descLine1;
}
```
One `<td rowSpan="2">` either holds one text node, or holds text -> a thin `<div>` divider (its thickness set inline from the setting) -> a second text node — all still inside a single cell, so it never splits into two grid columns, only two visual lines.

**`buildTwoLevelHeader()`** — the two `<tr>` header rows:
```ts
let i = 0;
while (i < columns.length) {
    const gIdx = columns[i].groupIndex;
    let span = 1;
    while (i + span < columns.length && columns[i + span].groupIndex === gIdx) span++;
    const th = document.createElement("td");
    th.colSpan = span;
    if (i > 0) th.classList.add("ct-group-gap");
    th.textContent = label && label.length > 0 ? label : `Group ${gIdx + 1}`;
    topRow.appendChild(th);
    i += span;
}
```
This is a manual **run-length encoding** pass: it walks the ordered `columns` array and, for every consecutive run of columns sharing the same `groupIndex`, emits one `<td colspan="N">` for the whole run instead of one cell per column. That's what makes the group header (e.g. "Act vs DB") stretch across all its sub-columns automatically, however many columns you dropped into that bucket.

**`appendToggleAndLabel()`** — shared by every clickable row label:
```ts
if (this.settings.showToggleIcon) {
    const icon = document.createElement("span");
    icon.textContent = this.expandedState[parentKey] ? "－ " : "＋ ";
    cell.appendChild(icon);
    (cell as any)._toggleIcon = icon;
}
cell.style.cursor = "pointer";
cell.addEventListener("click", () => {
    this.expandedState[parentKey] = !this.expandedState[parentKey];
    const rows = this.target.querySelectorAll(`tr[data-parent="${CSS.escape(parentKey)}"]`);
    rows.forEach((r: HTMLElement) => { r.style.display = this.expandedState[parentKey] ? "" : "none"; });
    const icon = (cell as any)._toggleIcon as HTMLElement;
    if (icon) icon.textContent = this.expandedState[parentKey] ? "－ " : "＋ ";
});
```
The icon `<span>` is *optional* — created only if `showToggleIcon` is on — but the click handler is attached to the whole cell regardless, so the row stays expandable/collapsible even with the icon hidden. `CSS.escape(parentKey)` guards against a department name containing characters that would otherwise break the `[data-parent="..."]` selector (quotes, etc.). Storing the icon reference on the cell itself (`(cell as any)._toggleIcon`) avoids a second DOM query every time the icon's text needs updating after a click.

**`buildDataRow()` / `buildTotalRow()`** — render one `<tr>`:
```ts
values.forEach((v, idx) => {
    const cell = document.createElement("td");
    if (columns[idx].isGroupStart && idx > 0) cell.classList.add("ct-group-gap");
    if (v < 0) cell.style.color = this.settings.negativeColor;
    if (isParent) cell.style.fontWeight = this.settings.boldMainRows ? "bold" : "normal";
    cell.textContent = formatNumber(v, 0);
    row.appendChild(cell);
});
```
Every value cell independently decides three things from the pre-computed `columns` array and the settings object: whether it needs the group-gap border, whether it needs red text (negative), and whether it needs bold (parent rows only). Nothing here is hard-coded to a specific metric — it's driven entirely by data + settings.

### 3.7 Theming and settings round-trip

```ts
private applyThemeVars() {
    this.target.style.setProperty("--ct-header-color", this.settings.headerColor);
    this.target.style.setProperty("--ct-group-header-color", this.settings.groupHeaderColor);
    this.target.style.setProperty("--ct-total-row-color", this.settings.totalRowColor);
    this.target.style.setProperty("--ct-gap-width", `${this.settings.groupGapWidth}px`);
}
```
Rather than setting inline colors on dozens of individual cells, four CSS custom properties are set once on the root container; `visual.less` references them (`var(--ct-header-color)` etc.), so a color change only touches four lines of JS instead of walking the whole table.

```ts
private parseSettings(dataView: DataView): ISettings {
    const settings = getDefaultSettings();
    const objects = dataView && dataView.metadata && dataView.metadata.objects;
    if (!objects) return settings;
    ...
}
```
This is the **read** half of the settings round-trip: Power BI stores whatever the user changed in the Format pane inside `dataView.metadata.objects`, keyed by object/property name exactly as declared in `capabilities.json`. Every `if (typeof x === "...")` guard exists because these values arrive as loosely-typed `any` — defensive checks stop a missing/malformed property from crashing the render.

```ts
public enumerateObjectInstances(options: powerbi.EnumerateVisualObjectInstancesOptions): powerbi.VisualObjectInstanceEnumeration {
    ...
    if (objectName === "dataPoint") {
        instances.push({ objectName, properties: { negativeColor: { solid: { color: settings.negativeColor } } }, selector: null });
    }
    ...
}
```
This is the **write** half: whenever the user opens a Format pane section, Power BI calls this method asking "what should I show as the current value of each control in this section?" Returning `settings.xxx` (rather than the hard-coded defaults) is what makes the pane reflect whatever was actually applied, including on first load.

---

## 4. `style/visual.less` — how the classes render

```less
.ct-visual-root {
    --ct-header-color: #E20000;
    --ct-gap-width: 8px;
}
.ct-group-gap {
    border-left: var(--ct-gap-width) solid #ffffff !important;
}
```
The `--ct-gap-width` variable is declared with a fallback default on the root, then overridden at runtime by `applyThemeVars()`. `.ct-group-gap` is applied by `visual.ts` to the first cell of every group (in both header rows and every data/total row) — a single CSS rule, driven by one JS-controlled variable, reproduces the `sec-gap` white separators from the original DAX table.

```less
.ct-desc-separator {
    border-top-style: solid;
    border-top-color: rgba(255, 255, 255, 0.7);
    margin: 3px 0;
}
```
Only `border-top-width` is left unset here — it's set inline per-render from `descSeparatorSize`, so the LESS file only needs to own the *style* and *color* of the line, not its thickness.

---

## 5. How the pieces connect, end to end

1. User drags `DB_PERFORMANCE`, `DB_FX`, `DB_ONEOFF`, `DB_RD_USO`, `ACT_VS_DB` into **Group 1**, and `YTD_DB_PERFORMANCE`... into **Group 2**.
2. Power BI builds a `DataView` where `categorical.values` contains all ten columns, each tagged with `source.roles.group1` or `source.roles.group2`.
3. `update()` runs -> `rawColumns` sorts them back into buckets 0 and 1 by role -> `columns` adds the `isGroupStart` flag.
4. `buildTwoLevelHeader()` walks `columns` and emits two `<td colspan="5">` cells reading "Group 1" / "Group 2" (or whatever the user typed in `groupLabels`), with five sub-header cells underneath.
5. For every Department, `sumIndices()` aggregates each of the ten columns; `buildDataRow()` renders the row, coloring negatives and bolding as configured.
6. If any `totalRows` entry is enabled, the same aggregation runs once more over a user-chosen subset of departments, producing a fully custom Total row.
7. `applyThemeVars()` pushes the four colors and the gap width onto the container as CSS variables; `visual.less` picks them up automatically.

No part of this pipeline is hard-coded to a specific number of groups, a specific number of sub-columns, or specific department names — every one of those is either derived from what the user dragged into the Data pane, or read from a Format-pane setting at render time.
