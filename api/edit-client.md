---
name: edit-client
type: task
version: 1.0.0
collection: client-intelligence
description: Member-facing task to edit a client instance. Supports updating template-field values, adding member-defined extension fields, removing extension fields, and updating linked external data references. Cannot structurally modify template-defined fields (additions, removals, property changes) — those go through edit-template. Requires Edit permission on the instance folder; authority is filesystem-enforced via try-and-catch on the write.
stateful: false
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks:
    - view-client
external_dependencies:
  - Remote filesystem access (adapter contract 2.0.0 or higher)
reads_from: "/shared/client-intelligence/"
writes_to: "/shared/client-intelligence/instances/"
---

## About This Task

Edit Client modifies the data on an existing client instance. The member walks through the current field values, decides which to change, and can also add new extension fields specific to this client. The task only writes inside the instance's own folder — it never modifies the template (that's `edit-template`) and it never modifies the public-index entry (the public-index only changes on rename or soft-delete; rename is out of scope for V1).

Authority is Edit permission on `/shared/client-intelligence/instances/{slug}/`, which the filesystem enforces. If the caller lacks edit, the write returns permission-denied and the task surfaces the standard error message.

### Inputs

- **`slug`** (required, interactive or argument) — the instance slug to edit.
- **`edits`** (interactive) — list of operations:
  - `set_template_field_value` — change a template-defined field's value. Sub-inputs: `field_name`, `new_value`.
  - `set_extension_field_value` — change an existing extension field's value. Sub-inputs: `field_name`, `new_value`.
  - `add_extension_field` — add a new extension field. Sub-inputs: `field_name`, `value`.
  - `remove_extension_field` — remove an existing extension field by name.

### Outputs

- `/shared/client-intelligence/instances/{slug}/instance.json` — updated (revision-aware).
- `/shared/client-intelligence/instances/{slug}/changelog.json` — one entry per edit (revision-aware append).

Confirmation message surfaced to the member showing what changed.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. If false, halt with the not-installed message.

### Step 2: Resolve slug

Same resolution as `view-client`: accept slug or name; disambiguate via public-index if needed.

### Step 3: Read the public-index entry

`aifs_read("/shared/client-intelligence/public-index/instances/{slug}.json")`. Capture template_slug, template_version, status.

If `status == archived`, surface: *"Client `{slug}` is archived. Edits aren't allowed on archived clients. Unarchive first (run `@ai:delete-client {slug}` and pick soft-unarchive — feature not in V1; an admin can flip status via direct filesystem edit)."* Halt. (V1 does not ship a separate unarchive task; this is a known limitation noted in ROADMAP.)

### Step 4: Read the instance with revision capture

- `aifs_stat("/shared/client-intelligence/instances/{slug}/instance.json")` — capture the current revision.
- `aifs_read(...)` — load current data.

If `aifs_read` returns permission-denied, halt with: *"You don't have edit access to this client. Ask an existing grantee or admin to grant Edit via `@ai:grant-permission {slug} {your_email} edit`."*

Note: a successful read does NOT prove edit access. The filesystem may grant Read separately from Write. The write attempt in Step 9 is the real authorization check.

### Step 5: Read the current template

`aifs_read("/shared/client-intelligence/templates/{template_slug}/template.json")`. Capture the current field list with `mandatory`/`optional` properties.

If the read fails (template was deleted or renamed), surface a warning but continue — the instance can still be edited against its recorded template_version, but the guardrails against modifying template-defined fields are now informational only.

### Step 6: Show current state and collect edits

Render the current state as a reference view (similar to `view-client` but in a more editable presentation), then walk the member through the edits one at a time:

For each edit, ask:

1. *"What kind of edit? (set template field / set extension field / add extension field / remove extension field)"*
2. Sub-questions based on the edit type.

Validate as you go:

- `set_template_field_value`: the field must exist in the current template's `fields` list. If not, halt the edit and surface: *"`{field_name}` isn't a field on template `{template_slug}`. Did you mean to add an extension field?"*
- `set_extension_field_value`: the field must exist in the instance's `extension_fields` object.
- `add_extension_field`: the field name must not collide with a template field name (case-sensitive) and must not already exist as an extension field.
- `remove_extension_field`: the field must exist in `extension_fields`.

### Step 7: Show summary and confirm

Show a summary of all collected edits. Ask for explicit confirmation. Do not proceed without it.

### Step 8: Build the new payload and new changelog entries

Apply the edits to a copy of the current instance data:

- `set_template_field_value`: update `template_fields.{field_name}` to the new value.
- `set_extension_field_value`: update `extension_fields.{field_name}` to the new value.
- `add_extension_field`: insert `extension_fields.{field_name}` with the value.
- `remove_extension_field`: delete `extension_fields.{field_name}`.

Compose changelog entries (one per edit):

```json
{
  "id": "{next_id, incrementing per entry}",
  "timestamp": "{ISO_TIMESTAMP}",
  "actor": {"display_name": "{caller.display_name}", "member_hash": "{caller.member_hash}"},
  "event": "field_edited | field_added | field_removed",
  "details": {
    "field": "{field_name}",
    "scope": "template | extension",
    "from": <old_value_or_null>,
    "to": <new_value_or_null>
  }
}
```

### Step 9: Write the instance (revision-aware)

`aifs_write("/shared/client-intelligence/instances/{slug}/instance.json", <new_payload>, if_revision=<captured_revision>)`.

**Branch on response:**

- **Success.** Continue to Step 10.
- **`REVISION_CONFLICT`.** Another writer edited the instance between Step 4 and now. Halt with: *"This client was edited by someone else while you were editing. Re-run `@ai:edit-client {slug}` to see the current state and start over."* No changelog entries are written.
- **Permission-denied.** Halt with: *"You don't have edit access to this client. Ask an existing grantee or admin to grant Edit via `@ai:grant-permission {slug} {your_email} edit`."*
- **Any other error.** Surface and halt.

### Step 10: Append changelog entries (revision-aware)

Read `instances/{slug}/changelog.json` with revision capture. Append the new entries. Bump `next_id`. Write back with `if_revision`. Retry up to 3 times on conflict.

If permission-denied (unlikely if Step 9 succeeded — same folder), halt with the standard edit-access error.

### Step 11: Confirm to member

```
Edits applied to `{name}` ({slug}):

- {edit_1 summary}
- {edit_2 summary}
- ...

{N} change(s) recorded in this client's changelog. View the updated client: @ai:view-client {slug}
```

---

## Directives

### Behavior

Conversational. Show the current state clearly before asking for edits so the member knows what's already there. Adapt verbosity to the member's expertise — a non-technical member benefits from gentle guidance on the template-field vs. extension-field distinction; an experienced member doesn't.

If the member tries to make a structural change to a template-defined field (rename, change mandatory/optional, remove), redirect: *"That's a template-level change. Run `@ai:edit-template {template_slug}` instead — and note that it affects every client created from this template, not just this one."*

### State Management

Not stateful from the agent's side. The interview is in-memory until Step 9. Partial state is possible if Step 9 succeeds but Step 10 fails — the instance has the new values but its changelog doesn't reflect them. This is a known partial-state shape; the member can re-run the task to detect that the instance values already match the intended edits and skip them (Step 6's diff check) while still appending the missing changelog entries.

### Constraints

- **Never modify the public-index entry** as part of an edit. The public-index changes only on rename (out of scope V1) or soft-delete.
- **Never modify template-defined field structure** — only their values. Refusal directs to `edit-template`.
- **Never call `aifs_share`, `aifs_unshare`, or `aifs_transfer_ownership`.** Edit doesn't change permissions.
- **Never pre-check edit permission via `aifs_get_permissions`.** Attempt the write; let the filesystem reject; translate.
- **Never edit a archived client.** Step 3 enforces.

### Edge Cases

- **Template was edited (via `edit-template` with Migrate) between Step 4 and Step 9.** The instance's `template_version` may not match the current template's version. The member's edits are still valid (they operate on the recorded field values), but the next time the instance is touched by Migrate, those edits could be overwritten. V1 accepts this; future versions may add a per-edit conflict-detection layer.
- **Template was deleted between Step 4 and Step 9.** The instance still exists. Step 5's read fails; the member can still edit values (the data is the source of truth), but the template-field-validity check in Step 6 has to be skipped. Surface the warning prominently.
- **Member adds an extension field, then in the same batch removes it.** Allowed. Both events go to the changelog. The net effect on the data is no change, but the audit trail records the operation.
- **Member tries to edit `name`, `slug`, `template_slug`, or other metadata fields.** These are not editable via this task. `name` and `slug` are immutable in V1 (rename is post-V1); `template_slug` is set at creation and only changed by migration. Surface: *"That field isn't directly editable. Names are immutable in V1; the template is set at creation."*
- **Revision conflict on the instance after one re-read.** Halt and recommend re-running.
- **Revision conflict on the changelog after 3 retries.** Halt with: *"The client's changelog is being modified concurrently. The instance edits succeeded but {N} changelog entries weren't appended. Re-run `@ai:edit-client {slug}` to retry, or accept the audit gap."*
