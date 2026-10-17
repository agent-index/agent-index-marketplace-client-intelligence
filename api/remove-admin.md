---
name: remove-admin
type: task
version: 2.1.2
collection: client-intelligence
description: Admin-only task to revoke admin role from a member. Symmetric inverse of add-admin - unshares both /shared/client-intelligence/templates/ and /shared/client-intelligence/config/. Enforces a last-admin guardrail - the task refuses to leave the collection with zero admins. Removals go through the core permission-changes procedure (confirmer collection_admin; caller confirms each; all-or-nothing). Authority is backend-enforced - only existing collection admins can unshare these folders.
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

Remove Admin revokes admin role from a member by unsharing `templates/` and `config/`. The task enforces a critical safety guardrail: the collection must always have at least one admin. If removing the target would leave zero admins, the task refuses.

**Authority is enforced by the backend, not by an org-admin check.** Client-intelligence admins need not be org admins, so the task runs the procedure with `confirmer: "collection_admin"`. Only an existing collection admin can unshare `templates/` and `config/`; for anyone else the backend refuses (the row comes back `failed`, access denied) and the task surfaces the admin-access error. **In practice, on a Google Shared Drive only drive Managers can change folder sharing (audit finding F12)**, so the person running this needs Manager rights on the drive, not just Editor on these folders. If they lack it, the procedure reports the change `failed` with access denied; tell them an org admin (a drive Manager) has to run it.

The task reads `aifs_get_permissions` on both folders for its own purposes: to decide whether the target is an admin at all, to count admins for the last-admin guardrail, and — after the removal — to report what access remains. These reads are purposive, not a generic authority gate (the procedure also reads current access, but only to plan the changes).

### Inputs

- **`member`** (required, interactive) — email or display name of the member to demote. Resolved against `members-registry.json`; if they aren't in the registry (e.g., a departed member who still holds Editor), chosen from the current writers on the two folders.
- **`confirmation`** (required, interactive) — the caller's explicit confirmation of each removal, collected by the permission-changes procedure. This is the only confirmation the task asks for.

### Outputs

The task passes two unshare changes to the core permission-changes procedure, which applies them under the caller's own credentials after the caller confirms each, on these resources:

- `/shared/client-intelligence/templates/` — remove Writer grant for `{target.sharing_identity}`
- `/shared/client-intelligence/config/` — remove Writer grant for `{target.sharing_identity}`

Drive Activity records both unshare events. No collection-side files are modified.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. Halt if not authenticated.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. Halt if false.

### Step 2: Resolve target member

Resolve via `members-registry.json` — read it by id anchor (`aifs_read("id:{resource_ids.members_registry}")`, id from org-config, itself read by id anchor `id:{org_config_id}`; on gdrive root-level paths don't resolve for non-Drive-members; OneDrive: read `/org-config.json` and `/members-registry.json` by path instead — a bare `id:` resolves against your own drive there, core `runtime-reference.md` § "Addressing: Owned Content and Sharing", finding 63) — to `member_hash + email + sharing_identity + display_name`. The target's subject for the rest of the task is `{target.sharing_identity}`, with `source: "member_request"`.

- **Multiple matches** → ask the caller to disambiguate.
- **No registry match** (e.g., a departed member who still holds Editor) → don't guess an address and don't halt yet. After the Step 3 read, list the writers on `templates/` and `config/` that aren't in the registry and ask the caller which one they mean. If they pick one, that subject (exactly as the backend reports it) is the target, with `source: "current_state"`, and its display name is the subject itself. If none matches, halt: *"I can't find `{input}` among the members or the current admins on client-intelligence. Nothing to revoke."*
- **Target is the caller themselves** → surface a notice, not a separate question: *"Heads-up: you're removing your own admin role on client-intelligence. You'll lose the ability to author templates, edit the default permission policy, and manage admins. The permission-change confirmations that follow are the only confirmation."* Continue (subject to the last-admin guardrail in Step 4).

### Step 3: Read current access on both folders (purposive read)

`aifs_get_permissions({path: "/shared/client-intelligence/templates/", include_inherited: true})` and `aifs_get_permissions({path: "/shared/client-intelligence/config/", include_inherited: true})`. Keep two views:

- **Unfiltered** — every permission on each folder, explicit and inherited, individual and group. Used to decide whether the target is an admin at all (Step 4a/4b).
- **Remaining admin set** — from the `templates/` writers, used **only** for the last-admin guardrail count (Step 4c). It counts everyone who will still be able to administer the collection after the removal:
  - **Explicit individual writers**, other than the target.
  - **Individuals with writer inherited from the drive root or another parent** (drive Managers and Content managers, such as the org owner). They keep admin capability however explicit grants change. These used to be excluded, and on a Shared Drive the bootstrap admin usually has *only* inherited access: the core permission-changes procedure skips their bootstrap grant as already in effect. The guardrail then saw the one added admin as "the only admin" and refused every removal (`ciadmincount`, 3.32.0 live test).
  - **Groups** with writer, counted as 1 admin each, **except the all-members group** (org-config `remote_filesystem.connection.all_members_group`), which never makes anyone an admin.

### Step 4: Admin test and last-admin guardrail

Match the target's subject against the **unfiltered** permissions of **both** folders (role `writer` or above):

- **4a. Not an admin** — the target holds no writer-level permission on either folder (explicit or inherited, as far as we can detect): *"`{target.display_name}` isn't currently an admin on client-intelligence. Nothing to revoke."* Halt.
- **4b. Inherited only** — the target holds writer-level access on one or both folders, but no explicit grant on either: *"`{target.display_name}`'s admin role comes from inheritance — typically a drive-level Editor grant from the org owner. To remove it, an org admin would need to adjust the parent grant; this collection-level task can't reach it."* Halt.
- **4c. Last-admin guardrail** — the target holds an explicit grant on at least one folder (a stray explicit grant on `config/` alone counts). If removing the target would leave the **remaining admin set** empty, refuse: *"`{target.display_name}` is the only admin on client-intelligence. Removing them would leave the collection with no admins, which means no one could author templates, edit the default permission policy, or manage permissions. Add another admin first via `@ai:add-admin`, then re-run this task."* Halt.

Otherwise continue to Step 5.

### Step 5: Apply (permission-changes procedure)

Tell the caller what removal means, then follow `/agent-index-core/internal/permission-changes.md`:

> *"Removing `{target.display_name}` as an admin means they lose Editor on `templates/` and `config/`: they can no longer author templates, set the default visibility prompt, or add/remove admins. Their own clients (private or org-public) and any access others granted them are unaffected."*

```json
{
  "calling_task": "remove-admin",
  "purpose": "Remove {target.display_name}'s client-intelligence admin role (Editor on templates/ and config/)",
  "confirmer": "collection_admin",
  "mode": "all_or_nothing",
  "changes": [
    { "op": "unshare", "resource": "/shared/client-intelligence/templates/", "resource_label": "client-intelligence templates/", "is_folder": true,
      "recipient": "{target.sharing_identity}", "reason": "Admin role = Editor on templates/ and config/", "source": "{member_request | current_state}" },
    { "op": "unshare", "resource": "/shared/client-intelligence/config/", "resource_label": "client-intelligence config/", "is_folder": true,
      "recipient": "{target.sharing_identity}", "reason": "Admin role = Editor on templates/ and config/", "source": "{member_request | current_state}" }
  ]
}
```

- `source` is `member_request` for a registry member the caller named, `current_state` for a subject picked from the Step 3 writers.
- The procedure's confirmation **is** this task's confirmation — there is no separate yes/no step, including for self-removal. Removals are high-impact, so they're confirmed one at a time.
- **`all_or_nothing`:** declining either removal applies neither. Admin role is coherent across both folders, and a partial revoke would leave the member in an inconsistent state.
- **The procedure handles explicit-only grants.** A folder where the target has no explicit grant (only inherited) shows as "inherited … can't be removed here" and is skipped.

**Act on the result:**

- **`complete`** → re-read `aifs_get_permissions({path, include_inherited: true})` on **both** folders (a fresh read; don't infer from the rows you intended to remove).
  - The target holds no writer-level access on either folder → Step 6.
  - The target still holds Editor on either folder (inheritance from a parent grant, or a group) → *"The explicit Editor grants were removed, but `{target.display_name}` still has Editor on `{folders}` via inheritance from a parent grant or group. To truly remove their admin role, an org admin needs to adjust that grant."* Halt; don't print Step 6.
- **All declined** (any row declined or the table not confirmed; in `all_or_nothing` mode that declines every row) → *"Remove-admin cancelled. `{target.display_name}` remains an admin. No permission changes applied."* Halt.
- **`failed` with access denied** → *"You don't have permission to revoke admin role. Changing sharing on these folders needs drive-Manager rights, so an org admin who is a drive Manager has to run this."* Halt.
- **Any other failure** (stopped part-way with one removal applied, `failed` for another reason, `verify_mismatch`, `not_attempted`) → relay the procedure's `summary_lines`. Tell the caller the admin role may be half-removed, and recommend **re-running this task to finish the removal** once the cause is fixed (the re-run skips whatever is already gone). If the procedure made a `revert_offer`, present it as the alternative (restore the removed grant as a new confirmed run). Halt the success path; don't print Step 6.

### Step 6: Confirm to caller

```
`{target.display_name}` ({target.sharing_identity}) is no longer an admin on client-intelligence.

Drive Activity has recorded the unshare events. Their own clients and any grants others made to them are unaffected.

To re-add them later, run @ai:add-admin {target.display_name}.
```

(Omit the re-add line for a target picked from current state who isn't in the registry — add-admin requires org membership.)

---

## Directives

### Behavior

Careful. Admin removal is consequential. The last-admin guardrail and the per-removal confirmation are deliberate friction. Don't bypass them, and don't add a second confirmation on top of them.

If the caller wants to remove the only admin "but it's fine, we have a workspace admin who can fix it" — politely refuse. The collection-level guardrail is independent of the workspace level; it's there to keep this collection self-sufficient.

### State Management

Not stateful.

### Constraints

- **Never bypass the last-admin guardrail.** Step 4c is unconditional.
- **Never call `aifs_unshare` outside the core permission-changes procedure.** Its per-removal confirmation is required; never proceed without it.
- **Never guess a target's address.** Registry member → `sharing_identity`; otherwise a subject the caller picked from the current writers.
- **Never report "no longer an admin" from the intended changes.** Step 6 follows only a fresh `aifs_get_permissions` read showing no Editor remains.
- **Never modify any collection-side file.** Drive Activity is the audit source.

### Edge Cases

- **Target has Editor only via inheritance (no explicit grant on `templates/` or `config/`).** Step 4b halts with the inheritance explanation.
- **Target has a stray explicit Editor grant on `config/` but not `templates/`.** Step 4 still treats them as an admin (both folders are tested unfiltered); the `templates/` row is skipped as nothing to remove and the `config/` grant is removed.
- **Target is a departed member no longer in the registry who still holds Editor.** Step 2 offers the non-registry writers from Step 3; the chosen subject is removed with `source: "current_state"`.
- **Target is the caller and they're the only admin.** Step 4c refuses unconditionally — even self-removal can't bypass the guardrail.
- **Caller invokes with the same target twice in quick succession (idempotency).** First call removes the grants; second call's Step 4a reports "not currently an admin" and halts cleanly.
- **Group membership defines admin status (e.g., admins are an admins-group; the target is in the group).** Removing the target's individual Editor grant doesn't change their admin status if they're an admin via the group. The post-removal re-read can't see group membership, so surface the group note whenever a group holds Editor on either folder.

---

## V1 Note

Same as `add-admin`: the inheritance limitation that affects per-instance tasks doesn't apply here, because `templates/` and `config/` have no permissive *member* parent grant (the all-members group only reads them). They do inherit writer for drive Managers on a Shared Drive. That's why those Managers count as remaining admins in Step 3 and are reported by Step 4b as inheritance-only when targeted.
