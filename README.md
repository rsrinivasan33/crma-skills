# CRMA Dashboard — Claude Code Skills

Four Claude Code skills for building and maintaining CRM Analytics (CRMA) dashboards from within Claude Code.

---

## Skills

| Skill | Purpose |
|---|---|
| `CRMA-dashboard-development` | Core methodology for building and updating CRMA dashboard JSON (`.wdash`) |
| `CRMA-dashboard-design` | UX design standards — layout, colors, typography, widget sizing |
| `CRMA-binding` | Reference guide for bindings — dynamic filters, cross-dataset interactions, computed colors |
| `CRMA-DLO-Query-Rules` | Addendum for dashboards backed by Data Cloud objects (DLOs, DMOs, Calculated Insights) |

---

## Prerequisites

| Requirement | Details |
|---|---|
| Claude Code | CLI installed (`claude --version` to verify) |
| Salesforce CLI (`sf`) | v2.x or later (`sf --version` to verify) |
| CRMA-enabled org | Any org with CRM Analytics enabled |

---

## Install

```bash
git clone https://github.com/rsrinivasan33/crma-skills.git
cp -r crma-skills/skills/* ~/.claude/skills/
```

---

## How the Skills Work Together

Always invoke **both** `CRMA-dashboard-development` and `CRMA-dashboard-design` before any dashboard task — new builds, updates, and redos. They are designed to be used together.

```
CRMA-dashboard-design          ← design standards (layout, colors, typography)
        +
CRMA-dashboard-development     ← build methodology (JSON structure, deploy, validate)
        +
CRMA-binding                   ← add when building interactive or cross-dataset filters
        +
CRMA-DLO-Query-Rules           ← add when data source is a Data Cloud object
```

**CRMA-DLO-Query-Rules** is an addendum — invoke it alongside the base skills whenever your dashboard queries a Data Lake Object (`.dll`), Data Model Object (`.dmo`), or Calculated Insight (`.ci`). Data Cloud-backed steps have different query structure rules that cause component errors if the standard rules are applied.

---

## Usage

```
/CRMA-dashboard-development
/CRMA-dashboard-design
```

Claude will ask for a reference dashboard and walk through the build following the design standards.
