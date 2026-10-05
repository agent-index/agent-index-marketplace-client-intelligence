---
name: transition-client
type: task
version: 2.1.0
collection: client-intelligence
description: Move a client between visibility tiers. Private → org-public copies the instance into the org commons (uniform access; the org gains custody). Org-public → private copies it into the owner's own My Drive and leaves an archived marker in the commons (members cannot delete /shared content — and going private does not un-publish what the org already saw). Owner-only; the universal-floor pointer is overwritten to match.
stateful: false
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
reads_from: "/shared/client-intelligence/, id:{member_folder_id}/clients/"
writes_to: "/shared/client-intelligence/, id:{member_folder_id}/clients/"
---

## About This Task

Transition Client moves a client between the two visibility tiers (design 07 §5). New in 2.0.0; registered with routing triggers like "make client X public", "share client X with the whole org", "take client X private".

**Private → org-public:** the org gains custody (the record survives the owner's departure). The instance files are COPIED from the owner's My Drive into `/shared/client-intelligence/instances/{slug}/` (the owner has all@ writer there); the pointer flips to `scope: "org_public"` with the /shared location; any existing per-person grants on the private copy become moot (optionally revoked); the My Drive copy is marked moved.

**Org-public → private:** the owner copies the instance into their My Drive and the pointer flips to the folder-ID location. **The /shared original cannot be deleted by a member** — it is overwritten with an archived marker (`status: "archived-moved-private"`) and its `instance.json` is reduced to a stub (name, slug, marker, pointer reference) so stale full data doesn't linger in the commons. **Surface plainly: going private does not un-publish what the org already saw**; an admin can purge the archived stub on request.

Owner-only in both directions (`owner_hash` on the pointer). For org-public clients created before anyone tracked an owner, the admin may run it.

### Inputs

- **`slug`** (required) — resolved via the universal-floor pointer.
- **`direction`** — inferred from the current tier; confirmed with the member.

## Workflow

### Step 1: Pre-flight + resolve

Auth/installed checks; read `member-index.json` (`member_hash`, `member_folder_id` — missing → `@ai:update` self-provisioning halt). Read the pointer; halt if missing or `status: archived`. **For a private client, capture `old_folder_id = location.folder_id` now** — Step 3A.4 replaces `location`, and Step 3A.6 still needs the old private folder. Verify caller is owner (admin override only for ownerless legacy org-public records, with explicit confirmation).

### Step 2: Confirm the move and its consequences

Private → public: *"This puts `{name}` in the org commons: every member can view and edit it, and the org keeps it regardless of who comes or goes. Your per-person grants ({list}) become irrelevant. Proceed?"*

Public → private: *"This moves `{name}` into your personal space: only you (and people you grant) can see it going forward — but the org has already had access to everything in it, and an archived stub stays in the commons (admins can purge it). The record will leave org custody and depend on your account. Proceed?"*

No writes before explicit confirmation.

### Step 3A: Private → org-public

1. Read every file under `id:{location.folder_id}/` (instance.json, changelog.json, any extras).
2. Write each to `/shared/client-intelligence/instances/{slug}/{same name}` (slug collision with an existing commons instance → halt; shouldn't happen given pointer uniqueness, but never overwrite someone else's record).
3. Update the copied `instance.json`: `"visibility": "org_public"`. Append a `transitioned_to_org_public` changelog event (actor = owner).
4. **Overwrite the pointer:** `scope: "org_public"`, `location: {"path": ...}`, `last_updated`.
5. Mark the My Drive copy: overwrite its `instance.json` with `"status": "moved-to-org"` stub (or, at the owner's choice, delete it — their space, their call; offer both).
6. Optional cleanup, offered not forced (skip if the owner deleted the My Drive copy in step 5): revoke now-moot per-person grants on the old private folder.
   - Read `aifs_get_permissions({path: "id:{old_folder_id}"})` **fresh** (not the pointer's old `scope`) and list the explicit grants, excluding the owner.
   - **Ask which to remove before building any table** — all, some (by name), or none. "None" (or no clear answer) ends the step; declining leaves harmless grants on a stub.
   - For the chosen subjects, follow `/agent-index-core/internal/permission-changes.md` with `calling_task: "transition-client"`, `confirmer: "owner"`, and **one `unshare` per subject**: resource `id:{old_folder_id}`, `resource_label` "Client '{name}' (old private copy)", `is_folder: true`, recipient = the subject exactly as the read reported it, `reason` "The client moved to the org commons; this grant on the old private copy is no longer needed", `source: "current_state"`. Each removal is confirmed by the owner individually.
   - Report with the procedure's summary lines. Nothing downstream depends on these rows (the pointer already points at the commons copy), so a declined or failed removal only means that grant stays on the stub.

### Step 3B: Org-public → private

1. Read every file under `/shared/client-intelligence/instances/{slug}/`.
2. Write each to `id:{member_folder_id}/clients/{slug}/`; `aifs_stat` the folder → `client_folder_id` **and `drive_id`** (the owner's drive — needed for C.1.3 `crossdriveread`).
3. Update the private `instance.json`: `"visibility": "private"`. Append a `transitioned_to_private` changelog event.
4. **Overwrite the pointer:** `scope: "private"`, `location: {"folder_id": client_folder_id, "item_drive_id": drive_id}`, `last_updated`. Always write `item_drive_id` — without it a later grantee on OneDrive can discover the client but not open it (readers open via `id:{item_drive_id}:{folder_id}/...`).
5. **Archive the commons original** (cannot delete): overwrite its `instance.json` with the stub `{"slug", "name", "status": "archived-moved-private", "moved": "{ISO}", "see": "public-index pointer"}` and append a final changelog event. `list-clients` filters this status; `view-client` on the stub redirects via the pointer.

### Step 4: Confirm

State the new tier, what changed, and the follow-on verbs (`grant-permission` for newly-private; nothing needed for org-public).

## Directives

### Constraints

- Owner-only (admin only for ownerless legacy records, confirmed explicitly).
- **Copy, then flip the pointer, then mark the source** — in that order; a failure mid-way must leave the pointer still pointing at a complete copy (the pointer is the source of truth for location).
- Never delete anything under /shared; never leave the commons holding full data for a privatized client (stub overwrite is mandatory, not optional).
- No writes before Step 2's confirmation.

### Edge Cases

- Copy fails mid-way (3A/3B step 2): pointer untouched → client still lives at the old location intact; surface what copied and advise re-run (idempotent overwrites).
- Pointer flip succeeds but source-marking fails: harmless inconsistency (pointer rules); surface and offer re-run of the marking step.
- Private→public where the owner had granted collaborators who actively edit: warn that their in-flight knowledge of the folder_id keeps working against the OLD copy — which is now a stub; they should re-discover via list-clients.
- A legacy 1.x instance mid-migration (upgrade script running): defer — "finish the 2.0 upgrade first."
