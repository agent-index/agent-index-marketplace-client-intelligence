---
name: view-template
type: task
version: 1.0.0
collection: client-intelligence
description: Member-facing read task that displays a specific template's current structure and version history. Shows the field list with mandatory/optional flags, version metadata, and a summary of changes from the template changelog. Any member with read access to /shared/client-intelligence/templates/ can run it.
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

View Template displays one template in detail — the current field list, the version number, the description, and the history of changes from the collection-wide template changelog. Members run it to understand a template before creating a client from it. Admins run it to verify their edits or to inspect an existing template before editing.

Authority is filesystem-enforced via universal Reader access on `/shared/client-intelligence/templates/`. Any collection member can run this task.

### Inputs

- **`slug`** (required, interactive or argument) — the template slug to view. If the member said something like "show me the pharma template," parse the slug; otherwise ask: *"Which template? You can run `@ai:list-templates` to see all of them by slug."*
- **`include_versions`** (optional, boolean, default `false`) — if true, show the full version history including snapshot diffs. If false (default), show only the current version + a one-line summary of changelog entries.

### Outputs

A formatted view of the template surfaced to the caller. Sections:

- Header: slug + display name + current version + created/last-updated dates
- Description (if present)
- Field list (table)
- Version history (one-line summaries of changelog entries; full diffs if `include_versions: true`)
- Next-action suggestions

---

## Workflow

### Step 1: Pre-flight checks

- Confirm filesystem access: `aifs_auth_status`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt with: *"Viewing a template needs remote filesystem access. Run `@ai:member-bootstrap` if the connection is broken."*
- Confirm the collection is installed: `aifs_exists("/shared/client-intelligence/collection-state.json")`. If false, halt with the standard not-installed message.

### Step 2: Resolve the slug

If the slug came in as an argument, use it directly. If not, ask interactively. Validate the format (kebab-case: lowercase letters, digits, hyphens; non-empty; no leading or trailing hyphen).

### Step 3: Confirm the template exists

`aifs_exists("/shared/client-intelligence/templates/{slug}/template.json")`. If false, halt with: *"No template with slug `{slug}` exists. Run `@ai:list-templates` to see all templates by slug, or `@ai:create-template` to author a new one."*

### Step 4: Read the current template

`aifs_read("/shared/client-intelligence/templates/{slug}/template.json")`. Parse the JSON. If parsing fails, halt with: *"The template file at `{path}` exists but is unreadable. An admin can investigate via the filesystem directly or by re-creating the template."*

If `aifs_read` returns a permission-denied response, halt with: *"You don't have read access to `/shared/client-intelligence/templates/{slug}/`. This usually means the install is broken or the all-members grant was revoked. Contact your org admin."*

### Step 5: Read the changelog (filtered)

`aifs_read("/shared/client-intelligence/templates/_changelog.json")`. Parse. Filter entries to those whose `template_slug` field matches the requested slug.

If the changelog is unreadable, surface a warning and proceed without version history. Don't halt — the current state is the more important answer.

### Step 6: (Optional) Read version snapshots

If `include_versions` is true, `aifs_list("/shared/client-intelligence/templates/{slug}/versions/")`. For each `vN.json`, read it. Build a per-version summary noting the diff against the prior version (which fields were added/removed/changed at each version).

### Step 7: Surface the view

Render in this layout:

```
# {display_name} (`{slug}`)

Current version: v{current_version}
Created: {created_date}    Last updated: {last_updated_date}

{description, if present}

## Fields

| Field | Property | Description |
|---|---|---|
| company_name | mandatory | The official name of the client company or organization. |
| primary_contact_name | optional | Name of the primary contact at this client. |
| ... | ... | ... |

## Version history

{N} change(s) on record. Most recent first:

- v3, 2026-05-14: Field `therapeutic_area` added (optional). Migration: no-impact.
- v2, 2026-04-30: Field `phone` removed. Migration: migrate.
- v1, 2026-04-15: Template created with 5 fields.
```

If `include_versions: true`, append per-version detail blocks with field-level diffs.

End with next-action suggestions:

> *"Create a client from this template: `@ai:create-client` (and pick `{slug}`). Edit the template: `@ai:edit-template {slug}`. List all templates: `@ai:list-templates`."*

---

## Directives

### Behavior

This task is straightforward — a member asks "what's in template X?" and gets a clear, structured answer. Adapt verbosity to the caller's apparent expertise: a non-technical member benefits from a brief gloss on the mandatory/optional distinction; a technical member doesn't. The default rendering is mid-verbosity.

If the member's intent is broader ("compare all templates," "find templates with a `phone` field"), recognize that this task answers one template at a time. Suggest they run `@ai:list-templates` for the broad view and then run this task per-template.

### State Management

Not stateful. Each invocation reads fresh from the filesystem.

### Constraints

- **Never write anything.** Read-only by definition.
- **Never call permission-modifying ops.** Templates are universally readable; no permission queries needed.
- **Never modify or repair a template** even if the read surfaces malformed data. Repair is the admin's job via `@ai:edit-template`.
- **Never expose raw JSON to the member** unless they explicitly ask for it. Render the structured view instead.

### Edge Cases

- **Slug doesn't exist.** Step 3 catches it. Suggest `@ai:list-templates`.
- **`template.json` is unparseable.** Halt at Step 4 with an investigative pointer. This is unusual and indicates corruption.
- **`_changelog.json` is unreadable or absent.** Surface a one-line warning, proceed with current-state-only display.
- **`include_versions: true` but `versions/` directory is empty or missing.** Surface "version snapshots aren't available for this template" and proceed with the changelog-only history view.
- **The current template's `version` field doesn't match the highest `vN.json` in the versions directory.** Surface a note: *"The template metadata claims version {current_version}, but the highest snapshot found is v{highest_snapshot}. This drift was likely introduced by a partial edit. Recommend running `@ai:edit-template {slug}` to resync or contacting an admin."* Then show the current template as authoritative.
