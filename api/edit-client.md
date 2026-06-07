---
name: edit-client
type: task
version: 2.1.0
collection: client-intelligence
description: Modify a client instance's data — template field values, extension fields. Tier-resolved via the universal-floor pointer; write authority is mechanical (org-public = any member with mandatory changelog attribution; private = owner and collaborators). Revision-aware writes; every edit appends an attributed changelog event.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
reads_from: "/shared/client-intelligence/, id:{folder_id}/"
writes_to: "/shared/client-intelligence/instances/, id:{folder_id}/"
---

## About This Task

Edit Client modifies an existing client's data: change template-field values, change/add/remove extension fields. It writes only inside the instance's own folder — never the template (that's `edit-template`), never the pointer except `last_updated` (rename remains out of scope as in 1.x).

**Write authority is mechanical per tier:**
- **Org-public:** any member can edit (uniform all@ writer — the commons deal). The guardrail is **mandatory changelog attribution**: every edit appends an event naming the editor; an edit that can't write the changelog must not stand silently (see Step 7).
- **Private:** the owner and collaborators (Drive `writer` grants the owner made). Readers and strangers get ACCESS_DENIED from Drive itself — surface the floor message ("private to {owner}; ask them for collaborator access").

### Inputs

As 1.x: `slug` (or name), then `edits` — `set_template_field_value`, `set_extension_field_value`, `add_extension_field`, `remove_extension_field`.

## Workflow

### Step 1: Pre-flight

Auth + installed checks.

### Step 2: Resolve via pointer

Slug or name → pointer. `status: archived` → *"Archived clients aren't editable. Unarchive: ask {owner|an admin} to reset status via `@ai:delete-client {slug}` (unarchive option)."* Halt. `archived-moved-private` stub → follow pointer; never edit stubs.

### Step 3: Tier-resolved paths

Org-public → base = `location.path`. Private → base = `id:{location.folder_id}/`. All subsequent reads/writes use the base.

### Step 4: Read instance with revision capture

`aifs_stat(base + "instance.json")` → revision; `aifs_read` → current data. (Private-tier ACCESS_DENIED here = caller lacks any access → floor message, halt. Read-only grantee discovers their limit at Step 7's write — see edge cases.)

### Step 5: Collect edits

As 1.x: walk current values, collect operations, validate (mandatory template fields cannot be set empty; extension names can't collide with template fields; removing a nonexistent extension field → notice).

### Step 6: Confirm

Show a before→after diff summary. No writes before explicit confirmation.

### Step 7: Write (revision-aware, attribution-mandatory)

1. `aifs_write(base + "instance.json", updated, if_revision={captured})`. On `REVISION_CONFLICT`: re-read, re-apply the member's edits onto the fresh data, re-confirm if anything they edited changed underneath them, retry (cap 3).
2. Append one changelog event per edit operation to `base + "changelog.json"` (revision-aware append): actor display_name + member_hash, event `field_edited`/`extension_added`/`extension_removed`, field name, old→new (values elided for long text; lengths noted).
3. **Attribution invariant (org tier especially):** if the changelog append fails after the instance write succeeded, retry it; if it still fails, surface loudly — *"Your edit is saved but UNATTRIBUTED in the changelog ({error}). Re-run `@ai:edit-client {slug}` later to repair the log, or tell your admin."* Never skip silently.
4. Refresh the pointer's `last_updated` (overwrite; best-effort — note on failure, don't roll back).
5. Private-tier ACCESS_DENIED on the write = caller is a **reader**, not collaborator: *"You can read `{name}` but not edit it — ask {owner} to make you a collaborator."*

### Step 8: Confirm

What changed, who's attributed, where it lives.

### Branding for documents (added in 2.1.0)

The member may set the instance's document-branding context, stored INSIDE the instance (tier-inherited, never org-readable unless the instance is):
- `branding/branding.json` — `{client_display_name}` (how the client is named on documents).
- `branding/client-logo.{png|svg|jpg}` — the client's logo for template slots. Upload via the member's local file; size guard >2MB warn (base64 transport).
These feed `client-brief` (and any future brand-aware surface) as SLOT VALUES — the org's brand book defines where/how they appear; this instance owns what they are. Writes use the same tier mechanics as all instance edits (org path or `id:` anchor) and append a changelog event.

## Directives

### Constraints

- Never modify templates, pointers (beyond `last_updated`), or other instances.
- Never edit stubs or archived clients.
- Revision-aware writes always; the attribution invariant is non-negotiable — it is the org tier's entire governance model (bug-reports precedent).
- No writes before Step 6 confirmation.

### Edge Cases

- Template field renamed/removed in a newer template version: the instance binds to its captured `template_version` — edit against the instance's own field set; note the template drift and point at `edit-template`'s migration flow.
- Concurrent editors (org tier): revision conflicts handled at Step 7; if the same field changed underneath, the member decides.
- Pointer `last_updated` refresh fails (index dir issue): note it; instance edit stands.
- `owner_departed` private client: collaborators can still edit (their grant survives); note that the owner can no longer manage access.
