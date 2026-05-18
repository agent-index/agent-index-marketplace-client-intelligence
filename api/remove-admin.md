---
name: remove-admin
type: task
version: 1.0.0
collection: client-intelligence
description: Admin-only task to revoke admin role from a member. Symmetric inverse of add-admin - unshares both /shared/client-intelligence/templates/ and /shared/client-intelligence/config/. Enforces a last-admin guardrail - the task refuses to leave the collection with zero admins. Uses permission-change-helper.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills:
    - permission-change-helper
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter contract 2.0.0 or higher)
  - Permission-change-helper binary (Go) or Node fallback present at mcp-servers/permission-helper-*
reads_from: "/shared/client-intelligence/"
writes_to: null
---

## About This Task

Remove Admin revokes admin role from a member by unsharing `templates/` and `config/`. The task enforces a critical safety guardrail: the collection must always have at least one admin. If removing the target would leave zero admins, the task refuses.

The last-admin check is the one place this task uses `aifs_get_permissions` for guardrail purposes (not just for the helper-spec `before` field). The read is purposive — it answers the question "how many admins are there right now?" — not a generic authority gate.

### Inputs

- **`member`** (required, interactive) — email or display name of the member to demote. Resolved against `members-registry.json`.
- **`confirmation`** (required, interactive) — explicit confirmation before the helper invocation.

### Outputs

The task builds a permission-change-helper spec containing two unshare operations and invokes the helper. The helper's apply-script (running under the caller's OAuth token, not the agent's) performs the underlying unshare calls on these resources:

- `/shared/client-intelligence/templates/` — remove Writer grant for `{member.email}`
- `/shared/client-intelligence/config/` — remove Writer grant for `{member.email}`

Drive Activity records both unshare events. No collection-side files are modified.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. Halt if not authenticated.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. Halt if false.
- Confirm permission-helper is installed.

### Step 2: Resolve target member

Resolve via `members-registry.json`. If the target is the caller themselves: ask explicitly *"You want to remove your own admin role on client-intelligence? You'll lose the ability to author templates, edit the default permission policy, and manage admins. Confirm yes/no."*. Allow on confirm (subject to the last-admin guardrail in Step 4).

### Step 3: Read current admin set (purposive read)

`aifs_get_permissions("/shared/client-intelligence/templates/")` — the writers on this folder are the current admin set.

Extract the list of subjects with role `writer`. Filter out:
- Inherited writers from the parent drive root (those are members with drive-level Editor, like the org owner; they're admins through inheritance rather than explicit grant on templates/).
- Group subjects whose membership we can't enumerate locally (e.g., admins-group@...). For safety, count groups as "1 admin" each — removing an explicit individual is still safe if a group has admin coverage.

Result: a list of `(subject, role, source)` rows representing the current admin set.

### Step 4: Last-admin guardrail

If removing the target would leave zero admins (after the filtering in Step 3), refuse: *"`{member.display_name}` is the only admin on client-intelligence. Removing them would leave the collection with no admins, which means no one could author templates, edit the default permission policy, or manage permissions. Add another admin first via `@ai:add-admin`, then re-run this task."*

If the target isn't currently an admin (their email isn't in the writers list on `templates/`, even via group membership we can detect): *"`{member.display_name}` isn't currently an admin on client-intelligence. Nothing to revoke."*

Halt in both cases.

### Step 5: Confirmation

> *"Remove `{member.display_name}` ({member.email}) as an admin on client-intelligence? They'll lose Editor access to `templates/` and `config/`, which means they can no longer author templates, edit the default permission policy, or add/remove admins. Their access to specific client instances (via grant-permission) is unaffected."*

Wait for explicit yes/no.

### Step 6: Capture pre-state for the helper

- `aifs_get_permissions("/shared/client-intelligence/templates/")` — `templates_pre`.
- `aifs_get_permissions("/shared/client-intelligence/config/")` — `config_pre`.

### Step 7: Filter no-ops

For each (resource, target) pair:
- If the target doesn't have an explicit grant on the resource (only inherited): drop. `aifs_unshare` removes explicit grants only.

If both ops are no-ops: *"`{member.display_name}` doesn't have explicit Editor grants on `templates/` or `config/`. Their admin role (if any) comes from inheritance — typically a drive-level Editor grant from the org owner. To remove admin via this path, an org admin would need to adjust the parent grant; this collection-level task can't reach it."*. Halt.

### Step 8: Build the helper spec

```json
{
  "version": "1.0",
  "operations": [
    {
      "op": "unshare",
      "resource": "/shared/client-intelligence/templates/",
      "recipient": "{member.email}",
      "before": {"recipients": <templates_pre.permissions>}
    },
    {
      "op": "unshare",
      "resource": "/shared/client-intelligence/config/",
      "recipient": "{member.email}",
      "before": {"recipients": <config_pre.permissions>}
    }
  ],
  "context": {
    "requestor": "{caller.member_hash}",
    "calling_task": "remove-admin",
    "purpose": "Revoke admin role from `{member.display_name}` ({member.email}) on client-intelligence. Both `templates/` and `config/` Editor grants will be removed."
  },
  "mode": "all_or_nothing"
}
```

`all_or_nothing` — admin role is coherent across both folders; partial revoke leaves the member in an inconsistent state.

### Step 9: Invoke the helper

Narrate:

> *"Here's a link to open a review page in your browser. It'll show 2 unshare operations — revoking `{member.display_name}`'s Editor on `templates/` and `config/`. Click the link, then click Accept on the review page to apply with your own credentials."*

Invoke `permission-change-helper`.

**Branch on outcome:**

- **`applied`** — both unshares succeeded. Verify post-state (Step 10).
- **`rejected`** — *"Remove-admin cancelled. `{member.display_name}` remains an admin. No permission changes applied."* Halt.
- **`timed_out` / `page_closed`** — same as rejected; offer to retry.
- **`partial_failure`** — with `all_or_nothing`, this shouldn't happen. If it does, surface and recommend a follow-up `@ai:add-admin` or `@ai:remove-admin` to clean up.
- **`apply_error`** — surface the admin-access error: *"You don't have permission to revoke admin role. Only existing admins can do that."*
- **`binary_not_found`** — halt with install-incomplete.

### Step 10: Verify post-state (on `applied` only)

- `aifs_get_permissions("/shared/client-intelligence/templates/")` — confirm the target is no longer a writer.
- `aifs_get_permissions("/shared/client-intelligence/config/")` — same.

Note: if the target had Editor via inheritance (e.g., drive owner), the unshare succeeded but they still have effective Editor via inheritance. Surface this clearly: *"The explicit Editor grants were removed, but `{member.display_name}` still has Editor on `templates/` and `config/` via inheritance from a parent grant. To truly remove their admin role, an org admin needs to adjust the parent grant."*

### Step 11: Confirm to caller

```
`{member.display_name}` ({member.email}) is no longer an admin on client-intelligence.

Drive Activity has recorded the unshare events. Their access to specific clients (via grant-permission) is unaffected.

To re-add them later, run @ai:add-admin {member.email}.
```

---

## Directives

### Behavior

Careful. Admin removal is consequential. The last-admin guardrail and the explicit yes/no confirmation are deliberate friction. Don't bypass them.

If the caller wants to remove the only admin "but it's fine, we have a workspace admin who can fix it" — politely refuse. The collection-level guardrail is independent of the workspace level; it's there to keep this collection self-sufficient.

### State Management

Not stateful.

### Constraints

- **Never bypass the last-admin guardrail.** Step 4 is unconditional.
- **Never call `aifs_unshare` directly.** Helper only.
- **Never proceed past Step 5 without explicit confirmation.**
- **Never modify any collection-side file.** Drive Activity is the audit source.

### Edge Cases

- **Target has Editor only via inheritance (no explicit grant on `templates/` or `config/`).** Step 7 catches it; halt with the inheritance explanation.
- **Target is the caller and they're the only admin.** Step 4 refuses unconditionally — even self-removal can't bypass the guardrail.
- **Caller invokes with the same target twice in quick succession (idempotency).** First call removes the grants; second call's Step 4 reports "not currently an admin" and halts cleanly.
- **Group membership defines admin status (e.g., admins are an admins-group; the target is in the group).** Removing the target's individual Editor grant doesn't change their admin status if they're an admin via the group. Step 10's verification will show they still have Editor; surface the group-inheritance note.

---

## V1 Note

Same as `add-admin`: the inheritance limitation that affects per-instance tasks doesn't apply here. `templates/` and `config/` don't have a permissive parent grant; explicit shares on those folders narrow correctly under V1's helper spec.
