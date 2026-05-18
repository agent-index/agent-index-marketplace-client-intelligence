---
name: grant-permission
type: task
version: 1.0.0
collection: client-intelligence
description: Grant View / Edit / Delete permission on a client instance to a member. Authority is filesystem-enforced — current grantees and admins can grant; others cannot. The actual share goes through the permission-change-helper skill (the agent never calls aifs_share directly).
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

Grant Permission adds a member's access to a client instance. The caller picks the target member, picks which permissions to grant (any combination of view / edit / delete), and the actual share is applied through the permission-change-helper skill — the helper renders a review page in the caller's browser, the caller clicks Accept, and the apply-script uses the caller's own OAuth token to call `aifs_share`. The agent never makes the privileged call directly.

Authority is filesystem-enforced. Current grantees on the instance can re-grant (Drive's normal sharing semantics); admins (writers on `templates/`) can grant too. Others get a permission-denied response from the helper's apply-script and the task surfaces a clear error.

### Inputs

- **`slug`** (required, interactive or argument) — the instance to grant on.
- **`member`** (required, interactive) — email or display name of the member to grant to. Resolved against `members-registry.json`.
- **`permissions`** (required, interactive) — any combination of `view`, `edit`, `delete`.

### Outputs

- `aifs_share` calls applied via the helper. Drive Activity records the share events.
- No collection-side files modified.

Confirmation surfaced to the caller after the helper reports `applied`.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. Halt if not authenticated.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. Halt if false.
- Confirm permission-helper is installed (Go or Node fallback). Halt with `@ai:update` guidance if not.

### Step 2: Resolve slug

Accept slug or name. Validate via `aifs_exists("/shared/client-intelligence/public-index/instances/{slug}.json")`. Halt if the client doesn't exist.

### Step 3: Resolve target member

Accept email or display name. Read `members-registry.json` to resolve to `member_hash + email`. If multiple matches, ask the caller to disambiguate. If no match: *"I can't find a member matching `{input}`. Run `@ai:invite-member` if they're new to the org, or check the spelling."*

If the target is the caller themselves: refuse. *"You already have whatever access you have via the existing grants. Grant-permission is for granting access to others."*

### Step 4: Collect permissions

Ask which permissions to grant. Accept `view`, `edit`, `delete` in any combination. The caller can say "view + edit", "all", "just view", etc. Parse accordingly.

### Step 5: Capture pre-state

`aifs_get_permissions("/shared/client-intelligence/instances/{slug}/")`. Capture the current ACL — used for the `before` field in the helper spec and for filtering no-ops.

### Step 6: Filter no-ops

For each requested permission, map to the corresponding Drive role:
- `view` → `reader`
- `edit` → `writer` (Drive doesn't separately model edit-without-delete)
- `delete` → `writer` (collection-layer enforced; Drive Writer is the FS-level grant)

Note: `edit` and `delete` both map to `writer` on Drive. If both are requested, only one `share` operation is needed. The collection-layer permission model (recording that the caller has BOTH edit and delete) requires a separate metadata layer, which V1 does not implement — instead, V1 trusts the Drive Writer grant as covering both. Future versions may add a per-instance `permissions.json` artifact to record finer-grained intent.

For each (target, role) pair, check the pre-state. If the target already has the role (via inheritance or explicit grant): drop as a no-op.

If the filtered list is empty: *"All requested permissions are already in place for `{member.display_name}` on `{slug}`. No changes needed."* Halt.

### Step 7: Build the helper spec

```json
{
  "version": "1.0",
  "operations": [
    {
      "op": "share",
      "resource": "/shared/client-intelligence/instances/{slug}/",
      "recipient": "{member.email}",
      "role": "{drive_role}",
      "before": {"recipients": <pre_state.permissions>}
    }
  ],
  "context": {
    "requestor": "{caller.member_hash}",
    "calling_task": "grant-permission",
    "purpose": "Grant `{member.display_name}` ({permissions joined}) on client `{slug}` ({name})."
  },
  "mode": "fail_soft"
}
```

(Typically a single operation. If view + edit are both requested, that's still one operation with role `writer`, since writer implies read.)

### Step 8: Invoke the helper

Narrate to the caller:

> *"Here's a link to open a review page in your browser. It'll show 1 share operation on `{slug}`. Click the link, then click Accept on the review page to apply it with your own credentials."*

Invoke `permission-change-helper` with the spec.

**Branch on outcome:**

- **`applied`** — verify post-state via `aifs_get_permissions`. Confirm to caller (Step 9).
- **`rejected`** — *"Grant cancelled. No permission changes applied."* Halt.
- **`timed_out` / `page_closed`** — same as rejected; offer to retry.
- **`partial_failure`** — surface failures. With a single operation, this is essentially `apply_error`.
- **`apply_error`** — surface verbatim. Typical cause: caller lacks Drive sharing authority on the folder. Surface: *"You don't have permission to grant access on this client. Existing grantees and admins can grant; check `@ai:view-permissions {slug}` to see who currently has access."*
- **`binary_not_found`** — halt with the install-incomplete message from Step 1's pre-check (rare here).

### Step 9: Confirm to caller

```
`{member.display_name}` ({member.email}) now has {permissions joined} on `{slug}` ({name}).

Drive Activity has recorded the share event. To revoke later, run @ai:revoke-permission {slug} {member.email}.
```

---

## Directives

### Behavior

Single-purpose and direct. The caller knows who they want to grant to and what; the task collects those, builds the helper spec, and invokes. If the caller is uncertain, suggest `@ai:view-permissions {slug}` first to see who currently has access.

When mapping permissions to Drive roles, be transparent about the edit/delete collapse: if the caller asks "what's the difference between edit and delete?" explain that V1 maps both to Drive Writer, with the distinction being enforced at the task layer (delete-client checks for the recorded delete permission). Future versions may add finer granularity.

### State Management

Not stateful. The interview is in-memory; the helper invocation and post-state verification are real-time.

### Constraints

- **Never call `aifs_share` directly.** All ACL changes go through the helper.
- **Never grant to the caller themselves.** Step 3 enforces.
- **Never bypass the helper outcome branch.** Continue only on `applied`.
- **Never modify any collection-side file.** Drive Activity is the audit source; no local mirror.

### Edge Cases

- **Target member is in the org but doesn't have a member_hash recorded yet.** Run `@ai:invite-member` first (referral) or halt with that suggestion. V1 doesn't auto-invite.
- **Target member is no longer in the org (revoked).** Surface: *"`{member.email}` is in the registry but isn't a current member. They've been removed. The grant would still apply at the Drive level, but they wouldn't be able to use it."* Allow if the caller confirms; useful for transition cases.
- **Caller requests `view + edit + delete` but already has `view` and the target has `writer` inherited.** Step 6 filters out the no-op `view` (target already has read access); the `writer` grant is also a no-op (target already has writer via inheritance). Result: empty spec; halt at Step 6.
- **Helper succeeds but post-state verification fails.** Likely Drive eventual-consistency lag. Retry the verification once with a 5-second pause; if still missing, surface a warning and trust the helper's report.

---

## V1 Limitations

Same as `create-client`: the helper spec v1.0 doesn't pass `inherit: false` through to `aifs_share`. The grant operates on top of the inherited all-members Writer from `instances/`. The grant is meaningful for audit (Drive Activity records it) and for the case where collection membership later restricts inheritance, but in V1 the all-members group has Writer access to every instance regardless. See `helper-spec-needs-inherit-passthrough` under the `builder-profile-adaptive-dev-process` umbrella in core-improvements.
