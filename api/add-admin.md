---
name: add-admin
type: task
version: 2.1.1
collection: client-intelligence
description: Admin-only task to grant admin role to another member of the org. Admin role is derived from the filesystem - a collection admin is any member with Drive Editor access to /shared/client-intelligence/templates/ and /shared/client-intelligence/config/. This task applies two shares (one per folder) as one all-or-nothing table through the core permission-changes procedure (confirmer collection_admin); the caller confirms both at once. Authority is backend-enforced - only existing collection admins can share these folders, so anyone else is refused by the backend.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter contract 2.0.0 or higher)
reads_from: "/shared/client-intelligence/"
writes_to: null
---

## About This Task

Add Admin grants admin role on the client-intelligence collection to another org member. The collection's admin model is derived from filesystem state: admins are members with Drive Editor access to `templates/` and `config/`. Granting admin means sharing both folders with Editor. The task applies both shares as one all-or-nothing table through the core permission-changes procedure, so the caller confirms a single, coherent permission change.

**Authority is enforced by the backend, not by an org-admin check.** Client-intelligence admins need not be org admins, so the task runs the procedure with `confirmer: "collection_admin"`. Only an existing collection admin can share `templates/` and `config/`; for anyone else the backend refuses the share (the row comes back `failed`, access denied) and the task surfaces the admin-access error. The task does not pre-check the caller's role and never uses `collection_admin` to stand in for an org-admin requirement. **In practice, on a Google Shared Drive only drive Managers can change folder sharing (audit finding F12)**, so the person running this needs Manager rights on the drive, not just Editor on these folders. If they lack it, the procedure reports the change `failed` with access denied; tell them an org admin (a drive Manager) has to run it.

### Inputs

- **`member`** (required, interactive) — email or display name of the member to promote to admin. Resolved against `members-registry.json`.

### Outputs

The task passes two share changes to the core permission-changes procedure, which applies them under the caller's own credentials after the caller confirms, on these resources:

- `/shared/client-intelligence/templates/` — Writer for `{member.sharing_identity}`
- `/shared/client-intelligence/config/` — Writer for `{member.sharing_identity}`

Drive Activity records both share events. No collection-side files are modified.

Confirmation surfaced to the caller after the procedure returns `complete` with changes applied.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. Halt if not authenticated.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. Halt if false.

### Step 2: Resolve target member

Accept email or display name. Resolve via `members-registry.json` — read it by id anchor (`aifs_read("id:{resource_ids.members_registry}")`, id from org-config, itself read by id anchor `id:{org_config_id}`; on gdrive root-level paths don't resolve for non-Drive-members; OneDrive: read `/org-config.json` and `/members-registry.json` by path instead — a bare `id:` resolves against your own drive there, core standards § Addressing, finding 63) — to `member_hash + email + sharing_identity + display_name`. Admin role is for org members, so the target must be in the registry. If multiple matches, ask the caller to disambiguate. If no match: *"I can't find a member matching `{input}`. Check the spelling, or run `@ai:invite-member` if they're new to the org. Note: a person must already be an org member before they can be made a client-intelligence admin."*

If the target is the caller themselves: *"You can't add yourself as an admin. If you don't currently have admin access, ask an existing admin to grant it to you."*

### Step 3: Apply (permission-changes procedure)

Follow `/agent-index-core/internal/permission-changes.md` with:

```json
{
  "calling_task": "add-admin",
  "purpose": "Make {member.display_name} a client-intelligence admin (Editor on templates/ and config/)",
  "confirmer": "collection_admin",
  "mode": "all_or_nothing",
  "changes": [
    { "op": "share", "resource": "/shared/client-intelligence/templates/", "resource_label": "client-intelligence templates/", "is_folder": true,
      "recipient": "{member.sharing_identity}", "role": "writer", "reason": "Admin role = Editor on templates/ and config/", "source": "member_request" },
    { "op": "share", "resource": "/shared/client-intelligence/config/", "resource_label": "client-intelligence config/", "is_folder": true,
      "recipient": "{member.sharing_identity}", "role": "writer", "reason": "Admin role = Editor on templates/ and config/", "source": "member_request" }
  ]
}
```

- **`confirmer: "collection_admin"`** — the client-intelligence admin role (Editor on `templates/` and `config/`), enforced by the backend: only someone who can already share these folders can complete the run. No org-admin check applies.
- **`mode: all_or_nothing` is intentional.** Granting admin role is a coherent two-folder operation. Half-applied admin role (Editor on `templates/` but not `config/`, or vice versa) leaves the member able to author templates but not edit the default permission policy, or the reverse. Declining either row applies neither.
- **The procedure handles no-ops.** If the target already has `writer` on a folder (explicit or inherited), that row is shown as already in effect. If only one is, say the run is repairing an inconsistent state, not granting fresh.

**Act on the result** (check in this order):

1. **`nothing_to_change: true`** (both folders already in effect; the procedure returns `complete` with every row `skipped_noop`) → *"`{member.display_name}` is already a client-intelligence admin. No permission changes were needed."* Halt. Don't print the Step 4 "is now an admin" message.
2. **`complete`** (both rows `applied`, or one `applied` and one `skipped_noop`) → Step 4.
3. **All declined** (the caller declined a row or didn't confirm; in `all_or_nothing` mode that declines every row) → *"Add-admin cancelled. `{member.display_name}` is not an admin. No permission changes were applied."* Halt.
4. **`failed` with access denied** → *"You don't have permission to grant admin role on client-intelligence. Changing sharing on these folders needs drive-Manager rights, so an org admin who is a drive Manager has to run this. To check who's currently an admin, look at the writers on `/shared/client-intelligence/templates/` (run `@ai:view-permissions` on any client — the admins-group resolution will surface them)."* Halt.
5. **Stopped part-way** (one share applied, the other failed) → the procedure offers to revert the applied one as a new confirmed run. Surface the inconsistent state and recommend accepting the revert, or re-running this task once the cause is fixed (the re-run repairs the missing folder). Halt the success path.
6. **Any other failure** (`failed` for another reason, `verify_mismatch`, `not_attempted`, `unsupported`) → relay the procedure's `summary_lines` verbatim. If nothing was `applied`, say *"No permission changes were applied. `{member.display_name}` is not an admin."* Halt; don't print the Step 4 message.

### Step 4: Confirm to caller

```
`{member.display_name}` ({member.sharing_identity}) is now an admin on client-intelligence.

They have Drive Editor on:
- /shared/client-intelligence/templates/
- /shared/client-intelligence/config/

They can now author templates (@ai:create-template), set the org's default visibility prompt (@ai:edit-default-permissions), and add/remove other admins (@ai:add-admin / @ai:remove-admin).

To revoke admin role later, run @ai:remove-admin {member.display_name}.
```

---

## Directives

### Behavior

Direct. The caller knows whom to promote; the task resolves them and hands both grants to the permission-changes procedure. Be transparent that admin role is two folders' worth of grants, shown together in one table.

If the caller asks "who's currently an admin?" — point at `aifs_get_permissions` on `templates/` as the source of truth. There's no admin list file by design.

### State Management

Not stateful.

### Constraints

- **Never call `aifs_share` outside the core permission-changes procedure.**
- **Never grant admin to the caller themselves.** Step 2 enforces.
- **Never grant admin to someone outside the members registry.** The admin role requires org membership (a caller rule, not a sharing limit).
- **Always pass both folders, in `all_or_nothing` mode.** The procedure skips whichever is already in effect, so both are applied as an atomic pair or the missing one is repaired.
- **Never modify any collection-side file.** Drive Activity is the audit source.

### Edge Cases

- **Target member is in members-registry but not in the all-members Google Group.** They may not actually be a current member. The share will succeed at the Drive level (Drive allows sharing with any address), but they can't use the collection until they're added back to the all-members group at the org level. Surface a note if this is detectable from members-registry.
- **Target already has Editor on one folder but not the other (inconsistent state from a prior failed run).** The procedure shows one row as already in effect and applies only the other. Note that the operation is repairing an inconsistent state, not fresh granting.
- **Both folders already have Editor for the target (idempotent re-run).** The procedure reports nothing to change; halt with the "already an admin" message (result check 1).
- **A share reports success but doesn't show up (`verify_mismatch` after the procedure's rechecks).** Surface as a warning (result check 6) and direct the caller to check `aifs_get_permissions` in a few minutes. Don't auto-retry the share; that risks duplicate grants in some adapter implementations.

---

## V1 Note

Admin role is derived from `templates/` write access. (2.0 note: per-instance ACLs no longer exist — the inherit-limitation caveat that lived here is obsolete. `templates/` and `config/` remain admin-Writer / org-reader; sharing them to a specific member as Writer is a plain additive grant. On a Shared Drive the caller still needs drive-Manager rights to change folder sharing — see the F12 note under About This Task.)
