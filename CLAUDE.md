# CLAUDE.md — ServiceNow Background Scripts

## Repository Overview

This repository is a collection of **ServiceNow Background Scripts** — standalone JavaScript scripts designed to be run directly in the ServiceNow platform via **System Definition > Scripts - Background** (or the equivalent background script runner). Each script is a self-contained utility targeting a specific operational or administrative task.

Scripts are written in **ServiceNow's server-side JavaScript** (Rhino engine / GlideScript), using the platform's built-in APIs (`GlideRecord`, `GlideSystem`, etc.).

---

## Repository Structure

```
Servicenow-Background-scripts/
├── SN-FindDuplicateCI        # Detect duplicate Configuration Items in CMDB
└── SN-TaskLogCollection      # Collect all execution traces for a given Task record
```

### File Naming Convention

- All scripts are prefixed with `SN-` (ServiceNow).
- **No file extension** — scripts are plain text files intended to be copy-pasted into the ServiceNow background script runner.
- Names use PascalCase after the prefix: `SN-ScriptName`.

---

## Scripts

### `SN-FindDuplicateCI`

Scans the `cmdb_ci` table and prints any Configuration Items that share the same `name` value, along with their `sys_id`.

**Usage:** Run as-is — no configuration required.

**Output format:**
```
Duplicate CI: <name> | Sys ID: <sys_id>
```

**Key tables:** `cmdb_ci`

---

### `SN-TaskLogCollection`

Aggregates all observable execution traces for a given Task record (Incident, RITM, SCTASK, Universal Request, etc.). Collects:

1. **Task metadata** — sys_id, table name
2. **Events** — from `sysevent` linked to the task's sys_id
3. **Emails** — from `sys_email` linked to the task's sys_id
4. **Flow executions** — from `sys_flow_context`
5. **Audit history** — from `sys_audit`
6. **Playbook executions** — from `sn_playbook_context` (guarded by a table existence check)
7. **Syslog entries** — from `syslog` (searched by sys_id in message text, limited to 50 rows)

**Configuration:** Set `TASK_NUMBER` at the top of the script before running:
```js
var TASK_NUMBER = 'INC0012345'; // or RITM, SCTASK, etc.
```

**Output format:** JSON printed via `gs.print(JSON.stringify(finalOutput, null, 2))`

**Key tables:** `task`, `sysevent`, `sys_email`, `sys_flow_context`, `sys_audit`, `sn_playbook_context`, `syslog`, `sys_db_object`

---

## Development Conventions

### Script Structure

- **Simple/flat scripts** (no wrapper): use when the script has no local variable scope concerns — e.g., `SN-FindDuplicateCI`.
- **IIFE wrapper** `(function() { ... })();`: use when the script has multiple sections and needs a clean local scope — e.g., `SN-TaskLogCollection`.
- No external modules, imports, or `require()` — the platform does not support them in background scripts.

### GlideRecord Patterns

Always follow the standard query pattern:
```js
var gr = new GlideRecord('table_name');
gr.addQuery('field', 'value');
gr.orderByDesc('sys_created_on');
gr.setLimit(50);           // always limit when scanning large/unbounded tables
gr.query();

while (gr.next()) {
    // access fields via gr.field_name.toString() or gr.getDisplayValue('field_name')
}
```

- Use `.toString()` on field values to avoid GlideElement object serialization issues.
- Use `.getDisplayValue('field')` for reference fields that need human-readable values.
- Use `.getUniqueValue()` for the record's sys_id.
- Use `.getTableName()` for the record's actual table name (important for task hierarchy).

### Optional Table Safety Check

Before querying a table that may not exist (e.g., from a plugin that may not be installed):
```js
var tableCheck = new GlideRecord('sys_db_object');
tableCheck.addQuery('name', 'table_name_here');
tableCheck.setLimit(1);
tableCheck.query();

if (tableCheck.next()) {
    // safe to query the table
}
```

### Output

- Use `gs.print('message')` for all output — this is the standard background script output mechanism.
- For structured/complex output, use `JSON.stringify(obj, null, 2)` and print with a delimiter:
  ```js
  gs.print('================ RESULT ================');
  gs.print(JSON.stringify(finalOutput, null, 2));
  ```
- Use emoji prefixes for status lines to aid readability: `✅`, `❌`, `⚠️`.
- Do **not** use `console.log()` — it does not work in ServiceNow server-side scripts.

### Configurable Parameters

- Place all user-configurable variables at the **very top** of the script, clearly marked with a comment.
- Use empty string defaults and instruct the user to fill them in before running:
  ```js
  var TASK_NUMBER = ''; // CHANGE THIS WITH THE TASK NUMBER WHICH YOU WANT TO FETCH THE LOGS FOR
  ```

### No Comments by Default

- Only add comments when the reason behind the code is non-obvious: workarounds, platform quirks, invariants.
- Section dividers (e.g., `// ----- 1. GET TASK -----`) are acceptable in longer IIFE scripts to aid navigation.

---

## Adding a New Script

1. Create a new file at the repo root named `SN-<ScriptName>` (no extension).
2. Write the script following the conventions above.
3. Test it in a ServiceNow background script runner before committing.
4. Use a clear, descriptive commit message including what the script does in the commit body.

---

## Key ServiceNow APIs Reference

| API | Purpose |
|---|---|
| `new GlideRecord('table')` | Open a table for query/update |
| `gr.addQuery('field', 'value')` | Add a WHERE condition |
| `gr.addQuery('field', 'CONTAINS', 'val')` | LIKE-style contains query |
| `gr.orderByDesc('field')` | Sort descending |
| `gr.setLimit(n)` | Limit result count |
| `gr.query()` | Execute the query |
| `gr.next()` | Advance to next row (returns false when done) |
| `gr.field.toString()` | Get field value as plain string |
| `gr.getDisplayValue('field')` | Get display/label value for reference fields |
| `gr.getUniqueValue()` | Get the record's sys_id |
| `gr.getTableName()` | Get actual table name (useful for task hierarchy) |
| `gs.print('msg')` | Print output in background script runner |

---

## Git Workflow

- Default branch: `main`
- Feature branches: `claude/<description>` for AI-driven contributions
- Commit messages: imperative tone, brief subject line, extended body describing purpose if needed
- No CI/CD pipeline — scripts are tested manually in a ServiceNow instance
