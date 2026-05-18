---
name: client-intelligence-collection-setup
type: collection-setup
version: 1.0.0
collection: client-intelligence
description: Org-admin setup for the client-intelligence collection — collects bootstrap admins, default permission policy preset, and example-template choice; then bootstraps /shared/client-intelligence/ on the remote filesystem with the folder tree, initial ACLs, seeded config, and (optionally) an example template.
upgrade_compatible: true
---

## Collection Setup Overview

This setup runs once when an org admin installs the client-intelligence collection. It collects three choices from the admin, then executes the install bootstrap: creates the folder tree under `/shared/client-intelligence/`, applies initial ACLs, seeds the default permission policy and the empty template changelog, and (optionally) ships the example client template.

After this setup completes, the collection is ready for day-to-day use: admins author templates, members create clients from templates, and per-instance permissions are managed via Drive ACLs through the access-control adapter contract.

---

## Prerequisites

- Remote filesystem access via `aifs_*` tools (test with `aifs_auth_status()`).
- The filesystem adapter must declare `contract_version: "2.0.0"` or higher (required for `aifs_share`, `aifs_unshare`, `aifs_get_permissions`, and revision-aware `aifs_write`). Verify before proceeding by reading the local adapter manifest at `mcp-servers/filesystem/adapter.json` and checking the `contract_version` field.
- An all-members Google Group defined at the org level. Read `org-config.json` and confirm `remote_filesystem.connection.all_members_group` is non-empty. If absent, halt and instruct the admin to set it via `@ai:edit-org` first.
- The installing member is an org admin. Read `org-config.json` and confirm the installing member's `member_hash` is present in the `admins[]` array. If not, halt and instruct.
- `/shared/client-intelligence/collection-state.json` does not exist (the collection is not already installed). If it exists, halt with an "already installed" message and offer the repair path described in Upgrade Behavior below.

---

## Parameters

### `bootstrap_admins` [org-mandated]

List of members who will receive admin role on the collection at install time. Admins have Drive Editor access to `templates/` and `config/`, which lets them author templates, edit the default permission policy, and add or remove other admins.

- Default: `[installing_member]` (the member running this setup)
- Ask: "Who should be admins of the client-intelligence collection? The default is just you. You can add more org members — they will be granted admin role at install. Admins can author templates, edit the default permission policy, and add/remove other admins after install."
- Read `members-registry.json` via `aifs_read("/members-registry.json")` and present available members. Let the admin select one or more.
- Store as an array of `member_hash` strings.

### `default_permission_policy_preset` [org-mandated]

The collection-level default permission policy applied to every newly created client instance. This is one of three mechanisms that compose at instance creation (creator default unconditionally grants all three permissions to the creator + this collection policy + per-creation explicit grants from the create-client invocation). Editable post-install via `@ai:edit-default-permissions`.

- Default: `admin-only`
- Ask: "What default access should new clients have? Three presets are available:
  1. **admin-only** (default) — All admins receive view permission on every new client. Creators still control who else has access.
  2. **open** — All collection members receive view permission on every new client. Edit and delete still require explicit grants.
  3. **closed** — No automatic grants beyond the creator. Each new client is private until the creator explicitly shares it.
  You can also customize the policy after install via `@ai:edit-default-permissions`."
- Store the chosen preset name (`"admin-only"`, `"open"`, or `"closed"`). The Setup Completion step resolves it into the corresponding `grants` array.

### `install_example_template` [org-mandated]

Whether to install the example client template at setup time. The example template defines a minimal client shape (`company_name` mandatory; `primary_contact_name`, `primary_contact_email`, `notes` optional) and gives new orgs a working template to exercise the collection with immediately.

- Default: `true`
- Ask: "Install the example client template? It gives you a basic template to try the collection with right away. You can delete it later if you author your own templates."
- Store as a boolean.

---

## Setup Completion

After the admin answers all three parameters, present a confirmation summary and ask the admin to confirm before any write happens. Then execute the bootstrap in order:

### 1. Validate prerequisites

- `aifs_auth_status()` — confirm authenticated. If not, halt.
- Read `mcp-servers/filesystem/adapter.json` locally and confirm the `contract_version` field is `"2.0.0"` or higher. If lower, halt and instruct the admin to update the filesystem adapter via `@ai:edit-org`.
- `aifs_read("/org-config.json")` — capture `remote_filesystem.connection.all_members_group`.
- `aifs_read("/org-config.json")` and confirm the installing member's `member_hash` appears in the `admins[]` array. If not, halt and instruct the admin to grant org admin role first.
- `aifs_exists("/shared/client-intelligence/collection-state.json")` — if true, halt with the "already installed" message and branch to the repair path on admin confirmation.

### 2. Resolve the default permission policy

Resolve the `default_permission_policy_preset` choice into a concrete `grants` array:

- `admin-only` → `[{"target": "group:admins", "permissions": ["view"]}]`
- `open` → `[{"target": "group:all-members", "permissions": ["view"]}]`
- `closed` → `[]`

### 3. Seed `collection-state.json`

Write `/shared/client-intelligence/collection-state.json` via `aifs_write`:

```json
{
  "version": "1.0.0",
  "installed": "{ISO_TIMESTAMP}",
  "installed_by": "{installing_member_hash}",
  "bootstrap_admins": ["{member_hash_1}", "{member_hash_2}", ...],
  "example_template_installed": {true | false},
  "default_permission_policy_preset": "{admin-only | open | closed}"
}
```

This file is the canonical record that the collection is installed. Its existence is what later setup invocations check to detect "already installed."

### 4. Seed `config/default-permissions.json`

Write `/shared/client-intelligence/config/default-permissions.json` via `aifs_write`:

```json
{
  "version": 1,
  "last_updated": "{ISO_TIMESTAMP}",
  "grants": {resolved_grants_from_step_2}
}
```

### 5. Seed `templates/_changelog.json`

Write `/shared/client-intelligence/templates/_changelog.json` via `aifs_write`:

```json
{
  "next_id": 1,
  "entries": []
}
```

If `install_example_template` is false, skip to step 7. Otherwise continue with step 6.

### 6. Seed the example template (if enabled)

Read the example template payload from `setup/example-template.json` in the collection source. Write it to the remote filesystem as the canonical example:

- `aifs_write("/shared/client-intelligence/templates/example-client/template.json", <payload>)`
- `aifs_write("/shared/client-intelligence/templates/example-client/versions/v1.json", <payload>)` (immutable snapshot of v1)

Then read `_changelog.json` and append a `template_created` event:

```json
{
  "id": 1,
  "timestamp": "{ISO_TIMESTAMP}",
  "actor": {"display_name": "{installer.display_name}", "member_hash": "{installer.member_hash}"},
  "template_slug": "example-client",
  "event": "template_created",
  "details": {
    "source": "install-time-setup",
    "fields": [
      {"name": "company_name", "property": "mandatory"},
      {"name": "primary_contact_name", "property": "optional"},
      {"name": "primary_contact_email", "property": "optional"},
      {"name": "notes", "property": "optional"}
    ]
  },
  "migration": null
}
```

Increment `next_id` to 2. Write `_changelog.json` back.

### 7. Seed folder marker files

Drive folders without files are ephemeral. To keep `public-index/instances/` and `instances/` enumerable via `aifs_list` even when no clients exist yet, write `.gitkeep` markers:

- `aifs_write("/shared/client-intelligence/public-index/instances/.gitkeep", "")`
- `aifs_write("/shared/client-intelligence/instances/.gitkeep", "")`

### 8. Apply ACLs via permission-change-helper

Do **not** call `aifs_share` directly from this step. Per `agent-index-core/standards.md` ("Permission-Modifying Operations"), agents are categorically prohibited from making security-changing calls on the user's behalf, even at install time. Preflight v1.2+ flags any direct call to `aifs_share` / `aifs_unshare` / `aifs_transfer_ownership` in a task workflow as an authoring error.

Route the ACL grants through the `permission-change-helper` skill, which surfaces a review page in the installer's browser and applies the changes with the installer's own OAuth token after explicit Accept. The canonical exemplar for this pattern is `agent-index-core/api/invite-member.md` Step 6.

**Build the spec:**

1. **Capture pre-state** for each of the five resources whose ACLs will change. `aifs_get_permissions` is agent-callable directly:
   - `aifs_get_permissions("/shared/client-intelligence/")`
   - `aifs_get_permissions("/shared/client-intelligence/public-index/")`
   - `aifs_get_permissions("/shared/client-intelligence/templates/")`
   - `aifs_get_permissions("/shared/client-intelligence/config/")`
   - `aifs_get_permissions("/shared/client-intelligence/instances/")`

2. **Build the operations list.** The full set of intended grants is:

   | Resource | Recipient | Role |
   |---|---|---|
   | `/shared/client-intelligence/` | `all_members_group` | `reader` |
   | `/shared/client-intelligence/public-index/` | `all_members_group` | `writer` |
   | `/shared/client-intelligence/templates/` | each `bootstrap_admin` | `writer` |
   | `/shared/client-intelligence/config/` | each `bootstrap_admin` | `writer` |
   | `/shared/client-intelligence/instances/` | `all_members_group` | `writer` |

   For each (resource, recipient, role) tuple, look up the recipient in the corresponding pre-state. **Omit** operations where the recipient already has the requested role on the path (no-op filter). For each non-no-op tuple, build:

   ```json
   {
     "op": "share",
     "resource": "<path>",
     "recipient": "<email_or_group>",
     "role": "<reader|writer>",
     "before": {"recipients": <pre-state.permissions>}
   }
   ```

3. **Compose the full spec:**

   ```json
   {
     "version": "1.0",
     "operations": [...],
     "context": {
       "requestor": "{installer_member_hash}",
       "calling_task": "client-intelligence/install",
       "purpose": "Bootstrap ACLs for the client-intelligence collection: {N_admins} bootstrap admin(s) get Editor on templates/ and config/; the all-members group gets Reader on the root and Writer on public-index/ and instances/ so members can register and create client instances."
     }
   }
   ```

4. **If the filtered operations list is empty** (entire ACL state already in place — happens on a clean repair run): skip Step 8 entirely. Surface "All required ACLs are already in place; no permission changes needed."

**Invoke the helper:**

Narrate to the installer before invoking:

> *"Here's a link to open a review page in your browser. It'll show {N} share operations covering the collection root, public-index, templates, config, and instances folders. Click the link, then click Accept on the review page to apply them with your own credentials."*

Call the `permission-change-helper` skill with the spec.

**Branch on outcome:**

- **`applied`** — All requested shares succeeded. Continue to Step 9 (Verify).
- **`rejected`** — Installer declined. Surface: "Install paused — no permissions modified. Files are in place (steps 3–7 completed) but the collection is not yet usable. Re-run the install in repair mode to resume." Halt.
- **`timed_out`** / **`page_closed`** — Surface: "Review window closed without a decision. Re-run the install in repair mode to resume." Halt.
- **`partial_failure`** — Surface a per-failure summary with the helper's `error_detail`. Offer to retry the failed ops only or to halt. Default: halt and recommend repair.
- **`apply_error`** / **`verification_failed`** — Hard failure. Surface the error verbatim. Halt. Recommend repair after the underlying issue is resolved.
- **`binary_not_found`** — Surface: "The permission helper isn't installed. Run `@ai:update` or `@ai:member-bootstrap` to complete the core install, then re-run client-intelligence install in repair mode."

The ordering of grants (parent before children, tighter child overriding broader parent) is handled inside the helper's apply-script per Drive's ACL semantics; the spec declares the desired end-state only.

### 9. Verify

The helper's apply-script already verifies post-share state per operation, but for an install we also do task-side post-verification:

- `aifs_read("/shared/client-intelligence/collection-state.json")` — confirm the write succeeded.
- `aifs_read("/shared/client-intelligence/config/default-permissions.json")` — confirm.
- `aifs_get_permissions("/shared/client-intelligence/")` — confirm the all-members Reader grant is present.
- `aifs_get_permissions("/shared/client-intelligence/public-index/")` — confirm the all-members Writer grant.
- `aifs_get_permissions("/shared/client-intelligence/templates/")` — confirm each bootstrap admin is a Writer.
- `aifs_get_permissions("/shared/client-intelligence/config/")` — confirm each bootstrap admin is a Writer.
- `aifs_get_permissions("/shared/client-intelligence/instances/")` — confirm the all-members Writer grant.

If any verification fails, surface which step failed and recommend running the repair path.

### 10. Write `collection-setup-responses.md`

Write a YAML record of the setup choices to `/setup/collection-setup-responses.md` (local to the install) for future reference:

```yaml
---
collection: client-intelligence
version: 1.0.0
installed: "{ISO_TIMESTAMP}"
installed_by: "{installer.display_name}"
bootstrap_admins: [...]
default_permission_policy_preset: "..."
install_example_template: true
---
```

### 11. Confirm to admin

> *"Client Intelligence is set up. {N} bootstrap admin(s), default permission policy: {preset}, example template: {installed | skipped}. Members can list available templates with `@ai:list-templates` or jump straight to creating a client with `@ai:create-client`. Admins can author new templates with `@ai:create-template` or edit the default permission policy with `@ai:edit-default-permissions`."*

---

## Repair Path

If the prerequisites detected an existing `collection-state.json` and the admin chose to run repair instead of installing fresh:

- Each of steps 3–8 above is preceded by a check of the current state. Operations that already match the expected state are skipped:
  - `aifs_write` calls are preceded by `aifs_read`; if the existing content matches what would be written (modulo timestamps), the write is skipped.
  - `aifs_share` calls are preceded by `aifs_get_permissions`; if the expected grant is already in place, the share call is skipped.
- The example template is re-installed only if its files are missing. The `_changelog.json` is not rewritten; if any seed entries are missing they are appended with current timestamps and a `repair_pass: true` flag in `details`.
- Verification (step 9) runs unchanged. The final confirmation message indicates "repaired" rather than "installed."

The repair path is safe to re-run multiple times.

---

## Upgrade Behavior

### Preserved Responses

On collection upgrade (v1.x → v1.y), these are preserved unchanged:

- `bootstrap_admins` — admin role is maintained by `templates/` and `config/` ACLs on the filesystem, not by this responses file. Changes to the admin set after install happen via `@ai:add-admin` / `@ai:remove-admin`.
- `default_permission_policy_preset` — the resolved `default-permissions.json` is preserved; the preset name on file is historical metadata.
- `install_example_template` — historical after install; not consulted again.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- v1.0.0 → future versions: migration notes will be added here as new versions are published. Any version that changes the default permission policy schema will require an admin to confirm the new policy shape before the upgrade applies; the previous policy value is preserved for reference in `collection-setup-responses.md`.

---

## Error Handling

| Failure | Response |
|---|---|
| `aifs_auth_status` reports not authenticated | Halt; suggest `@ai:member-bootstrap`. |
| Filesystem adapter contract < 2.0.0 | Halt; suggest updating the adapter via `@ai:edit-org`. |
| `all_members_group` missing from org-config | Halt; suggest setting it via `@ai:edit-org`. |
| Installer not org admin | Halt with clear message; do not proceed. |
| `collection-state.json` already exists, no repair confirmation | Halt with "already installed" message; offer the repair path. |
| Any seed write fails | Stop, report which step failed, leave partial state in place. Caller can re-run in repair mode to complete. |
| Any ACL apply fails | Stop, report which folder and which grant failed. Note that the collection is in a partially-installed state and members may not be able to use it until the ACL is applied. Recommend re-run in repair mode. |
| Bootstrap admin not findable in members-registry | Halt with the specific member identifier and ask the admin to verify it. |

The setup never silently leaves an inconsistent state. Every failure surfaces a specific, actionable error.
