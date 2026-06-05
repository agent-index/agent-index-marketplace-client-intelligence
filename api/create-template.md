---
name: create-template
type: task
version: 2.0.0
collection: client-intelligence
description: Admin-only task to author a new client template. Interviews the admin for a slug, display name, and field list (each field mandatory or optional), then writes the template file plus an immutable v1 snapshot and appends a `template_created` event to the collection-wide template changelog.
stateful: false
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter contract 2.0.0 or higher)
reads_from: "/shared/client-intelligence/"
writes_to: "/shared/client-intelligence/templates/"
---

## About This Task

Create Template authors a new client template for the Client Intelligence collection. Templates define the fields a client instance carries — each field is marked mandatory or optional. The template's slug becomes the identifier used by `create-client`; the display name is what members see when picking a template at instance-creation time.

Authority on this task is enforced by the filesystem, not by collection-side checks. Only members with Drive Editor access to `/shared/client-intelligence/templates/` can write template files; any other caller will receive a permission-denied response on the first write and be told to request admin access via `@ai:add-admin`. The task does not pre-check admin status — it attempts the operation and translates the response.

### Inputs

- **`slug`** (interactive) — kebab-case identifier unique within the collection (e.g., `pharma-client`, `consulting-engagement`). Used as the folder name under `/shared/client-intelligence/templates/{slug}/`.
- **`display_name`** (interactive) — human-readable name shown to members at template-selection time (e.g., "Pharma Client").
- **`description`** (interactive, optional) — short description explaining what kind of client this template is for.
- **`fields`** (interactive) — list of field definitions. Each field has:
  - `name` — field identifier (e.g., `company_name`, `primary_contact_email`).
  - `property` — either `mandatory` or `optional`.
  - `description` (optional) — short human-readable description of the field.

### Outputs

Three files written to the remote filesystem:

- `/shared/client-intelligence/templates/{slug}/template.json` — the canonical current template.
- `/shared/client-intelligence/templates/{slug}/versions/v1.json` — immutable snapshot of v1.
- `/shared/client-intelligence/templates/_changelog.json` — collection-wide changelog with a new `template_created` event appended.

Confirmation message surfaced to the admin including the slug, field count, and example invocation for `create-client`.

---

## Workflow

### Step 1: Pre-flight checks

- Confirm filesystem access: `aifs_auth_status`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt with: *"Creating a template needs remote filesystem access. Run `@ai:member-bootstrap` if the connection is broken."*
- Confirm the collection is installed: `aifs_exists("/shared/client-intelligence/collection-state.json")`. If false, halt with: *"client-intelligence is not installed on this org. An org admin must install it before templates can be authored."*

### Step 2: Collect inputs

Interview the admin conversationally. Ask one question at a time using progressive disclosure:

1. **Slug.** *"What slug should this template have? Use kebab-case, like `pharma-client` or `consulting-engagement`. The slug is the identifier — it becomes the folder name and members reference it at client-creation time."* Validate: lowercase letters, digits, hyphens only; non-empty; no leading or trailing hyphen.
2. **Display name.** *"What's the display name for this template? This is what members will see when picking a template — for example, 'Pharma Client'."*
3. **Description (optional).** *"Want to add a short description of what kind of client this template is for? Optional."*
4. **Fields.** Walk the admin through adding fields one at a time. For each:
   - *"Field name (kebab-case or snake_case): "*
   - *"Mandatory or optional?"*
   - *"Short description (optional): "*
   - After each field, *"Add another field?"* until the admin says no.

Show a summary of all decisions and ask for confirmation before any write.

### Step 3: Validate inputs

- **Slug uniqueness.** `aifs_exists("/shared/client-intelligence/templates/{slug}/template.json")`. If true, halt with: *"A template with slug `{slug}` already exists. Pick a different slug or use `@ai:edit-template` to modify the existing one."*
- **Field name uniqueness.** Within the proposed fields list, no two fields may share a `name`. If duplicates, surface the collision and ask the admin to pick distinct names.
- **At least one field.** Templates with zero fields are not useful. If the admin authored no fields, ask whether they want to add at least one before proceeding.

### Step 4: Build the template payload

Compose:

```json
{
  "slug": "{slug}",
  "name": "{display_name}",
  "version": 1,
  "created": "{ISO_DATE}",
  "last_updated": "{ISO_DATE}",
  "description": "{description or empty string}",
  "fields": [
    {"name": "{field_name}", "property": "mandatory|optional", "description": "{description or empty string}"},
    ...
  ]
}
```

### Step 5: Write the template files

In order:

1. `aifs_write("/shared/client-intelligence/templates/{slug}/template.json", <payload>)` — writes the canonical current template.
2. `aifs_write("/shared/client-intelligence/templates/{slug}/versions/v1.json", <payload>)` — writes the immutable v1 snapshot.

If either write returns a permission-denied response, surface: *"Creating a template requires admin access on the client-intelligence collection. Admins are members with Drive Editor access to `/shared/client-intelligence/templates/`. Ask an existing admin to grant you via `@ai:add-admin`."* and halt — do not proceed to Step 6.

If either write returns any other error, surface the error verbatim and halt. The admin can re-run after resolving the underlying issue.

### Step 6: Append to the template changelog

This is a revision-aware append to a shared file. Multiple admins could in principle write concurrently.

1. `aifs_stat("/shared/client-intelligence/templates/_changelog.json")` — capture the current revision.
2. `aifs_read("/shared/client-intelligence/templates/_changelog.json")` — read the current contents.
3. Compose the new entry:
   ```json
   {
     "id": "{next_id from current contents}",
     "timestamp": "{ISO_TIMESTAMP}",
     "actor": {"display_name": "{caller.display_name}", "member_hash": "{caller.member_hash}"},
     "template_slug": "{slug}",
     "event": "template_created",
     "details": {
       "fields": [<copy of the fields array>]
     },
     "migration": null
   }
   ```
4. Append the entry; bump `next_id` by 1.
5. `aifs_write` with `if_revision` set to the captured revision. On `REVISION_CONFLICT`, re-read and retry — at most 3 retries. After 3 failed retries, halt with: *"The template changelog is being modified concurrently. Try again in a moment."*

If the write returns permission-denied: same admin-access error as Step 5. This shouldn't happen if Step 5 succeeded (both writes target `templates/`), but handle it for completeness.

### Step 7: Confirm to admin

Surface:

> *"Template `{slug}` created. {field_count} field(s) — {N_mandatory} mandatory, {N_optional} optional. Members can now create clients from this template with `@ai:create-client` and selecting `{slug}`. Edit the template later with `@ai:edit-template {slug}` or view it with `@ai:view-template {slug}`."*

---

## Directives

### Behavior

This task operates as a conversational interview. The admin describes what they want; the task translates that into the structured template payload. Adapt verbosity to the admin's experience level: if they're new, explain what mandatory vs. optional means; if they're experienced, keep questions terse.

Be liberal in interpreting the admin's input. If they say "client name (required)", parse that as `{name: "client_name", property: "mandatory"}`. If they say "primary contact (optional, email)", parse the name and property; if they offer "email" as a type hint, note that V1 fields are all free-text (no field types) and ask whether they want to keep the description as "email" or drop it.

### State Management

This task is not stateful from the agent's side. All state lives in the remote filesystem after the writes complete. The interview itself is in-memory only.

### Constraints

- **Never call `aifs_share`, `aifs_unshare`, or `aifs_transfer_ownership`.** This task does not modify permissions on any folder; it only writes content files. Authority is filesystem-enforced via the existing ACLs on `/shared/client-intelligence/templates/`.
- **Never pre-check admin status via `aifs_get_permissions`.** Attempt the write; let the filesystem reject if the caller isn't an admin; translate the response.
- **Never overwrite an existing template.** Step 3's slug-uniqueness check enforces this. The slug is the identity; reuse means edit, not create.
- **Never write a template with zero fields without confirming the admin really wants that.**

### Edge Cases

- **Admin authors a slug that exists as a soft-deleted template.** V1 has no soft-delete for templates (only for instances). If a `{slug}/template.json` exists, the slug is taken — even if the template was marked archived (which it can't be in V1 anyway). Future versions that introduce template soft-delete will need to revisit Step 3.
- **Admin cancels mid-interview.** Any time before Step 5, abandoning the interview produces no side effects. The task tracks no partial state.
- **Filesystem available for `aifs_exists` (Step 3) but not for `aifs_write` (Step 5).** Halt at Step 5 with the surfaced error. The template directory might be partially created (the `aifs_exists` check is read-only, but if Step 5's first write succeeded and the second failed, an orphaned `template.json` could exist without its `versions/v1.json`). Recommend re-running the task; the slug-uniqueness check in Step 3 will fire on the retry, surfacing the partial state — the admin can then manually clean up or call `@ai:edit-template` to repair.
- **Revision conflict on changelog after 3 retries.** Surface and halt. Document recommends the admin retry; if conflicts persist, that's a signal of contention that should be investigated separately.
