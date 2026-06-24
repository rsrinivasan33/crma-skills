# CRMA Recipe Development Skill

Build, deploy, update, and run CRMA Data Prep Recipes. This skill captures the non-obvious traps and workarounds discovered in OCA-Dev.

---

## When to invoke

- Create or update a CRMA recipe
- Run a recipe and monitor status
- Troubleshoot recipe deployment or execution errors

---

## File format

Recipes (`.wdpr`) are **plain JSON** — NOT the `["...", null]` wrapped format used by `.wdf` dataflows. Always use `"version": "67.0"`.

The `ui` block is **mandatory**. Recipes missing it will throw an error when opened in the CRMA UI.

---

## Do NOT use SFDX metadata deploy for recipes

`sf project deploy start --metadata "WaveRecipe:..."` fails with an opaque "unexpected error" in OCA-Dev sandbox. **Use the Analytics REST API instead** — it is the only reliable path.

---

## Creating a recipe via REST API

The POST body must be **multipart** with the recipe definition nested inside the `recipe` JSON part as `recipeDefinition`. Do NOT send it as a separate file part, and do NOT include `dataflowId` or `targetDataflowId` in the POST body.

```python
import json, http.client, uuid

with open("recipe.wdpr") as f:
    recipe_content = json.load(f)

metadata = json.dumps({
    "label": "My Recipe",
    "name": "My_Recipe",
    "format": "R3",
    "recipeDefinition": recipe_content   # nested dict, not a string
})

boundary = uuid.uuid4().hex
body = (
    f"--{boundary}\r\n"
    f'Content-Disposition: form-data; name="recipe"\r\n'
    f"Content-Type: application/json\r\n\r\n"
    f"{metadata}\r\n"
    f"--{boundary}--\r\n"
).encode("utf-8")

conn = http.client.HTTPSConnection(instance_host)
conn.request("POST", "/services/data/v66.0/wave/recipes", body=body,
    headers={
        "Authorization": f"Bearer {access_token}",
        "Content-Type": f"multipart/form-data; boundary={boundary}"
    }
)
```

---

## Updating a recipe via REST API

Use `http.client` with `"PATCH"` — `urllib.request` does not support PATCH natively and will silently fail or error. Same multipart body format as CREATE.

```python
conn = http.client.HTTPSConnection(instance_host)
conn.request("PATCH", f"/services/data/v66.0/wave/recipes/{recipe_id}",
    body=body, headers={...})
```

---

## Running a recipe

Recipes run via their backing `targetDataflowId` — get it from the recipe list:

```bash
# Get targetDataflowId
GET /services/data/v66.0/wave/recipes  → find recipe → read "targetDataflowId"

# Trigger run
sf analytics dataflow start --dataflowid <targetDataflowId> --target-org <alias>
```

Poll job status via:
```
GET /services/data/v66.0/wave/dataflowjobs/{jobId}
```
Fields: `status` (Queued → Running → Success/Failed), `progress` (0.0–1.0), `duration` (seconds).

---

## Data Sync prerequisite

Recipes use the `SFDC_LOCAL` connector which requires objects to have an existing edgemart. If a run fails with:

> `The '<Object>' object for connector SFDC_LOCAL doesn't have replicated edgemart`

The object is registered for sync (`replicated: true`) but has never actually been synced. **There is no API to trigger Data Sync** — it must be done from the UI:

CRMA UI → Data Manager → Data Sync → find the object → Run Now

Do this for every object in the recipe. Each object may need to be synced separately — the error surfaces one at a time per run attempt.

---

## Getting credentials

```bash
sf org display --target-org OCA-Dev 2>&1 | grep -E "(Access Token|Instance Url)"
```

---

## Node naming conventions

Always follow these conventions for recipe node names and UI labels:

**Node keys (IDs)** — PascalCase, no spaces or underscores. Used in `sources` references so must be valid identifiers:
- `LoadDatasetCase`, `LoadDatasetAccount`
- `JoinCaseAccount`
- `LoadAccountCase`

**UI labels (canvas display)** — same words as the node key but with spaces:
- `"Load Dataset Case"`, `"Load Dataset Account"`
- `"Join Case Account"`
- `"Load Account Case"`

**Naming rules:**
- **LOAD nodes:** `LoadDataset<ObjectName>` — e.g. `LoadDatasetCase`, `LoadDatasetAccount`
- **JOIN nodes:** `Join<Child><Parent>` — child object first, parent second — e.g. `JoinCaseAccount` (Case is the child, Account is the parent via AccountId)
- **OUTPUT nodes:** `Load<Result>` — no "Output" prefix — e.g. `LoadAccountCase`
- **Default join type is always `LOOKUP`** — only use a different join type if there is an explicit reason

---

## Error reference

| Error | Cause | Fix |
|---|---|---|
| `WaveRecipe deploy: unexpected error` | SFDX metadata deploy broken in this sandbox | Use REST API |
| `JSON_PARSER_ERROR: Unrecognized field "definition"` | Wrong field name | Use `recipeDefinition` |
| `JSON_PARSER_ERROR: Unrecognized field "dataflowId"` | Can't set dataflow on create | Omit it — platform creates it automatically |
| `POST_BODY_PARSE_ERROR: Binary data expected with name "recipeFile"` | Wrong multipart part name | Use `recipe` part with `recipeDefinition` nested inside |
| `503: doesn't have replicated edgemart` | Object not yet synced | Run Data Sync in CRMA UI for that object |
| Recipe opens with error in UI | Missing `ui` block | Always include `ui` with nodes, connectors, hiddenColumns |
