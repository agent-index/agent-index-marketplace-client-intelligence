---
name: edit-template
type: task
version: 2.0.0
collection: client-intelligence
description: Admin-only task to edit an existing client template. Supports field additions, removals, property changes (mandatory <-> optional), and renames. At each edit the admin chooses Migrate (apply the change to all existing instances created from this template) or No-impact (leave existing instances unchanged; the change applies only to instances created from this point forward). Writes a new version snapshot, updates the canonical template file, appends per-change entries to the collection-wide template changelog, and on the Migrate path walks every affected instance to apply the schema change with cross-referenced per-instance changelog entries.
stateful: false
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks:
    - view-template
external_dependencies:
  - Remote filesystem access (adapter contract 2.0.0 or higher)
reads_from: "/shared/client-intelligence/"
writes_to: "/shared/client-intelligence/"
---

## About This Task

Edit Template is the most complex day-to-day task in the collection because it touches three concerns simultaneously: the template definition itself, the immutable version history, and (on the Migrate path) every client instance created from this template. The task is admin-only — only members with Drive Editor access to `/shared/client-intelligence/templates/` can complete it — and authority is enforced by the filesystem, not by the task.

The two migration paths are mutually exclusive on a per-edit basis:

- **Migrate** — the template change is applied to every existing instance created from this template. New mandatory fields are seeded with a default value the admin provides at edit time. Removed fields are deleted from existing instance data. Renamed fields have their values copied to the new key and removed from the old. Property changes (mandatory <-> optional) require no instance-data changes. Each per-instance change appends a `field_*_by_migration` event to that instance's changelog with a reference to the template changelog entry that caused it.
- **No-impact** — the template change is recorded in the changelog but no existing instances are touched. The change applies only to instances created from this point forward. Existing instances continue to operate against the version of the template they were created from (a known kind of drift that this task does not auto-reconcile).

The admin's choice between Migrate and No-impact is required and is recorded in the template changelog entry alongside the change description.

### Inputs

- **`slug`** (required) — the template slug to edit. Validated via `aifs_exists` on `templates/{slug}/template.json`.
- **`edits`** (interactive) — list of per-field operations. Each operation is one of:
  - `add` — add a new field. Sub-inputs: `name`, `property` (mandatory or optional), optional `description`. If `property` is `mandatory` and the chosen migration path is Migrate, an additional input is required: the default value to seed existing instances with.
  - `remove` — remove an existing field by name.
  - `rename` — rename an existing field. Sub-inputs: old `name`, new `name`.
  - `change_property` — flip a field between mandatory and optional. Sub-inputs: field `name`, new `property`.
- **`migration_choice`** (required) — `migrate` or `no_impact`. Picked once for the entire batch of edits in this invocation.

### Outputs

- `/shared/client-intelligence/templates/{slug}/template.json` — updated to the new version (`version` bumped, `last_updated` set, `fields` reflects all edits).
- `/shared/client-intelligence/templates/{slug}/versions/v{N+1}.json` — immutable snapshot of the new version.
- `/shared/client-intelligence/templates/_changelog.json` — one new entry per field change in the batch, each carrying the chosen `migration` value.
- On Migrate path: `/shared/client-intelligence/instances/{instance_slug}/instance.json` updated for each affected instance, plus a `field_*_by_migration` entry appended to each affected instance's `changelog.json`.

A summary surfaced to the admin showing what was changed, the new version number, and (on Migrate) per-instance success/skip outcomes.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt with the standard not-authenticated message.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. If false, halt with the standard not-installed message.

### Step 2: Resolve the slug

Validate kebab-case format. Confirm `aifs_exists("/shared/client-intelligence/templates/{slug}/template.json")`. If false, halt with: *"No template with slug `{slug}` exists. Run `@ai:list-templates` to see all templates by slug."*

### Step 3: Load current template

`aifs_stat("/shared/client-intelligence/templates/{slug}/template.json")` — capture the current revision (used for the revision-aware write in Step 9).

`aifs_read("/shared/client-intelligence/templates/{slug}/template.json")` — parse. Note the current `version`, `fields`, and other metadata. This becomes the baseline for the edit.

### Step 4: Collect edits

Walk the admin through the edits conversationally. Show the current field list at the top so they can refer to it. For each edit they want to make, ask:

1. *"What kind of edit? (add / remove / rename / change-property)"*
2. Sub-questions based on the edit type, as listed under Inputs above.
3. *"Add another edit?"* — loop until they say no.

Validate as you go:

- `add`: the new field name must not collide with an existing field name (case-sensitive). If it does, surface the collision and ask for a different name or whether the admin meant `rename`.
- `remove`: the field name must exist in the current fields. If not, halt the edit and ask the admin to pick from the actual field list.
- `rename`: the old name must exist; the new name must not collide with an existing field name (other than the one being renamed).
- `change_property`: the field must exist; the new property must differ from the current property.

### Step 5: Collect migration choice

After all edits are collected, ask:

> *"Should existing client instances be migrated to the new template, or left as-is? Migrate updates every existing client created from this template to apply the schema changes. No-impact records the change but leaves existing instances on the previous version."*

Capture as `migration_choice ∈ {migrate, no_impact}`.

If `migrate` and any edit is `add` with `property: mandatory`: ask the admin for a default value to seed existing instances with. Without a default, existing instances would become invalid against the new template. Capture per-field defaults.

### Step 6: Build the dry-run preview

Compute the new fields list by applying the edits to the current fields:

- `add`: append.
- `remove`: drop by name.
- `rename`: rename in place; preserve order.
- `change_property`: flip the property; preserve order.

The new fields list becomes part of the new template payload.

If `migration_choice == migrate`, enumerate the affected instances:

1. `aifs_list("/shared/client-intelligence/public-index/instances/")`. Filter to directory entries (or .json files depending on layout — V1 uses one JSON per instance directly under `public-index/instances/`).
2. For each entry, `aifs_read("/shared/client-intelligence/public-index/instances/{slug}.json")`. Filter entries whose `template_slug` matches the slug being edited.
3. For each affected instance, `aifs_get_permissions("/shared/client-intelligence/instances/{instance_slug}/")` to check whether the editing admin currently has Drive Writer (Editor) access on the instance folder. Tag each instance as `editable_by_admin: true | false`.

Render a preview table:

```
Template: {slug}
Version: {current_version} -> v{current_version + 1}
Migration: {migrate | no-impact}

Changes:
- add field "phone" (optional)
- remove field "fax"
- rename field "addr" -> "address"

Affected instances ({N} found):
- acme-pharma     [editable: yes]   actions on this instance: phone added, fax removed, addr renamed
- globex-pharma   [editable: yes]   actions on this instance: phone added, fax removed, addr renamed
- initech-pharma  [editable: NO]    actions skipped — admin lacks edit permission on the instance folder
```

If any instance is tagged `editable: NO`, surface explicitly: *"{N} instance(s) won't be migrated because you don't have edit access on them. They'll continue to operate against the previous template version. You can ask the instance owners to grant you edit, or proceed and accept the drift."*

### Step 7: Confirmation gate

Ask the admin to explicitly confirm the preview. Three acceptable responses:

- `confirm` — proceed to Step 8.
- `cancel` — halt cleanly. No writes happen. Surface: *"No changes made."*
- `adjust` — return to Step 4 to revise the edit list.

Do not proceed without explicit confirmation. The phrase "looks good" or similar yes-equivalents count as confirm; ambiguous responses get clarified.

### Step 8: Write the new version snapshot

`aifs_write("/shared/client-intelligence/templates/{slug}/versions/v{N+1}.json", <new_payload>)`.

The new payload has the new fields list, version `N+1`, `created` field preserved from current, `last_updated` set to now.

If the write returns permission-denied, halt with the admin-access error and proceed to no further writes. The version snapshot has not been created, so the canonical template hasn't moved either — clean state.

### Step 9: Write the canonical template (revision-aware)

`aifs_write("/shared/client-intelligence/templates/{slug}/template.json", <new_payload>, if_revision=<captured_revision_from_step_3>)`.

If `REVISION_CONFLICT`: another admin edited this template between Step 3 and now. Halt with: *"This template was edited by another admin while you were editing. Restart `@ai:edit-template {slug}` to see the current state."* The version snapshot from Step 8 remains on disk — it's not addressable via the canonical template until the canonical is bumped. This is an acceptable form of partial state: future repairs can clean orphan snapshots, but they don't break anything.

If permission-denied: halt with the admin-access error. The version snapshot may remain on disk; recommend re-running after permissions are restored.

### Step 10: Append to the template changelog (revision-aware)

Read `_changelog.json` with revision capture. Append one entry per edit in the batch:

```json
{
  "id": "{next_id from current contents}",
  "timestamp": "{ISO_TIMESTAMP}",
  "actor": {"display_name": "...", "member_hash": "..."},
  "template_slug": "{slug}",
  "event": "field_added | field_removed | field_renamed | field_property_changed",
  "details": {
    "field": "{name}",
    "from": {...},
    "to": {...}
  },
  "migration": "{migrate | no_impact}"
}
```

For `field_added` with `migrate` and a mandatory property, include `details.default_value: "{seed value}"`.

Bump `next_id` by the number of new entries. Write back with `if_revision`. Retry up to 3 times on conflict.

### Step 11: (Migrate path only) Apply per-instance changes

For each affected instance tagged `editable: yes` in Step 6:

1. `aifs_stat("/shared/client-intelligence/instances/{instance_slug}/instance.json")` — capture revision.
2. `aifs_read(...)` — load current instance data.
3. Apply the edits to the instance's `template_fields` object:
   - `add` with mandatory: insert key with the admin's seed value.
   - `add` with optional: insert key with empty string or null.
   - `remove`: delete the key.
   - `rename`: copy value from old key to new key; delete old key.
   - `change_property`: no data change.
4. Bump the instance's `template_version` field to match the new template version.
5. `aifs_write(..., if_revision=...)`. On `REVISION_CONFLICT`, re-read and re-apply once (up to 1 retry per instance to avoid blocking the entire migration on a noisy instance). After 1 retry, mark the instance as failed and continue.
6. Append entries to `instances/{instance_slug}/changelog.json` (revision-aware, same retry semantics). One entry per edit applied to this instance, each referencing the template changelog entry ID from Step 10:

   ```json
   {
     "id": "{next_id}",
     "timestamp": "{ISO_TIMESTAMP}",
     "actor": {"display_name": "...", "member_hash": "..."},
     "event": "field_added_by_migration | field_removed_by_migration | field_renamed_by_migration | field_property_changed_by_migration",
     "details": {
       "field": "{name}",
       "template_changelog_id": "{id of the corresponding template changelog entry}"
     }
   }
   ```

For each instance tagged `editable: no` in Step 6, skip — do not attempt the write. Record the skip in the per-run summary.

For any instance where a write returns permission-denied at runtime (despite the pre-check), record the failure and continue with remaining instances. Do not abort the whole migration on a single instance failure.

### Step 12: Surface the per-run summary

```
Template `{slug}` edited.
Version: {current_version} -> v{new_version}
Migration: {migrate | no-impact}

Changes:
- {N} edits applied to the template.

{If migrate:}
Per-instance migration:
- {N_succeeded} instance(s) migrated successfully.
- {N_skipped_no_perm} instance(s) skipped — no edit permission.
- {N_failed} instance(s) failed mid-write (likely intermittent).

{If any skipped or failed:}
Skipped instances:
- acme-pharma (no edit permission)
- ...

Failed instances:
- globex-pharma (revision conflict after retry; instance has been edited concurrently; manually inspect)
- ...

Recommend: re-run `@ai:edit-template {slug}` later to retry failed instances, or ask instance owners to grant you edit access on skipped instances.
```

End with next-action suggestions:

> *"View the updated template: `@ai:view-template {slug}`. List clients using this template: `@ai:list-clients` filtered by template_slug={slug}."*

---

## Directives

### Behavior

This is a high-stakes task that touches many files. Always show the dry-run preview before any write. Never proceed past Step 7 without explicit admin confirmation. When the admin chooses Migrate, walk them through the implications — particularly that mandatory adds need a default value for existing instances and that they won't be able to migrate instances they lack edit access on.

Adapt to admin expertise: a non-technical admin gets careful explanations of mandatory-vs-optional and migrate-vs-no-impact; an experienced admin gets a terser flow with just the confirmation gates.

If the admin describes their edit ambiguously ("rename the contact field"), ask for the exact old and new names. Do not infer field names from partial mentions.

### State Management

Partially stateful within an invocation. The collected edits and confirmation are in-memory until Step 8; nothing on disk changes until Step 8. After Step 8, partial state is possible (snapshot written but canonical not bumped; canonical bumped but changelog not appended; per-instance migrations partially applied). The dry-run preview makes the intent explicit so partial state is recognizable on re-run.

### Constraints

- **Never call `aifs_share`, `aifs_unshare`, or `aifs_transfer_ownership`.** This task modifies content files only. Authority is filesystem-enforced.
- **Never pre-check admin status via `aifs_get_permissions` on `templates/`.** Try the write; let the filesystem reject; translate the response. The exception is `aifs_get_permissions` on per-instance folders during the dry-run preview, which is a purposive read for the editable/non-editable tag, not an authority gate.
- **Never proceed past the confirmation gate without an explicit admin confirmation.** The dry-run preview is the contract; the confirmation is the consent.
- **Never delete the prior version snapshot.** Old `vN.json` files are immutable history. Only `template.json` (the canonical pointer) moves forward.
- **Never auto-cascade migrations across templates.** If a renamed field happens to share a name with a field in a different template, this task touches only the template being edited.

### Edge Cases

- **Admin tries to remove the only field.** Surface a warning: *"This template will have zero fields after the edit. Templates with no fields are not useful — every client created from this template will be empty. Confirm or cancel?"*
- **Admin tries to add a field, then rename it within the same edit batch.** Order matters; treat the operations in the order given. If the rename targets a name that doesn't yet exist (because the add hasn't been "applied" in linear order), surface and ask the admin to reorder or combine.
- **Admin tries to remove a field, then add a field with the same name.** Allow it; record both events in the changelog. The semantic is "drop the old field's data; re-introduce as a new field." Make this consequence clear in the preview.
- **`_changelog.json` revision conflict persists after 3 retries.** Halt with a contention message. The template and snapshot writes from Steps 8–9 succeeded; the changelog is now out of sync. Recommend manual reconciliation by re-running the task or by adding a `template_*_repair` event via direct filesystem edit (admin's call).
- **Per-instance migration is partway done when the admin's filesystem connection drops.** The instances already migrated stay migrated. Re-running the task with the same edits later will detect the new version in Step 3 and refuse to apply duplicate changes; the admin then runs `@ai:list-clients` to identify which instances are still on the old version and either manually migrates them or accepts the drift.
- **Instance migration partially fails on a single instance (e.g., one field write succeeds but the second fails).** The instance is left in a mixed state. The per-instance changelog reflects only the entries that were appended. Record this in the per-run summary as "failed (partial)" and recommend the admin manually inspect the instance.
- **A field rename collides with an extension field on an existing instance.** An instance with `template_fields.contact` being renamed to `phone` — but the instance also has an extension field `phone`. This would clobber the extension data. Pre-emptively, during the dry-run, scan all affected instances for extension-field collisions; surface them in the preview and let the admin decide (proceed with the rename and accept the clobber, or cancel and pick a different rename target).
