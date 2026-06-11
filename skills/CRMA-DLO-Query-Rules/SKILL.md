# CRMA Data Cloud Query Rules

Addendum skill for building CRMA dashboards on **Data Cloud** data sources. Invoke this alongside `CRMA-dashboard-development` and `CRMA-dashboard-design` whenever the dashboard data source is any Data Cloud object:

- **Data Lake Object (DLO)** — suffix `.dll`
- **Data Model Object (DMO)** — suffix `.dmo`
- **Calculated Insight (CI)** — suffix `.ci`

All layout, design, widget, and color rules from the base skills still apply. This skill covers **only what differs** for Data Cloud-backed steps.

---

## Before You Start

1. **Retrieve a working Data Cloud reference dashboard from the org** before writing any step JSON. Data Cloud step schemas must be validated against a live reference — do not guess.
   - Use the REST API (not `sf project retrieve`) since Data Cloud dashboards are often in personal folders and won't retrieve via metadata API:
     ```bash
     # Get access token
     sf org display --target-org <alias>
     # Fetch dashboard state
     curl -s -H "Authorization: Bearer <token>" \
       "<instanceUrl>/services/data/v62.0/wave/dashboards/<dashboardId>"
     ```
   - Use **API version v62.0 or higher** — lower versions reject non-default dataspace queries with error `550`.

2. **Discover real field names** before building — never assume field names. Use the Data Cloud SQL endpoint:
   ```bash
   curl -s -X POST \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     "<instanceUrl>/services/data/v62.0/ssot/query" \
     -d '{"sql": "SELECT * FROM <ObjectName>__dll LIMIT 1"}'
   ```
   Replace `__dll` with `__dmo` or `__ci` as appropriate. The response `metadata` block lists every field with its type.

---

## Data Cloud Step Rules (Critical — these differ from standard CRMA)

### Rule 1 — Never use `useExternalFilters`
Standard CRMA steps use `"useExternalFilters": true` to receive filter broadcasts. **Data Cloud steps do not support this property** — its presence causes component errors. Omit it entirely from every Data Cloud step.

```json
// ✅ Correct for Data Cloud
{
  "type": "aggregateflex",
  "useGlobal": true
}

// ❌ Wrong — causes errors on Data Cloud
{
  "type": "aggregateflex",
  "useGlobal": true,
  "useExternalFilters": true
}
```

### Rule 2 — Never duplicate dimension fields in both `columns` and `groups`
When a field is used as a group dimension, it must **not** also appear as a raw column entry in `columns`. Data Cloud steps reject this duplication; standard CRMA datasets tolerate it.

```json
// ✅ Correct — dimension only in groups, aggregate only in columns
{
  "columns": [{"field": ["count", "*"], "name": "Interactions"}],
  "groups": ["TopicApiName__c"]
}

// ❌ Wrong — TopicApiName__c duplicated in both
{
  "columns": [
    {"field": ["count", "*"], "name": "Interactions"},
    {"field": "TopicApiName__c", "name": "TopicApiName__c"}
  ],
  "groups": ["TopicApiName__c"]
}
```

### Rule 3 — Date grouping requires array-of-arrays format
Date fields grouped by granularity (year, month, week, day) must use the `[["granularity", "FieldName"]]` array-of-arrays format — not bare string field names.

```json
// ✅ Correct — date granularity as array-of-arrays
{
  "groups": [["year", "QuestionTimestamp__c"], ["week", "QuestionTimestamp__c"]],
  "orders": [
    {"name": "year_QuestionTimestamp__c", "nulls": "last", "filters": [], "ascending": true},
    {"name": "week_QuestionTimestamp__c", "nulls": "last", "filters": [], "ascending": true}
  ]
}

// ❌ Wrong — bare string fails on Data Cloud date fields
{
  "groups": ["QuestionTimestamp__c"]
}
```

- Order entries for date-grouped steps must use the flattened name format: `"<granularity>_<FieldName>"` (e.g. `"year_QuestionTimestamp__c"`).

### Rule 4 — Ungrouped aggregations require `"groups": ["all"]`
KPI steps that aggregate across all rows with no dimension grouping must set `"groups": ["all"]` explicitly. An empty `"groups": []` does not work for Data Cloud aggregate steps.

```json
// ✅ Correct — explicit "all" for ungrouped totals
{
  "columns": [
    {"field": ["count", "*"], "name": "Total"},
    {"field": ["avg", "ResponseTimeSecs__c"], "name": "Avg_Response"}
  ],
  "groups": ["all"]
}

// ❌ Wrong — empty groups causes errors on Data Cloud KPI steps
{
  "columns": [...],
  "groups": []
}
```

### Rule 5 — Use `["count", "*"]` not `["count", "FieldName"]`
Count aggregates should use the wildcard `*` form. Counting a specific field name is unreliable on Data Cloud sources.

```json
// ✅ Correct
{"field": ["count", "*"], "name": "Total_Interactions"}

// ❌ Avoid
{"field": ["count", "InteractionId__c"], "name": "Total_Interactions"}
```

### Rule 6 — Detail table (row-level) steps: `groups` must be `[]`, no aggregates in columns
For raw row-level tables, use empty `"groups": []` and list only plain field references in `columns` (no aggregate arrays). This is the same as standard CRMA — confirming it works correctly on Data Cloud objects too.

```json
// ✅ Correct — plain fields, no groups
{
  "columns": [
    {"field": "QuestionTimestamp__c", "name": "QuestionTimestamp__c"},
    {"field": "UserQuestion__c", "name": "UserQuestion__c"},
    {"field": "AgentAnswer__c", "name": "AgentAnswer__c"}
  ],
  "groups": []
}
```

---

## Minimal Working Data Cloud Step Template

Use this as the base for any Data Cloud-backed step. Adjust `cdpObjects`, `name`, `columns`, and `groups` per chart.

```json
"step_name": {
  "broadcastFacet": true,
  "cdpObjects": ["<ObjectName>__dll"],
  "datasets": [],
  "dataspace": "default",
  "isGlobal": false,
  "query": {
    "aggregateFilters": [],
    "columnGroups": [],
    "columnTotals": [],
    "limit": 250,
    "orders": [],
    "rowTotals": [],
    "sourceFilters": {},
    "sources": [
      {
        "columns": [
          {"field": ["count", "*"], "name": "Total"}
        ],
        "filters": [],
        "groups": ["DimensionField__c"],
        "joins": [],
        "name": "<ObjectName>__dll"
      }
    ]
  },
  "receiveFacetSource": {"mode": "all", "steps": []},
  "selectMode": "single",
  "type": "aggregateflex",
  "useGlobal": true
}
```

Key properties:
- `cdpObjects`: array containing the Data Cloud object API name (e.g. `MyObject__dll`, `MyObject__dmo`, `MyInsight__ci`)
- `dataspace`: `"default"` for the standard Data Cloud dataspace
- `useGlobal`: `true` to participate in global filter broadcasting
- No `useExternalFilters` — omit entirely

---

## Pre-Deploy Validation Checklist

Before deploying any Data Cloud-backed dashboard, verify every step against this checklist:

- [ ] No step has `useExternalFilters` (any value)
- [ ] No dimension field appears in both `columns` and `groups` in the same source
- [ ] Date-grouped steps use `[["granularity", "FieldName"]]` array format, not bare strings
- [ ] Date-grouped steps have matching `orders` entries using `"granularity_FieldName"` naming
- [ ] KPI/ungrouped aggregate steps have `"groups": ["all"]`
- [ ] All count aggregates use `["count", "*"]`
- [ ] All field names verified against live schema (via `ssot/query SELECT * LIMIT 1`)