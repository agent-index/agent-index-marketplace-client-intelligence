---
name: revoke-permission
type: task
version: 1.0.0
collection: client-intelligence
description: Revoke a member's View / Edit / Delete access on a client instance. Symmetric inverse of grant-permission. Cannot revoke the creator's permissions (creator retains all three permanently per the design). Authority is filesystem-enforced via the helper.
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

Revoke Permission removes a member's access on a client instance. Like Grant Permission, the actual unshare goes through the permission-change-helper skill — the agent never calls `aifs_unshare` directly. Current grantees and admins can revoke; others can't.

Critical guardrail: the **creator** of an instance cannot be revoked. The design specifies that the creator retains View, Edit, and Delete permanently as the "creator default" — that's one of the three mechanisms composing initial permissions at instance creation. Attempting to revoke the creator is refused with a clear message.

### Inputs

- **`slug`** (required, interactive or argument) — the instance to revoke on.
- **`member`** (required, interactive) — email or display name of the member to revoke from. Resolved against `members-registry.json`.
- **`permissions`** (required, interactive) — `view`, `edit`, `delete`, any combination, or `all`. `all` revokes everything.

### Outputs

- `aifs_unshare` calls applied via the helper. Drive Activity records the unshare events.
- No collection-side files modified.

Confirmation surfaced to the caller after the helper reports `applied`.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. Halt if not authenticated.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. Halt if false.
- Confirm permission-helper is installed.

### Step 2: Resolve slug

`aifs_exists("/shared/client-intelligence/public-index/instances/{slug}.json")`. Halt if false.

### Step 3: Read public-index entry (for creator check)

`aifs_read("/shared/client-intelligence/public-index/instances/{slug}.json")`. Capture `created_by` (the creator's member_hash). Used in Step 5's guardrail.

### Step 4: Resolve target member

Accept email or display name. Resolve via `members-registry.json` to `member_hash + email`.

If the target is the caller themselves: ask explicitly *"You want to revoke your own access on `{slug}`? You'll lose the ability to view, edit, or further manage this client. Confirm yes/no."*. Allow on confirm — a member may legitimately want to remove themselves.

### Step 5: Creator guardrail

If the resolved target's `member_hash` matches the `created_by` from Step 3: refuse with: *"`{member.display_name}` is the creator of `{slug}`. Creator permissions are permanent — they can't be revoked. If the creator no longer needs ownership, run `@ai:transfer-ownership` (not in V1) or contact your admin to discuss reassignment."*

Halt — do not proceed to build a helper spec.

### Step 6: Collect permissions

Ask which permissions to revoke. Accept `view`, `edit`, `delete`, any combination, or `all`. If `all`, expand to `[view, edit, delete]`.

### Step 7: Capture pre-state

`aifs_get_permissions("/shared/client-intelligence/instances/{slug}/")`. Capture the current ACL.

### Step 8: Filter no-ops

For each requested permission, map to the Drive role (per `grant-permission`'s mapping: `view` → `reader`, `edit` and `delete` → `writer`).

For each (target, role) pair to revoke, check the pre-state. If the target doesn't have the role (neither inherited nor explicit): drop as a no-op.

If the filtered list is empty: *"`{member.display_name}` doesn't have any of the requested permissions on `{slug}`. Nothing to revoke."* Halt.

### Step 9: Build the helper spec

```json
{
  "version": "1.0",
  "operations": [
    {
      "op": "unshare",
      "resource": "/shared/client-intelligence/instances/{slug}/",
      "recipient": "{member.email}",
      "before": {"recipients": <pre_state.permissions>}
    },
    ... (one per role to revoke; typically one operation since view+edit collapses to writer)
  ],
  "context": {
    "requestor": "{caller.member_hash}",
    "calling_task": "revoke-permission",
    "purpose": "Revoke `{member.display_name}` ({permissions joined}) on client `{slug}` ({name})."
  },
  "mode": "fail_soft"
}
```

### Step 10: Invoke the helper

Narrate:

> *"Here's a link to open a review page in your browser. It'll show {N} unshare operation(s) on `{slug}`. Click the link, then click Accept on the review page to apply with your own credentials."*

Invoke `permission-change-helper`.

**Branch on outcome:**

- **`applied`** — verify post-state. Confirm to caller.
- **`rejected`** — *"Revoke cancelled. No permission changes applied."* Halt.
- **`timed_out` / `page_closed`** — same; offer to retry.
- **`partial_failure`** — surface failures; offer to retry just the failed ops.
- **`apply_error`** — surface: *"You don't have permission to revoke access on this client. Existing grantees and admins can revoke."*
- **`binary_not_found`** — halt with install-incomplete message.

### Step 11: Confirm to caller

```
`{member.display_name}` ({member.email}) no longer has {permissions joined} on `{slug}` ({name}).

Drive Activity has recorded the unshare event. To re-grant later, run @ai:grant-permission {slug} {member.email}.
```

---

## Directives

### Behavior

Direct and unambiguous. Revocation is a destructive operation from the target's perspective (they lose access). The narration before the helper invocation should make clear what's about to be revoked from whom.

If the caller asks broad questions ("revoke everyone except Alice"), this isn't the right task — V1 doesn't support bulk revocation. Suggest running `@ai:view-permissions {slug}` to see the list, then running this task per-member.

### State Management

Not stateful.

### Constraints

- **Never revoke the creator.** Step 5 enforces unconditionally.
- **Never call `aifs_unshare` directly.** All ACL changes go through the helper.
- **Never bypass the post-state verification on `applied`.** The helper's apply-script verifies, but task-side verification ensures Drive's eventual-consistency hasn't lagged.
- **Never modify any collection-side file.** Audit comes from Drive Activity.

### Edge Cases

- **Target has the requested role only via inheritance from a parent folder** (e.g., they have Writer inherited from `instances/` via the all-members group). `aifs_unshare` only removes explicit grants, not inherited ones. The unshare succeeds (no explicit grant to remove → no-op or specific behavior depending on adapter), but the target still has the inherited role. Surface this clearly: *"`{member.display_name}` had {role} only via inheritance from the all-members group. The explicit unshare succeeded, but they still have {role} access via inheritance. To truly remove their access, they would need to be removed from the all-members group at the org level."*
- **Target is no longer in the org.** Allow the revoke — useful for cleaning up after a member departs.
- **Caller revokes themselves and they're the only non-creator grantee.** Allowed. The creator retains full access; the caller loses theirs.
- **All requested permissions are no-ops.** Step 8 catches it.
- **Helper returns `applied` but post-state shows the target still has access.** Likely inheritance (see first edge case) or eventual-consistency lag. Surface accordingly.

---

## V1 Limitations

Same inheritance limitation as `create-client` and `grant-permission`: the helper spec v1.0 doesn't pass `inherit: false` through. Revoking via `aifs_unshare` only removes explicit grants — inherited grants from the all-members Writer on `instances/` cannot be removed at the per-instance level until the helper spec extension ships (Phase 5 / `helper-spec-needs-inherit-passthrough` under the umbrella).
