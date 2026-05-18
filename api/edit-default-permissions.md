---
name: edit-default-permissions
type: task
version: 1.0.0
collection: client-intelligence
description: Admin-only task to edit /shared/client-intelligence/config/default-permissions.json - the collection-level default permission policy applied to every newly created client instance. Supports three preset shapes (admin-only, open, closed) or a custom grants array. Revision-aware write with try-and-catch authority enforcement.
stateful: false
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter contract 2.0.0 or higher)
reads_from: "/shared/client-intelligence/config/"
writes_to: "/shared/client-intelligence/config/"
---

## About This Task

Edit Default Permissions modifies the collection-level default permission policy. The policy is read by `create-client` at every instance creation to determine which grants to apply automatically on top of the creator's default permissions.

The task supports three preset shapes (the same presets the install-time setup offered) or a custom grants array. Authority is filesystem-enforced — only members with Drive Editor access to `/shared/client-intelligence/config/` (i.e., admins) can complete the write.

This task is NOT a permission-modifying task in the sense the helper guards against — it edits a policy file, not Drive ACLs. Direct `aifs_write` is appropriate; the helper isn't invoked.

### Inputs

- **`shape`** (required, interactive) — one of:
  - `admin-only` preset — `[{target: "group:admins", permissions: ["view"]}]`
  - `open` preset — `[{target: "group:all-members", permissions: ["view"]}]`
  - `closed` preset — `[]`
  - `custom` — caller provides a custom `grants` array (collected interactively).

### Outputs

- `/shared/client-intelligence/config/default-permissions.json` — updated (revision-aware).

The new policy takes effect for every new client created from this point forward. Existing clients are unaffected — their already-applied initial permissions persist as-is.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. Halt if not authenticated.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. Halt if false.

### Step 2: Read current policy

- `aifs_stat("/shared/client-intelligence/config/default-permissions.json")` — capture the current revision.
- `aifs_read(...)` — parse. Capture current `grants` and other metadata.

If the read fails with `permission-denied`, halt with: *"You don't have access to read the default permission policy. This usually means you don't have admin role on client-intelligence — only admins can read or edit the policy."*

If the read fails with `path-not-found`, the policy file is missing. Surface a data-integrity warning: *"`config/default-permissions.json` is missing. The install may be incomplete or corrupted. Run the repair install path (re-run `@ai:install-collection` against client-intelligence)."*

### Step 3: Show current policy

```
Current default permission policy:
- Preset (recorded): {preset_name_if_any | "custom"}
- Grants (resolved):
  - target: group:admins, permissions: [view]
  - target: ...
- Last updated: {ISO_TIMESTAMP}
```

### Step 4: Collect the new shape

Ask:

> *"What's the new default policy?
> 1. **admin-only** — All admins receive view permission on every new client. (Recommended default.)
> 2. **open** — All collection members receive view on every new client. Edit and delete still require explicit grants.
> 3. **closed** — No automatic grants beyond the creator. Each new client is private until the creator explicitly shares it.
> 4. **custom** — Define a custom grants array."*

Capture as `shape`.

If `shape == custom`: walk the caller through building the grants array. For each grant:
- *"Target (member:{hash}, group:admins, or group:all-members):"* Validate format.
- *"Permissions (any combo of view, edit, delete):"*

Continue until the caller says no more.

### Step 5: Resolve the chosen shape into a grants array

- `admin-only` → `[{target: "group:admins", permissions: ["view"]}]`
- `open` → `[{target: "group:all-members", permissions: ["view"]}]`
- `closed` → `[]`
- `custom` → the array built in Step 4.

### Step 6: Show diff and confirm

Render a diff of the current grants vs. the new grants:

```
Default permission policy change:

Current:
  - target: group:admins, permissions: [view]

New:
  - target: group:all-members, permissions: [view]

Confirm yes/no?
```

If yes, proceed. If no, cancel cleanly — no writes.

### Step 7: Build the new payload

```json
{
  "version": 1,
  "last_updated": "{ISO_TIMESTAMP}",
  "preset": "{admin-only | open | closed | custom}",
  "grants": <resolved_grants_array>
}
```

Bump the policy's internal `version` field if the schema changed (V1 doesn't; this is a placeholder for future schema evolution).

### Step 8: Write (revision-aware)

`aifs_write("/shared/client-intelligence/config/default-permissions.json", <payload>, if_revision=<captured_revision>)`.

**Branch on response:**

- **Success.** Continue to Step 9.
- **`REVISION_CONFLICT`.** Another admin edited the policy between Step 2 and now. Halt with: *"The default permission policy was edited by another admin while you were editing. Re-run `@ai:edit-default-permissions` to see the current state and start over."*
- **Permission-denied.** Halt with: *"You don't have admin access on client-intelligence. Only admins (members with Drive Editor on `templates/` and `config/`) can edit the default permission policy. Ask an existing admin to grant you via `@ai:add-admin`."*
- **Any other error.** Surface verbatim and halt.

### Step 9: Confirm to caller

```
Default permission policy updated.

Preset: {new_preset}
Grants now applied to every new client:
- {grant_1}
- ...

Existing clients are unaffected — their already-applied permissions persist. The new policy takes effect for every client created from this point forward.

To view: read /shared/client-intelligence/config/default-permissions.json directly, or look at the per-instance grants the next client gets at creation.
```

---

## Directives

### Behavior

The diff in Step 6 is the contract — the caller sees exactly what's changing before any write. Don't elide it even if the change is small.

For non-technical admins, prefer presets over custom. Custom is for when an org needs a non-standard shape (e.g., grant a specific role default view-and-edit on every client, which would be expressed as a `custom` policy with a `role:` target — note: V1 doesn't ship role-based targets; that's `member:` or `group:` only).

### State Management

Not stateful from the agent's side. The single write in Step 8 is the only persistent effect.

### Constraints

- **Never call `aifs_share`, `aifs_unshare`, or `aifs_transfer_ownership`.** This task edits a content file, not Drive ACLs.
- **Never pre-check admin status via `aifs_get_permissions`.** Attempt the write; let the filesystem reject; translate.
- **Never bypass the diff confirmation in Step 6.**
- **Never modify the schema** without bumping the policy's `version` field and (in a future release) writing a migration in `/upgrade/`.

### Edge Cases

- **Caller picks `custom` with zero grants.** Equivalent to the `closed` preset; produce the same effect but record `preset: custom` so the choice is auditable. Surface: *"Your custom policy has zero grants — that's equivalent to the `closed` preset. Want to pick `closed` instead for clarity, or keep `custom`?"*
- **Caller picks a preset that matches the current policy.** Step 6's diff shows no changes; offer to cancel or proceed with a no-op write (which still bumps `last_updated`).
- **Revision conflict on retry.** Halt and recommend re-running. Do not loop the retry — concurrent admin edits should be resolved interactively.
- **Custom policy references a member who isn't in the registry.** Surface: *"`member:{hash}` doesn't resolve to a current org member. The grant will be persisted but won't apply at instance-creation time until the member exists. Proceed anyway, or cancel and fix?"*
- **Custom policy references `group:all-members` but the org's `all_members_group` is missing from `org-config.json`.** Step 4 should detect this and refuse the custom grant with: *"The org doesn't have an all-members group configured. `group:all-members` targets won't resolve. Configure the group via `@ai:edit-org` first."*
