---
name: list-templates
type: task
version: 2.0.0
collection: client-intelligence
description: Member-facing read task that enumerates all templates available for client creation. Returns each template's slug, display name, current version, and field count. Any member with read access to /shared/client-intelligence/templates/ can run it.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter contract 2.0.0 or higher)
reads_from: "/shared/client-intelligence/templates/"
writes_to: null
---

## About This Task

List Templates is the discovery surface for templates. Members run it before creating a client to see what templates are available, what each template is for, and what fields each carries. It's a pure read — the task makes no writes.

Authority is filesystem-enforced. Every collection member has Reader access to `/shared/client-intelligence/templates/` by virtue of the install-time setup grant to the all-members Google Group; the task does not pre-check. If the caller somehow lacks read access (e.g., the install is broken or the all-members grant was revoked manually), `aifs_list` will return a permission-denied response and the task surfaces a clear error.

### Inputs

None required. Optional filter or sort hints can be inferred from the member's request:

- **`filter_by_name`** (optional, free-text) — name-prefix filter to narrow the list (e.g., "show me pharma templates").
- **`include_deprecated`** (optional, boolean, default `false`) — V1 has no template deprecation concept; this input is reserved for future versions and currently ignored.
- **`sort_by`** (optional) — `name` (default), `created`, or `last_updated`.

### Outputs

A formatted list of templates surfaced to the caller. For each template:

- Slug
- Display name
- Current version
- Field count + breakdown (e.g., "4 fields: 1 mandatory, 3 optional")
- Created date
- Last-updated date

If no templates exist, surface: *"No templates exist yet on this org. An admin can create one with `@ai:create-template`."*

---

## Workflow

### Step 1: Pre-flight checks

- Confirm filesystem access: `aifs_auth_status`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt with: *"Listing templates needs remote filesystem access. Run `@ai:member-bootstrap` if the connection is broken."*
- Confirm the collection is installed: `aifs_exists("/shared/client-intelligence/collection-state.json")`. If false, halt with: *"client-intelligence is not installed on this org. An org admin must install it first."*

### Step 2: Enumerate templates

- `aifs_list("/shared/client-intelligence/templates/")`. Filter the result to entries whose `type` is `directory`. The `_changelog.json` file is in this directory but excluded by the directory filter.
- If `aifs_list` returns a permission-denied response, halt with: *"You don't have read access to `/shared/client-intelligence/templates/`. This usually means the collection's installer didn't share the templates folder with the all-members group. Contact your org admin."*
- If the entries list is empty, surface the "no templates" message and exit.

### Step 3: Read each template

For each directory entry from Step 2, call `aifs_read` on `/shared/client-intelligence/templates/{slug}/template.json`. Parse the JSON. Collect:

- `slug`
- `name`
- `version`
- `fields` (count, and per-field count of mandatory vs. optional)
- `created`
- `last_updated`
- `description` (if present)

If a template directory exists but `template.json` is missing or unparseable, skip the entry and note it in a "skipped templates" section of the output. Do not halt — partial enumeration is better than no output.

### Step 4: Apply filters and sort

If `filter_by_name` was supplied, filter by case-insensitive prefix match on `slug` or `name`. If `sort_by` was supplied, sort accordingly (default: alphabetical by name).

### Step 5: Surface the list

Render the list in a compact table format. One row per template:

```
| Slug | Name | v | Fields | Created |
|---|---|---|---|---|
| example-client | Example Client | 1 | 4 (1 mandatory, 3 optional) | 2026-05-13 |
| pharma-client | Pharma Client | 3 | 6 (2 mandatory, 4 optional) | 2026-04-15 |
```

If a description was set on a template, include a brief excerpt below its row. If the list is long (>10), offer to summarize or page.

After the list, suggest next actions: *"To see a template's full field list, run `@ai:view-template {slug}`. To create a client from one, run `@ai:create-client`."*

If templates were skipped due to malformed files, surface the skipped count separately: *"Note: {N} template(s) had unreadable `template.json` files and were skipped. An admin can investigate via `@ai:view-template {slug}` or repair them."*

---

## Directives

### Behavior

This task is conversational but lightweight. The member typically just wants to see what's available; render the list and move on. If the member's intent suggests they want more depth — *"tell me about the pharma template"* — chain into `@ai:view-template` rather than dumping all fields here.

For non-technical members, accompany the list with a brief explanation if no templates are returned: *"Templates are like blueprints for clients — they define what information each client record carries. An admin authors them; you pick one when creating a client."*

### State Management

Not stateful. Each invocation reads fresh from the filesystem.

### Constraints

- **Never write anything.** This task is read-only by definition. If the workflow ever needs to update state, it's a different task.
- **Never call `aifs_get_permissions`, `aifs_share`, `aifs_unshare`, or `aifs_transfer_ownership`.** Templates are universally readable by collection members; permission queries aren't this task's job.
- **Do not enumerate `instances/`** as part of the same task — that's `list-clients`. Cross-collection visibility floor concerns don't apply here because templates have no visibility floor (they're universally readable).
- **Never display malformed template content** to the caller. If a template.json is unparseable, mention the skip but don't expose the raw error or contents.

### Edge Cases

- **Templates directory is empty.** Surface the "no templates yet" message with the suggestion to create one.
- **Templates directory has files that aren't template folders.** The `_changelog.json` is one such case; the directory filter in Step 2 excludes it. Any other stray files (e.g., a `.gitkeep`, a stray archive file) are also excluded.
- **A template folder lacks `template.json`.** Note in Step 3's skip list; don't fail the whole task.
- **A template's `versions/` directory is missing entirely.** Doesn't affect this task — list-templates doesn't read `versions/`. (`view-template` is the one that reads version history.)
- **Filesystem returns a transient error mid-enumeration.** Surface what was successfully listed and a note about the partial result. Recommend re-running.
