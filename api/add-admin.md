---
name: add-admin
type: task
version: 2.0.0
collection: client-intelligence
description: Admin-only task to grant admin role to another member of the org. Admin role is derived from the filesystem - a collection admin is any member with Drive Editor access to /shared/client-intelligence/templates/ and /shared/client-intelligence/config/. This task batches two share operations (one per folder) into a single permission-change-helper invocation; the caller reviews and accepts both at once. Try-and-catch authority - only existing admins can complete the operation.
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

Add Admin grants admin role on the client-intelligence collection to another org member. The collection's admin model is derived from filesystem state: admins are members with Drive Editor access to `templates/` and `config/`. Granting admin means sharing both folders with Editor. The task batches both shares into one permission-change-helper invocation so the caller reviews and accepts a single, coherent permission change.

Authority is filesystem-enforced. The caller must already be an admin to grant admin role to someone else; otherwise the helper's apply-script returns `apply_error` and the task surfaces the admin-access error.

### Inputs

- **`member`** (required, interactive) — email or display name of the member to promote to admin. Resolved against `members-registry.json`.

### Outputs

The task builds a permission-change-helper spec containing two share operations and invokes the helper. The helper's apply-script (running under the caller's OAuth token, not the agent's) performs the underlying share calls on these resources:

- `/shared/client-intelligence/templates/` — Writer for `{member.email}`
- `/shared/client-intelligence/config/` — Writer for `{member.email}`

Drive Activity records both share events. No collection-side files are modified.

Confirmation surfaced to the caller after the helper reports `applied`.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. Halt if not authenticated.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. Halt if false.
- Confirm permission-helper is installed.

### Step 2: Resolve target member

Accept email or display name. Resolve via `members-registry.json` to `member_hash + email + display_name`. If multiple matches, ask the caller to disambiguate. If no match: *"I can't find a member matching `{input}`. Check the spelling, or run `@ai:invite-member` if they're new to the org. Note: a person must already be an org member before they can be made a client-intelligence admin."*

If the target is the caller themselves: *"You can't add yourself as an admin. If you don't currently have admin access, ask an existing admin to grant it to you."*

### Step 3: Capture pre-state

Read current ACLs on both target folders:

- `aifs_get_permissions("/shared/client-intelligence/templates/")` — capture as `templates_pre`.
- `aifs_get_permissions("/shared/client-intelligence/config/")` — capture as `config_pre`.

### Step 4: Filter no-ops

For each (resource, target) pair:
- Look up the target in the pre-state.
- If the target already has `writer` (explicit or inherited): drop as a no-op.

If both operations are no-ops: *"`{member.display_name}` is already a client-intelligence admin (has Drive Editor on both `templates/` and `config/`). No changes needed."* Halt.

If only one is a no-op: include only the non-no-op operation. Surface a note: *"`{member.display_name}` already has Editor on `{folder_that_is_noop}`; only the share on `{other_folder}` needs to be applied."*

### Step 5: Build the helper spec

Typically two operations:

```json
{
  "version": "1.0",
  "operations": [
    {
      "op": "share",
      "resource": "/shared/client-intelligence/templates/",
      "recipient": "{member.email}",
      "role": "writer",
      "before": {"recipients": <templates_pre.permissions>}
    },
    {
      "op": "share",
      "resource": "/shared/client-intelligence/config/",
      "recipient": "{member.email}",
      "role": "writer",
      "before": {"recipients": <config_pre.permissions>}
    }
  ],
  "context": {
    "requestor": "{caller.member_hash}",
    "calling_task": "add-admin",
    "purpose": "Grant `{member.display_name}` ({member.email}) admin role on client-intelligence. Both `templates/` and `config/` will be shared with them as Drive Editor; both are required for the role to function."
  },
  "mode": "all_or_nothing"
}
```

`mode: all_or_nothing` is intentional — granting admin role is a coherent two-folder operation. Half-applied admin role (Editor on `templates/` but not `config/`, or vice versa) leaves the member able to author templates but not edit the default permission policy, or vice versa. The atomic semantics prevent that degradation.

### Step 6: Invoke the helper

Narrate to the caller:

> *"Here's a link to open a review page in your browser. It'll show 2 share operations — granting `{member.display_name}` Editor on `templates/` and `config/`. Both folders together constitute admin role. Click the link, then click Accept on the review page to apply with your own credentials."*

Invoke `permission-change-helper`.

**Branch on outcome:**

- **`applied`** — both shares succeeded. Verify post-state (Step 7).
- **`rejected`** — *"Add-admin cancelled. `{member.display_name}` is not an admin. No permission changes were applied."* Halt.
- **`timed_out` / `page_closed`** — same as rejected; offer to retry.
- **`partial_failure`** — with `mode: all_or_nothing`, the apply-script should not leave partial state; both ops apply or neither does. If `partial_failure` is reported anyway (e.g., one succeeded before the other failed and rollback couldn't undo), surface the partial state and recommend that the caller manually run `@ai:remove-admin` against the same member to clean up, or re-run this task. Halt the success path.
- **`apply_error`** — surface: *"You don't have permission to grant admin role on client-intelligence. Only existing admins can add new ones. To check who's currently an admin, look at the writers on `/shared/client-intelligence/templates/` (run `@ai:view-permissions` on any client — the admins-group resolution will surface them)."*
- **`binary_not_found`** — halt with install-incomplete message.

### Step 7: Verify post-state (on `applied` only)

- `aifs_get_permissions("/shared/client-intelligence/templates/")` — confirm the target now has Editor.
- `aifs_get_permissions("/shared/client-intelligence/config/")` — confirm the target now has Editor.

If either verification reports the target without Editor, retry once after a 5-second pause (Drive eventual-consistency). If still missing, surface a warning to the caller; the helper reported `applied` so the writes succeeded, but Drive's visible state may catch up shortly. Recommend re-running `@ai:view-permissions` on a client a few moments later to confirm admin status propagated.

### Step 8: Confirm to caller

```
`{member.display_name}` ({member.email}) is now an admin on client-intelligence.

They have Drive Editor on:
- /shared/client-intelligence/templates/
- /shared/client-intelligence/config/

They can now author templates (@ai:create-template), set the org's default visibility prompt (@ai:edit-default-permissions), and add/remove other admins (@ai:add-admin / @ai:remove-admin).

To revoke admin role later, run @ai:remove-admin {member.email}.
```

---

## Directives

### Behavior

Direct. The caller knows whom to promote; the task collects, builds the spec, invokes the helper. Be transparent that admin role is two folders' worth of grants and that the helper will show both operations on the review page.

If the caller asks "who's currently an admin?" — point at `aifs_get_permissions` on `templates/` as the source of truth. There's no admin list file by design.

### State Management

Not stateful.

### Constraints

- **Never call `aifs_share` directly.** Helper only.
- **Never grant admin to the caller themselves.** Step 2 enforces.
- **Never include just one folder in the spec when both are needed.** `all_or_nothing` mode + Step 4's filtering work together: if one is a no-op, include only the other; if both are needed, include both as an atomic pair.
- **Never modify any collection-side file.** Drive Activity is the audit source.

### Edge Cases

- **Target member is in members-registry but not in the all-members Google Group.** They may not actually be a current member. The helper share will succeed at the Drive level (Drive allows sharing with any email), but they can't use the collection until they're added back to the all-members group at the org level. Surface a note if this is detectable from members-registry.
- **Target already has Editor on one folder but not the other (inconsistent state from a prior failed run).** Step 4 produces a single-operation spec to bring them to the consistent state. The narration in Step 6 should note that the operation is repairing an inconsistent state, not fresh granting.
- **Both folders already have Editor for the target (idempotent re-run).** Step 4 catches it; halt with "already an admin" message.
- **Helper succeeds but verification (Step 7) fails consistently.** Surface as a warning and direct the caller to check `aifs_get_permissions` manually in a few minutes. Don't auto-retry the share — that risks duplicate grants in some adapter implementations.

---

## V1 Note

Admin role is derived from `templates/` write access. (2.0 note: per-instance ACLs no longer exist — the inherit-limitation caveat that lived here is obsolete. `templates/` and `config/` remain admin-Writer / org-reader; sharing them to a specific member as Writer is a plain additive grant. The caller — an existing admin with Manager rights — Accepts, so F12 does not apply to THIS task.)
