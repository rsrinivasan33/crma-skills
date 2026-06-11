# CRMA Dashboard Design Skill

Apply this UX design methodology for all CRM Analytics (CRMA) dashboard work. These are non-negotiable design standards — follow them precisely.

---

## Dashboard Properties

| Property | Value |
|---|---|
| Columns | 48 |
| Row Height | Fine |
| Cell Spacing (Horizontal) | 0 |
| Cell Spacing (Vertical) | 0 |

---

## Layout Structure (Top to Bottom)

1. **Header** — Logo (left) + Dashboard Title (right)
2. **Filters** — List selectors, below the header
3. **KPIs** — Below filters
4. **Visualizations** — Below KPIs, max 3 per row (left to right)
5. **Detail Table** — Mandatory at the bottom

Use a **2-row gap** between every major section (filters → KPIs → visualizations → table). Exception: header → filters uses a 1-row gap.

---

## Component Placement

### Dashboard Title
```
"row": 0, "column": 5, "rowspan": 6, "colspan": 43
```
Use a **text widget** with font size **32px**, **center-aligned**, bold, white (`#FFFFFF`) text.

### Dashboard Logo
```
"row": 0, "column": 0, "rowspan": 6, "colspan": 5
```
Use an **image widget** (`"type": "image"`) — not a text widget. Reference a static resource by name:
```json
{"alignmentX": "center", "alignmentY": "center", "fit": "fitheight", "image": {"name": "<StaticResourceName>", "namespace": ""}, "interactions": []}
```

### Filters
- Default start: `"column": 1, "rowspan": 4, "colspan": 7`
- Each additional filter matches the same rowspan and colspan

### KPIs
- Use only a `number` widget — no separate container or label text widget
- Title is set via the `number` widget's built-in `"title"` property
- Recommended sizing: `"colspan": 4, "rowspan": 7`
- Start at `"column": 1` to align with the filter row
- Use a 1-column gap between KPIs: second KPI `"column"` = first KPI column + colspan + 1
- Set `"titleSize": 18`

---

## Filter Rules

- Use **list selector** widgets — not global filters
- **Multi-select** by default unless specified otherwise
- When multiple datasets are involved, only include fields **common across all datasets**
- Set **measure fields to none**
- Ensure every filter can be applied to all underlying datasets
- Use **Global Filters** only when a condition must apply across every dashboard component

---

## Visualization Rules

- **Never use the built-in visualization title** — always use a **Text widget** for the title
- Group each Text widget title + visualization together in a **Container**
- Maximum **3 visualizations per row**, left to right
- Use the **Date Widget** for date-based inputs unless specified otherwise
- Always enable `"absoluteModeEnabled": true` and `"presetsEnabled": true` on date widgets

#### Visualization Sizing Standard
| Element | rowspan |
|---|---|
| Container | 30 |
| Title (text widget) | 3 |
| Chart | 27 |

- The chart `"row"` must start exactly where the title ends (title row + title rowspan) to avoid overlap

#### Visualization Column Layout
The goal is to avoid congestion while making effective use of horizontal space. These principles apply to visualizations only — filters and KPIs follow their own fixed standards.

- **1 column margin** on the left and right edges
- **1 column gap** between side-by-side visualizations
- Distribute remaining columns evenly across visualizations in the row
- The detail table can be placed alongside a visualization rather than always full-width

**Reference spans for common layouts (48-column grid):**

| Layout | colspan per widget |
|---|---|
| 2 side-by-side | Left: col=1, colspan=22 / Right: col=24, colspan=23 |
| 3 side-by-side | col=1, colspan=14 / col=16, colspan=15 / col=32, colspan=15 |
| Full-width | col=1, colspan=46 |

---

## Pages

- Use multiple pages when there are **more than 8 visualizations** (KPIs not counted)
- Group pages by logical, related content

---

## Detail Table

- Every dashboard must have a table at the bottom
- The table surfaces key critical fields providing detailed context behind the dashboard insights — it acts as the "drill-down" layer

---

## Widget Type Reference (API names — case-sensitive)

Use these exact `"type"` values in the `.wdash` JSON. The API rejects unknown or incorrectly-cased names.

| Widget | Correct `type` value |
|---|---|
| Text | `text` |
| Image (use for Logo in header) | `image` |
| Chart | `chart` |
| KPI / Number | `number` |
| Container | `container` |
| List selector (dimension filter) | `listselector` |
| Date selector | `dateselector` |
| Detail table | `table` |
| Pill/toggle selector | `pillbox` |
| Range selector | `rangeselector` |
| Filter panel (avoid — use listselector instead) | `filterpanel` |

**Common mistakes confirmed by deployment failures:**
- `listSelector` ❌ → `listselector` ✓ (all lowercase)
- `date` / `dateRange` ❌ → `dateselector` ✓
- `pivotTable` ❌ → `table` ✓

---

## Colors

- Use the **CRM Analytics (CRMA) native color palette** for all dashboard components
