# CRMA Dashboard Development Skill

Development guidelines for building CRM Analytics (CRMA) dashboards. Follow this methodology for every dashboard build.

---

## Before You Start — Required Steps

1. **Invoke the design skill** — Always invoke `CRMA-dashboard-design` alongside this skill before writing any `.wdash` JSON.
   - **New dashboard:** Invoke both skills upfront, no exceptions.
   - **Updating existing dashboard:** Assess whether the change is a drastic overwrite or a fine-tune. If it could accidentally overwrite existing formatting, ask the user first. Do not silently overwrite layout or style decisions.
   - **Data Cloud data source:** If the dashboard queries any Data Cloud object — Data Lake Object (`.dll`), Data Model Object (`.dmo`), or Calculated Insight (`.ci`) — also invoke `CRMA-DLO-Query-Rules`. Data Cloud-backed steps have different query structure rules that will cause component errors if ignored.

2. **Ask for a reference dashboard** — Before writing any JSON, ask: *"Is there an existing dashboard I can use as a reference?"* Read that file fully before building. It provides exact widget parameter formats, `widgetStyle` patterns, step structures, and layout conventions that eliminate schema guesswork.
   - **Verify the filename matches exactly** — after locating the file, confirm the filename matches the user-provided name precisely before reading it. Do not assume the first plausible match is correct (e.g. `Srini_Claude_Dashboard.wdash` ≠ `Srini_Test_Dashboard.wdash`).
   - **If the exact file is not found locally, retrieve it from the org** before proceeding — run `sf project retrieve start --metadata "WaveDashboard:<ApiName>"` to pull it down. Never substitute a similar-sounding file. If the API name is unknown, run `sf analytics dashboard list --target-org <alias> | grep -i "<label>"` to find it first.
   - **On every redo or restart of a dashboard task**, re-invoke both `CRMA-dashboard-development` and `CRMA-dashboard-design` skills from scratch — do not treat prior invocations as still active. Skipping this is the single most common cause of layout, filter type, and schema errors.

3. **Validate before deploying** — When writing `.wdash` JSON from scratch, compare your structure against the reference dashboard field by field before the first deploy. Don't rely on deploy errors to discover schema issues one at a time.

---

## Step 1 — Assess the Brief

**Always start by explicitly asking the user:**

> "Is this dashboard requirement a rough draft / initial idea, or is it finalized with the key components defined?"

Then proceed based on the answer:

- **Rough / idea stage** — Ask a few high-level questions about data availability, key metrics, and intended visuals. Then create a first draft to iterate from.
- **Finalized / detailed spec** — Work through one dashboard component at a time. Ask relevant questions for that component, build it, then move to the next.

If the user has provided all details upfront in one go, still ask this question — their answer determines how strictly to follow the component-by-component approach.

---

## Step 2 — Dataset Strategy

- Prefer pushing complex logic to the **underlying datasets** rather than the dashboard layer
- If a dimension, measure, or attribute calculation is getting complex in the dashboard, flag it: state what dataset enhancement is needed so it can be planned
- Ask the user whether a **placeholder** should be used while the dataset change is being made

---

## Step 3 — KPI & Calculations

- Use **Compare Table** as the primary mechanism for calculations and powering KPI metrics
- Where possible, combine multiple KPIs into a **single Compare Table**

---

## Step 4 — Data Sources per Page

- Each page should ideally draw from **2–3 different datasets** to balance richness and performance

---

## Step 5 — Visualization Approach (Priority Order)

Build visualizations using this hierarchy — exhaust earlier options before moving to later ones:

1. Native CRMA chart/widget types
2. Compare Table for calculated metrics
3. **SAQL-based visualizations** — last resort only; always ask the user for explicit approval before implementing

---

## Step 6 — Colors

Ask the user whether to follow:
- **Salesforce / CRMA native colors**, or
- **Client brand colors**

Apply the confirmed palette consistently across the dashboard. Specifically confirm:
- **Header background color** — used for the dashboard title bar
- **Visualization title background color** — should it match the header or use a different color?
- **Visualization title text color**
- **KPI background and text colors**

---

## Step 7 — Detail Table (Values Table)

To display row-level detail (not aggregated), use the `sources`-based aggregateflex query pattern — **not** measures/groups and **not** SAQL. This is the compact form supported natively by CRMA.

```json
{
  "type": "aggregateflex",
  "query": {
    "aggregateFilters": [],
    "columnGroups": [],
    "columnTotals": [],
    "limit": 2000,
    "orders": [],
    "rowTotals": [],
    "sourceFilters": {},
    "sources": [
      {
        "columns": [
          {"field": "FieldName", "name": "FieldName"}
        ],
        "filters": [],
        "groups": [],
        "joins": [],
        "name": "<dataset_name>"
      }
    ]
  },
  "sortable": true,
  "useExternalFilters": true
}
```

- Add each field to display as a `{"field": "...", "name": "..."}` entry under `columns`
- Set `"limit": 2000` for detail tables
- Set `"sortable": true` to allow column sorting

---

## Step 8 — Iterative Build

Build the dashboard iteratively — don't wait for full clarity before starting:
- If logic for a component isn't fully clear, build it with the best available understanding
- Clearly inform the user what assumptions were made and what the next steps or open questions are
- This keeps momentum while giving the user something concrete to react to
