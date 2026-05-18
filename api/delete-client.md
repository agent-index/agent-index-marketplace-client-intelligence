---
name: delete-client
type: task
version: 1.0.0
collection: client-intelligence
description: Member-facing task to delete a client instance. Two modes - soft delete (flips status to archived; preserves data and changelog; recoverable in principle though no V1 task implements unarchive) and hard delete (permanently removes the instance folder and public-index entry; appends a finalizing entry to the collection-wide template changelog because the instance's own log dies with it). Hard delete is irreversible and requires explicit caller confirmation.
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
writes_to: "/shared/client-intelligence/"
---

## About This Task

Delete Client removes a client instance from the collection. Two modes:

- **Soft delete** marks the client as archived in the public-index entry and appends a `soft_deleted` event to the instance changelog. The instance folder and all data remain on the filesystem. Default views (e.g., `list-clients` with `filter_status=active`) hide archived clients, but the data is still readable by anyone with view permission. V1 ships without an `unarchive` task — admins can manually flip the status field if needed (a future task may automate this).
- **Hard delete** is permanent. The instance changelog gets a final `hard_delete_pending` entry, then the public-index entry is deleted, then the entire instance folder is removed. A finalizing `instance_hard_deleted` entry is appended to the collection-wide template changelog (since the instance's own log is going away with it). No recovery path.

Both modes require Delete permission on the instance folder (which V1 maps to Drive Writer on the folder since Drive doesn't distinguish edit from delete — the distinction is task-layer enforced). Hard delete additionally requires explicit caller confirmation: *"yes, permanently delete X"* or equivalent. Soft delete proceeds with normal "are you sure" confirmation.

### Inputs

- **`slug`** (required, interactive or argument) — the instance slug to delete.
- **`mode`** (required, interactive) — `soft` or `hard`.
- **`confirmation`** (required, interactive) — explicit confirmation phrase. For hard delete, the phrase must include "permanently" or equivalent.

### Outputs

- **Soft delete:**
  - `/shared/client-intelligence/public-index/instances/{slug}.json` — `status` updated to `archived` (revision-aware).
  - `/shared/client-intelligence/instances/{slug}/changelog.json` — appended `soft_deleted` event (revision-aware).
- **Hard delete:**
  - `/shared/client-intelligence/instances/{slug}/changelog.json` — appended `hard_delete_pending` event (revision-aware, final write to this file).
  - `/shared/client-intelligence/public-index/instances/{slug}.json` — deleted via `aifs_delete`.
  - `/shared/client-intelligence/instances/{slug}/` — entire folder deleted via `aifs_delete` (recursive, or file-by-file then folder).
  - `/shared/client-intelligence/templates/_changelog.json` — appended `instance_hard_deleted` event (revision-aware).

Confirmation message surfaced to the caller.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. If false, halt with the not-installed message.

### Step 2: Resolve slug

Same resolution as `view-client`: accept slug or name; disambiguate via public-index if needed.

### Step 3: Read what's being deleted

`aifs_read("/shared/client-intelligence/public-index/instances/{slug}.json")` to confirm existence and show metadata. If false, halt with: *"No client with slug `{slug}` exists. Run `@ai:list-clients` to see all clients."*

Attempt `aifs_read("/shared/client-intelligence/instances/{slug}/instance.json")` to display field summary in the confirmation prompt. If permission-denied, proceed with name-only display in the prompt — but note in the prompt that the caller doesn't have view access to the data being deleted, which is rare but possible (caller has delete but not view).

### Step 4: Ask which mode

> *"How should `{name}` ({slug}) be deleted?
> 1. Soft delete — flip the status to archived. Data is preserved; the client just stops appearing in default lists. Recoverable in principle (V1 doesn't ship an unarchive task; an admin can manually flip the status).
> 2. Hard delete — permanently remove the client folder, all data, and the public-index entry. Cannot be undone."*

Capture as `mode ∈ {soft, hard}`.

### Step 5: Confirm

**For soft delete:**

> *"Soft-delete `{name}` ({slug})? It'll be hidden from default lists but the data stays. Confirm yes/no."*

Proceed only on yes.

**For hard delete:**

Show the data summary (or "no view access" note if the read failed). Then:

> *"This will permanently delete `{name}` ({slug}) — all data, the changelog, the public-index entry. There is no recovery. To confirm, type 'yes, permanently delete {slug}' exactly."*

Match the caller's response exactly (case-insensitive on the literal phrase; the slug substitution must match the actual slug). Any deviation halts with: *"Confirmation didn't match. Cancelled — no changes made."*

### Step 6: Execute the delete

**Soft delete path:**

1. Read `public-index/instances/{slug}.json` with revision capture.
2. Update `status` to `archived`; set `archived_at` to current ISO timestamp; set `archived_by` to the caller's `member_hash`.
3. `aifs_write` with `if_revision`. On conflict, re-read and retry (up to 3).
4. Read `instances/{slug}/changelog.json` with revision capture.
5. Append a `soft_deleted` entry:
   ```json
   {
     "id": "{next_id}",
     "timestamp": "{ISO_TIMESTAMP}",
     "actor": {"display_name": "{caller.display_name}", "member_hash": "{caller.member_hash}"},
     "event": "soft_deleted",
     "details": {}
   }
   ```
   Bump `next_id`. Write back with `if_revision`. Retry up to 3.
6. Proceed to Step 7.

**Hard delete path:**

1. Read `instances/{slug}/changelog.json` with revision capture.
2. Append a `hard_delete_pending` entry as the FINAL write to this file before it gets deleted:
   ```json
   {
     "id": "{next_id}",
     "timestamp": "{ISO_TIMESTAMP}",
     "actor": {"display_name": "{caller.display_name}", "member_hash": "{caller.member_hash}"},
     "event": "hard_delete_pending",
     "details": {
       "confirmed_at": "{ISO_TIMESTAMP}"
     }
   }
   ```
   Bump `next_id`. Write back with `if_revision`. Retry up to 3.
3. `aifs_delete("/shared/client-intelligence/public-index/instances/{slug}.json")`. If fails, halt — the partial state is acceptable (the instance still exists; the changelog has a hard_delete_pending entry but no public-index removal yet; caller can re-run).
4. Delete the contents of `instances/{slug}/`. If the adapter supports recursive delete on a directory, prefer that. Otherwise enumerate and delete file-by-file, then delete the folder. Specifically: delete `instances/{slug}/instance.json`, `instances/{slug}/changelog.json`, then the folder itself.
5. Append a finalizing entry to `templates/_changelog.json` (revision-aware). This entry records that the instance was hard-deleted, since the instance's own changelog is gone:
   ```json
   {
     "id": "{next_id}",
     "timestamp": "{ISO_TIMESTAMP}",
     "actor": {"display_name": "{caller.display_name}", "member_hash": "{caller.member_hash}"},
     "template_slug": "{template_slug from the public-index entry}",
     "event": "instance_hard_deleted",
     "details": {
       "instance_slug": "{slug}",
       "instance_name": "{name from the public-index entry}",
       "created": "{created from the public-index entry}",
       "created_by": "{created_by from the public-index entry}"
     },
     "migration": null
   }
   ```
   Bump `next_id`. Write back with `if_revision`. Retry up to 3.
6. Proceed to Step 7.

### Step 7: Confirm to caller

**Soft:**

> *"`{name}` ({slug}) soft-deleted. Status: archived. The data is preserved; the client is hidden from default lists. To view archived clients: `@ai:list-clients filter_status=archived`."*

**Hard:**

> *"`{name}` ({slug}) permanently deleted. The instance folder, all data, and the public-index entry have been removed. The deletion is recorded in the collection's template changelog. There is no recovery."*

---

## Directives

### Behavior

For soft delete, behave like a normal confirmation flow — yes/no is enough. For hard delete, be deliberate. The literal-phrase confirmation in Step 5 is a guardrail against accidental destructive operations; do not bypass it even if the member sounds certain. If they object to the phrasing, suggest soft delete instead.

If the member's stated intent is ambiguous ("delete X"), ask the mode question (Step 4) before any confirmation. Don't assume hard.

### State Management

Not stateful from the agent's side. Partial state is possible on hard delete if the multi-step write sequence is interrupted (changelog entry written, but public-index not deleted; or public-index deleted but folder still partially intact). The hard_delete_pending event in the instance changelog (Step 6.2) is the durable signal that the deletion was initiated — re-running the task with the same slug will detect a `hard_delete_pending` entry and offer to resume the deletion from that point.

### Constraints

- **Never bypass the literal-phrase confirmation** for hard delete. The phrase is the gate.
- **Never call `aifs_share`, `aifs_unshare`, or `aifs_transfer_ownership`.** Delete doesn't change permissions on remaining resources.
- **Never pre-check delete permission via `aifs_get_permissions`.** Attempt the write; let the filesystem reject; translate.
- **Never hard-delete archived state without confirmation.** The path-from-soft is the same: caller must run delete-client again with mode=hard and re-confirm with the literal phrase.
- **Never modify the templates/ contents from this task** beyond appending to `_changelog.json`. Templates themselves are not affected by instance deletion.

### Edge Cases

- **Public-index entry exists but the instance folder is missing.** Step 6.4 on hard delete becomes a no-op (nothing to delete in `instances/{slug}/`); proceed. Surface a note that the state was already partially gone.
- **Instance folder exists but public-index entry is missing.** Step 3 catches it (`aifs_read` on public-index fails). The caller should be told the client is in an inconsistent state and to ask an admin to investigate; the task halts without modifying state.
- **Revision conflict on public-index status update (soft).** Retry up to 3, then halt with: *"The client's public-index entry is being modified concurrently. Try again."*
- **Revision conflict on changelog appends (soft or hard).** Same.
- **Revision conflict on `templates/_changelog.json` after 3 retries during hard delete.** The instance has already been deleted at this point (Step 6.4 completed). Surface: *"`{slug}` was deleted, but the deletion couldn't be recorded in the template changelog due to concurrent edits. Audit gap; consider manually appending the entry or accepting it."* The deletion itself is permanent regardless.
- **Caller tries to delete an instance they can't view but DO have delete on.** Allowed. The caller sees only name + slug in the confirmation; the rest of the data summary is omitted with a note that the caller doesn't have view access to see what they're deleting. This is unusual but not blocked.
- **Caller tries to soft-delete an already-archived client.** Surface: *"`{slug}` is already archived. No-op."* The status field is already `archived`; appending another `soft_deleted` event is redundant.
- **Caller tries to hard-delete an already-archived client.** Allowed. Proceeds to Step 5's hard-delete confirmation.
