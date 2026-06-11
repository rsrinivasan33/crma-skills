# CRMA Binding Skill

Reference guide for implementing bindings in CRM Analytics (CRMA) dashboards. Use this skill whenever the user asks about connecting widgets, creating dynamic filters, switching measures/groupings, or making dashboards interactive beyond basic faceting.

---

## Overview

**Bindings** enable interactions among dashboard components by connecting the selection or results of one step to drive another step's query or widget properties.

**Two types:**
- **Selection binding** — triggered when a user makes a selection in a widget. Evaluated each time.
- **Results binding** — driven by the query results of a step, not user interaction. Used for initial states, computed values, color coding, etc.

**When to use bindings vs. facets:**
- Use **facets** first — they're simpler and automatically filter widgets sharing the same dataset.
- Use **bindings** when you need cross-dataset filtering, dynamic measures/groupings, computed colors, initial filter states, or anything facets can't do.

---

## Syntax (Dashboard Designer)

Bindings are written inside double braces `{{ }}` embedded in the JSON step or widget definition.

**Basic reference pattern:**
```
<stepID>.<result|selection>
```

Examples:
```
mySourceStep.selection
mySourceStep.result
```

> Always use the step **ID**, not the label.

**Full binding pattern — nested functions:**
```
<serialization_function>( <manipulation_function>( <selection_function>(stepID.selection, ...) ) )
```

Example:
```
coalesce(cell(mySourceStep.selection, 0, "grouping"), "state").asString()
```

---

## Function Reference

### Data Selection Functions

These select data from a step's selection or results. They return a table with named columns and zero-indexed rows.

#### `cell(source, rowIndex, columnName)`
Returns a single scalar value from a specific row and column.

```json
cell(myStep.selection, 0, "stateName")
// Output: "CA"

cell(myStep.selection, 1, "Amount")
// Output: 200
```

#### `column(source, [columnNames...])`
Returns one or more columns as a 1D or 2D array.

```json
column(myStep.selection, ["stateName"])
// Output: ["CA", "TX", "OR", "AL"]

column(myStep.selection, [])
// Output: [["CA","TX","OR","AL"], ["100","200","300","400"]]
```

#### `row(source, [rowIndices...], [columnNames...])`
Returns one or more rows.

```json
row(myStep.selection, [0], ["Amount"])
// Output: ["100"]

row(myStep.selection, [0,2], [])
// Output: [["CA","100"], ["OR","300"]]

row(myStep.selection, [], ["stateName"])
// Output: [["CA"],["TX"],["OR"],["AL"]]
```

---

### Data Manipulation Functions

Transform data between selection and serialization.

#### `coalesce(source1, source2, ...)`
Returns the first non-null value. Essential for providing default values when no selection is made.

```json
coalesce(cell(step1.selection, 0, "column1"), "green")
// Returns "green" if no selection
```

#### `concat(source1, source2, ...)`
Joins arrays. Null sources are skipped. Sources must be the same shape.

```json
concat(["a", "b"], ["c", "d"])
// Output: ["a","b","c","d"]
```

#### `flatten(source)`
Flattens a 2D array to 1D.

```json
flatten([["CDG","SAN"], ["BLR","HND"]])
// Output: ["CDG","SAN","BLR","HND"]
```

#### `join(source, token)`
Converts an array to a single string using a separator.

```json
join(["a","b","c"], "+")
// Output: ["a+b+c"]
```

#### `slice(source, start, end)`
Returns a subset of a 1D array. Negative indices supported.

```json
slice(step.selection, -1, 0)
// Returns last selected row
```

#### `toArray(source1, source2, ...)`
Converts scalars to a 1D array, or 1D arrays to a 2D array. Commonly needed for compact-form measures, groups, and orders.

```json
toArray(cell(Opportunities.selection, 0, "Region"))
// Output: ["Americas"]

toArray("APAC")
// Output: ["APAC"]

toArray(cell(Opportunities.selection, 0, "Region"), "APAC")
// Output: ["Americas","APAC"]
```

#### `valueAt(source, index)`
Returns the scalar at the given index. Negative indices supported. Returns null if index doesn't exist.

```json
valueAt(cell(step.selection, 0, "column"), -1)
// Returns last selected value
```

---

### Data Serialization Functions

Convert data into the format expected by the target step. Every binding must end with one of these.

#### `.asString()`
Serializes a scalar, 1D, or 2D array as a string. Escapes double quotes.

```json
cell(myStep.selection, 0, "grouping").asString()
// Use in SAQL group, order, string comparisons, widget title/color

// Widget color coding example:
"numberColor": "{{cell(color_1.result, 0, \"color\").asString()}}"
```

#### `.asEquality(fieldName)`
Returns an equality or `in` filter condition for SAQL.
- Scalar input → `fieldName == "value"`
- 1D array → `fieldName in ["a","b"]`
- Null → `fieldName by all` (no filter)

```json
cell(myStep.selection, 1, "measure").asEquality("bar")
// Output: bar == 32

column(myStep.selection, ["grouping"]).asEquality("bar")
// Output: bar in ["first","second"]

// In a SAQL query:
q = filter q by {{cell(stepFoo.selection, 1, "measure").asEquality("bar")}};
```

#### `.asRange(fieldName)`
Returns an inclusive numeric range filter. Input must be 1D array with min as first element, max as second.
Null → `fieldName by all`.

```json
row(myStep.selection, [0], ["min","max"]).asRange("bar")
// Output: bar >= 19 && bar <= 32

// In SAQL:
q = filter q by {{row(stepFoo.selection, [0], ["min","max"]).asRange("bar")}};
```

#### `.asDateRange(fieldName)`
Returns a date range filter for SAQL. Inclusive. Null → `fieldName in all`.

Input formats:
- 1D array of 2 epoch numbers → `fieldName in [dateRange([yyyy,m,d], [yyyy,m,d])]`
- 1D array of 2 strings → `fieldName in ["1 month ago".."current day"]`
- 2D array where each sub-array has 2 elements → relative date format `["2 years ago".."1 year ahead"]`
- 2D array where each sub-array has 3 elements → `dateRange([yyyy,m,d],[yyyy,m,d])`

```json
row(stepFoo.selection, [0], ["min","max"]).asDateRange("date(year, month, day)")
// Output: date(year, month, day) in [dateRange([2002,3,19], [2010,8,12])]

// Open-ended relative date:
// If source has [null, "current day"] → date(year,month,day) in [.."current day"]
```

> Note: `date_to_epoch()` returns seconds, but date range bindings require **milliseconds**.

#### `.asGrouping()`
Returns a SAQL grouping string. Input must be scalar or 1D array. Null → `group by all`.

```json
cell(stepFoo.selection, 1, "grouping").asGrouping()
// Output: 'second'

column(stepFoo.selection, ["grouping"]).asGrouping()
// Output: ('first', 'second')
```

#### `.asOrder()`
Returns a SAQL order clause. Supports scalar, 1D, and 2D (tuple) arrays.

```json
cell(stepFoo.selection, 1, "order").asOrder()
// Output: 'second'

column(stepFoo.selection, ["order"]).asOrder()
// Output: ('first', 'second')

row(stepFoo.selection, [], ["order","direction"]).asOrder()
// Output: ('first' desc, 'second' asc)

// In SAQL:
q = order q by {{row(stepFoo.selection, [], ["order","direction"]).asOrder()}};
```

#### `.asProjection()`
Returns a `<expression> as '<alias>'` string for SAQL `foreach` statements. Input must be row/column data with expression and alias columns.

```json
row(stepFoo.selection, [0], ["expression","alias"]).asProjection()
// Output: first as 'foo'

row(stepFoo.selection, [], ["expression","alias"]).asProjection()
// Output: first as 'foo', second as 'bar'

// In SAQL:
q = foreach q generate {{row(stepFoo.selection, [], ["expression","alias"]).asProjection()}};
```

#### `.asObject()`
Passes data through with no serialization. Returns as an object (array of strings). Used in compact-form queries for measures, groups, filters, order.

```json
column(StaticMeasureNames.selection, ["value"]).asObject()
cell(static_1.selection, 0, "value").asObject()
```

---

## Use Case Patterns

### 1. Measure Switching (Toggle → Chart Measure)

Create a `staticflex` step with measure options. Each value is `["aggregation", "field"]`.

```json
"MeasuresController_1": {
  "type": "staticflex",
  "selectMode": "singlerequired",
  "values": [
    { "display": "Total Amount", "step_property": ["sum", "Amount"] },
    { "display": "Avg Amount", "step_property": ["avg", "Amount"] },
    { "display": "Count", "step_property": ["count", "*"] }
  ],
  "start": { "display": ["Total Amount"] }
}
```

Bind in the target step:
```json
"PieByProduct_2": {
  "query": {
    "measures": [
      "{{ cell(MeasuresController_1.selection, 0, \"step_property\").asObject() }}"
    ],
    "groups": ["Product"]
  }
}
```

**Critical:** Replace `columnMap` in the chart widget with an empty `columns` array for charts created Spring '18+:
```json
"chart_2": {
  "parameters": {
    "visualizationType": "pie",
    "step": "PieByProduct_2",
    "columns": []
  }
}
```

---

### 2. Group (Dimension) Switching

Create a `staticflex` step where each value is the field name.

```json
"GroupingsController_1": {
  "type": "staticflex",
  "selectMode": "singlerequired",
  "values": [
    { "display": "Country", "value": "Country" },
    { "display": "Product", "value": "Product" }
  ],
  "start": { "display": ["Product"] }
}
```

Bind in the target step:
```json
"groups": [
  "{{ cell(GroupingsController_1.selection, 0, \"value\").asString() }}"
]
```

Replace `columnMap` with `"columns": []` in the chart widget.

---

### 3. Cross-Dataset Filter Binding

When widgets use different datasets, faceting won't work. Use a binding to pass selection from one step into a filter on another.

```json
"Status_1": {
  "query": {
    "filters": [
      [
        "AccountId.Name",
        "{{column(AccountId_Name_1.selection, [\"AccountId.Name\"]).asObject()}}",
        "in"
      ]
    ]
  }
}
```

---

### 4. Filter Bindings (SAQL Form)

**Equality:**
```saql
q = filter q by {{cell(stepFoo.selection, 1, "measure").asEquality("bar")}};
// Produces: q = filter q by bar == 32;

q = filter q by {{column(stepFoo.selection, ["grouping"]).asEquality("bar")}};
// Produces: q = filter q by bar in ["first","second"];
```

**Inequality:**
```saql
q = filter q by bar > {{cell(stepFoo.selection, 1, "measure").asString()}};
// Produces: q = filter q by bar > 32;
```

**Matches:**
```saql
q = filter q by bar matches "{{cell(stepFoo.selection, 1, "grouping").asString()}}";
```

**Range:**
```saql
q = filter q by {{row(stepFoo.selection, [0], ["min","max"]).asRange("bar")}};
// Produces: q = filter q by bar >= 19 && bar <= 32;
```

**Date Range:**
```saql
q = filter q by {{row(stepFoo.selection, [0], ["min","max"]).asDateRange("date(year, month, day)")}};
// Produces: q = filter q by date(year, month, day) in [dateRange([2002,3,19], [2010,8,12])];
```

---

### 5. Dynamic Date Range from Static Step

Create a static step with relative date range values:

```json
"step_date_static_with_start": {
  "type": "staticflex",
  "selectMode": "singlerequired",
  "values": [
    { "display": "Last 5 Years", "value": [[["year",-5],["year",0]]] },
    { "display": "Last 6 Years", "value": [[["year",-6],["year",0]]] }
  ]
}
```

Apply in compact-form filter:
```json
"filters": [["CreatedDate", "{{selection(step_date_static_with_start)}}"]]
```

Apply in SAQL:
```saql
q = filter q by date('CloseDate_Year','CloseDate_Month','CloseDate_Day') in {{selection(step_date_static_with_start)}};
```

---

### 6. Limit and Offset Bindings

No serialization function needed — use `.asString()`.

```json
"StaticLimits": {
  "type": "staticflex",
  "values": [
    { "display": "5", "value": 5 },
    { "display": "10", "value": 10 }
  ]
}
```

```saql
q = limit q {{cell(StaticLimits.selection, 0, "value").asString()}};
q = offset q {{cell(StepOffset.selection, 0, "offset").asString()}};
```

---

### 7. Widget Property Bindings (Color Coding)

Compute a color in a step using SAQL `case when`, then bind to `numberColor` in a number widget.

SAQL in step:
```saql
q = foreach q generate count() as 'count',
  (case when count() < 25000 then "#EE0A50"
        when count() < 50000 then "#F8CE00"
        else "#0FD178" end) as 'color';
```

Widget binding:
```json
"numberColor": "{{cell(color_1.result, 0, \"color\").asString()}}"
```

---

### 8. Initial Filter Selection (Results Binding for Logged-In User)

Use a SOQL step to get user attributes, then bind the result to another step's `start` property.

```json
"UserData": {
  "type": "soql",
  "query": "SELECT Username, Country FROM User WHERE Name = '!{user.name}'"
}

"Billing_Country_2": {
  "type": "saql",
  "start": "{{cell(UserData.result, 0, \"Country\").asObject() }}"
}
```

> Only `staticflex`, `saql`, or `soql` steps support bindings for initial filter selections.

---

### 9. Nested Bindings

Use the result of one binding as the column name in another binding (creates deeper dependencies).

```saql
q = filter q by {{
  column(
    RevenueDynamicGrouping.selection,
    column(StaticGroupingNames.selection, ["value"])
  ).asEquality(
    cell(StaticGroupingNames.selection, 0, "value")
  )
}};
```

---

### 10. Map Type Toggle

Use `coalesce` to fall back to result when no selection is made yet:

```json
"groups": [
  "{{coalesce(cell(static_1.selection, 0, \"grouping\"), cell(static_1.result, 0, \"grouping\")).asString()}}"
]
```

Widget property:
```json
"map": "{{coalesce(cell(static_1.selection, 0, \"mapType\"), cell(static_1.result, 0, \"mapType\")).asString()}}"
```

---

### 11. Reference Line Binding

```json
"referenceLines": [
  {
    "color": "#9271E8",
    "value": "{{cell(all_1.result, 0, \"avg_Amount\").asString()}}",
    "label": "Avg: {{cell(all_1.result, 0, \"avg_Amount\").asString()}}"
  }
]
```

---

## SAQL-Form Queries: Dual-Definition Rule

When binding measures or groups in a **SAQL-form query step**, you must define the binding in **two places**:
1. Inside `pigql` (the SAQL string)
2. In the `measures` or `groups` field of the step

```json
"query": {
  "pigql": "... q = group q by {{column(StaticGroupings.selection, [\"value\"]).asGrouping()}}; ...",
  "measures": "{{column(StaticMeasures.selection, [\"cf\"]).asObject()}}",
  "groups": "{{column(StaticGroupings.selection, [\"value\"]).asObject()}}"
}
```

The `pigql` binding handles the SAQL execution. The `measures`/`groups` bindings handle chart remapping.

---

## Classic Designer Binding Operations

The classic designer uses a different syntax with `{{ }}` templates and these operations:

| Operation | Purpose | Example |
|-----------|---------|---------|
| `selection(stepID)` | Returns the current selection of a step | `{{ selection(step_group) }}` |
| `value(source)` | Extracts scalar from selector array; returns all if empty | `{{ value(selection(step_limit)) }}` |
| `single_quote(source)` | Converts array to single-quoted SAQL format | `{{ single_quote(value(selection(step_StageName))) }}` |
| `no_quote(source)` | Removes all quotes — used for sort direction keywords | `{{ no_quote(value(selection(step_order))) }}` |
| `field(source, fieldName)` | Extracts a named field from each object in an array | `{{ value(field(selection(step_measure), 'proj')) }}` |

Classic designer example — dynamic group and filter in SAQL:
```saql
q = filter q by 'Account-Name' in {{ selection(step_Account_Owner_Name_2) }};
q = group q by {{ single_quote(value(selection(step_StageName_3))) }};
q = foreach q generate {{ single_quote(value(selection(step_StageName_3))) }} as {{ value(selection(step_StageName_3)) }}, sum('Amount') as 'sum_Amount';
```

Classic designer compact-form:
```json
"query": {
  "groups": "{{ selection(step_group) }}",
  "measures": "{{ selection(step_measure) }}",
  "limit": "{{ value(selection(step_limit)) }}"
}
```

---

## Limitations

- Widget property bindings work **only on chart and number widgets**.
- Unsupported number widget properties: `borderEdges`, `borderWidth`, `compact`, `exploreLink`, `measureField`, `textAlignment`.
- Unsupported chart widget properties: `borderEdges`, `borderRadius`, `borderWidth`, `exploreLink`, `measureField`.
- Bindings on widget **titles and subtitles are ignored** when the widget is opened in Explorer.
- For `grain`-type (values table) steps, you must use the **classic designer binding syntax**, not the dashboard designer syntax.
- Classic designer bindings can only reference **grouping columns** (not full rows).

---

## Error Types

**Validation errors** — wrong syntax, illegal arguments, or unescaped double quotes inside a binding string.

Common gotcha — always escape inner double quotes:
```json
"numberColor": "{{cell(color_1.result, 0, \"color\").asString()}}"
```

**Execution errors** — wrong data shape at runtime (e.g., binding expected a cell but received a row, or expected row index 3 but only 2 rows exist).

Use `coalesce()` to guard against null values:
```json
coalesce(cell(step1.selection, 0, "column1"), "defaultValue").asString()
```
