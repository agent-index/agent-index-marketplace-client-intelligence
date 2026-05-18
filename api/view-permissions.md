---
name: view-permissions
type: task
version: 1.0.0
collection: client-intelligence
description: Read-only task that lists who has what permissions on a client instance. Calls aifs_get_permissions directly (this is the read op's primary purpose, not a guard). Surfaces the per-member ACL as a structured table.
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

View Permissions shows the current ACL on a client instance. For each subject (individual member or group), it reports the Drive role (`reader`, `commenter`, `writer`) and whether the grant is explicit on the instance folder or inherited from a parent.

This is the only V1 task where the agent calls `aifs_get_permissions` as its primary purpose rather than as a precondition check. The read is the function; the task's job is to format the result for human consumption.

Note: permission **history** ("when did Sarah get view? who granted it?") is NOT part of this task. That's the `view-client-audit` query, which is deferred to post-V1 pending the access-control `aifs_get_audit` operation. This task surfaces only the current state.

### Inputs

- **`slug`** (required, interactive or argument) — the instance to inspect.

### Outputs

A formatted table of permissions surfaced to the caller. For each subject:
- Email or group address
- Drive role (reader / commenter / writer)
- Source: explicit on this instance / inherited from parent (with the parent path)
- Permission ID (for diagnostics)

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. Halt if not authenticated.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. Halt if false.

### Step 2: Resolve slug

`aifs_exists("/shared/client-intelligence/public-index/instances/{slug}.json")`. Halt if false with the standard not-found message.

### Step 3: Read public-index entry (for context)

`aifs_read("/shared/client-intelligence/public-index/instances/{slug}.json")`. Capture name, status, template_slug, created_by — used for the header in Step 5's output.

### Step 4: Read permissions

`aifs_get_permissions("/shared/client-intelligence/instances/{slug}/")`. Capture the full ACL.

If the call returns permission-denied: surface *"You don't have permission to view the ACL on `{slug}`. This usually means you don't have any access on the instance. Run `@ai:view-client {slug}` to confirm — if you only see the name (visibility-floor view), you don't have view access either. Ask an existing grantee or admin to grant you via `@ai:grant-permission`."* Halt.

If the call returns any other error, surface verbatim.

### Step 5: Translate Drive roles to client-intelligence concepts

The collection-layer permission model has three permissions: View, Edit, Delete. The Drive role mapping:

- Drive `reader` → client-intelligence `view`
- Drive `writer` → client-intelligence `view + edit + delete` (V1 collapses; future versions may distinguish)
- Drive `commenter` → client-intelligence `view` (commenter ~= reader for our purposes; surface as `view` with a footnote)

For each row in the ACL, compute the equivalent client-intelligence permissions.

### Step 6: Render the table

```
# Permissions on `{name}` ({slug})

Status: {active | archived}
Created by: {created_by_display_name} (creator — has view/edit/delete permanently)

| Subject | View | Edit | Delete | Source |
|---|---|---|---|---|
| bill@agent-index.ai | ✓ | ✓ | ✓ | explicit (creator) |
| alice@example.com | ✓ |   |   | explicit |
| all@agent-index.ai | ✓ | ✓ | ✓ | inherited from /shared/client-intelligence/instances/ |
| admins-group@... | ✓ |   |   | explicit |
```

Annotate the creator row with `(creator)` and note that creator permissions are permanent (can't be revoked).

If any grants are inherited from a parent folder, surface a note at the bottom:

> *"Note: {N} grant(s) on this client come from a parent folder's ACL, not from explicit shares on this instance. Until access-control Phase 5 ships the helper-spec extension for `inherit: false`, the all-members group has Writer inherited from `/shared/client-intelligence/instances/`, which means every collection member effectively has full access on every instance regardless of explicit grants. The visibility floor on names works (see `list-clients`); the visibility floor on data is degraded in V1. The codename pattern is the operational confidentiality mechanism."*

End with next-action suggestions:

> *"To grant additional access: `@ai:grant-permission {slug}`. To revoke: `@ai:revoke-permission {slug} {member}`. Note: the creator's permissions are permanent and cannot be revoked."*

---

## Directives

### Behavior

Pure read. The output is informational and explicitly notes the V1 inheritance limitation so callers understand what the displayed permissions mean operationally.

If the caller asks follow-up questions about history ("when did Alice get access?"), explain that V1 doesn't surface permission history — Drive Activity has the events but no V1 task queries them. Point at `view-client-audit` as a post-V1 capability.

### State Management

Not stateful.

### Constraints

- **Never write anything.** Read-only.
- **Never call `aifs_share`, `aifs_unshare`, or `aifs_transfer_ownership`.** This task displays state, doesn't change it.
- **Always include the V1 inheritance note when inherited grants are present** in the ACL. Hiding the note would obscure the data-visibility-floor limitation.
- **Always mark the creator row** with the permanence indicator. The Step 5 logic identifies the creator from the public-index entry's `created_by` field.

### Edge Cases

- **The caller has no access on the instance.** Step 4's permission-denied catches it; surface the standard guidance to request access.
- **The ACL contains a subject not in the members-registry.** Display as raw email/group address with a footnote: *"Subject `{subject}` isn't in the org's members registry. This may be an external Drive share, a Google Group, or a stale grant."*
- **The instance folder has been deleted but the public-index entry remains** (rare data-integrity case). Step 4's `aifs_get_permissions` returns path-not-found. Surface a data-integrity warning and exit.
- **The ACL is empty.** Shouldn't happen — the creator-default grant should always be in place — but if it is, surface a warning: *"No permissions are recorded for `{slug}`. This is a data-integrity issue; the creator should have permanent access. Contact your org admin."*
