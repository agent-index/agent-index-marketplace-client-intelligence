---
name: client-intelligence-collection-setup
type: collection-setup
version: 1.1.0
collection: client-intelligence
description: Org-admin setup for the client-intelligence collection — collects bootstrap admins, the default client-visibility prompt, and example-template choice; then bootstraps /shared/client-intelligence/ on the remote filesystem with the folder tree, initial ACLs, seeded config, and (optionally) an example template.
upgrade_compatible: true
---

## Collection Setup Overview

This setup runs once when an org admin installs the client-intelligence collection. It collects three choices from the admin, then executes the install bootstrap: creates the folder tree under `/shared/client-intelligence/`, seeds `collection-state.json` and the empty template changelog, (optionally) ships the example client template, and applies the folder grants through the core permission-changes procedure (the admin confirms the exact changes).

After this setup completes, the collection is ready for day-to-day use: admins author templates, members create clients from templates, and each client lives in one of two tiers — private (in the owner's own My Drive, shared per-person by the owner through the core permission-changes procedure) or org-public (in the commons under uniform member access). There are no per-instance ACLs in 2.x.

---

## Prerequisites

- Remote filesystem access via `aifs_*` tools (test with `aifs_auth_status()`).
- agent-index-core 3.32.0+ (the core permission-changes procedure applies every grant in step 8).
- The filesystem adapter must declare `contract_version: "2.0.0"` or higher (required by the permission-changes procedure, `aifs_get_permissions`, and revision-aware `aifs_write`). Verify before proceeding by reading the local adapter manifest at `mcp-servers/filesystem/adapter.json` and checking the `contract_version` field.
- An all-members Google Group defined at the org level. Read `org-config.json` **by id anchor** (`aifs_read("id:{org_config_id}")`, id from local `agent-index.json` → `remote_filesystem.connection`; on gdrive root-level paths don't resolve for non-Drive-members; OneDrive: read `/org-config.json` by path — a bare `id:` resolves against your own drive there, core standards § Addressing, finding 63) and confirm `remote_filesystem.connection.all_members_group` is non-empty. If absent, halt and instruct the admin to set it via `@ai:edit-org` first.
- The installing member is an org admin. In the same org-config, confirm the installing member's `member_hash` is present in the `admins[]` array. If not, halt and instruct.
- `/shared/client-intelligence/collection-state.json` does not exist (the collection is not already installed). If it exists, halt with an "already installed" message and offer the Repair Path below — unless its `version` is below `2.0.0` (a 1.x org) and the review isn't recorded yet, in which case route to `upgrade/1-to-2.md` instead (see Repair Path; the review is recorded once `upgrade_reviews_confirmed` includes `"1-to-2"`).

---

## Parameters

### `bootstrap_admins` [org-mandated]

List of members who will receive admin role on the collection at install time. Admins have Drive Editor access to `templates/` and `config/`, which lets them author templates, edit the default permission policy, and add or remove other admins.

- Default: `[installing_member]` (the member running this setup)
- Ask: "Who should be admins of the client-intelligence collection? The default is just you. You can add more org members — they will be granted admin role at install. Admins can author templates, edit the default permission policy, and add/remove other admins after install."
- Read `members-registry.json` by id anchor (`aifs_read("id:{resource_ids.members_registry}")`, id from org-config; OneDrive: `/members-registry.json` by path — core standards § Addressing, finding 63) and present available members. Let the admin select one or more.
- Store as an array of `member_hash` strings.

### `clients_default_visibility` [org-mandated] (replaces 1.x `default_permission_policy_preset`)

How create-client asks the visibility question. There is no ACL policy in 2.0 — access is structural per tier (private = owner-only until granted, in the member's own My Drive; org-public = uniform commons). Editable post-install via `@ai:edit-default-permissions`.

- Default: `private_first`
- Ask: "When members create a client, which tier should be the default answer?
  1. **private_first** (default) — private is pre-selected; org-public must be chosen explicitly. Best when members do a lot of personal prospecting.
  2. **ask** — neutral prompt, no default.
  3. **org_public_first** — org-public pre-selected. Best when the org treats client records as shared assets first (custody note: org-public records survive member departures; private ones leave with their owner)."
- Store the chosen value.

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
- `aifs_read("id:{org_config_id}")` (org-config by id anchor; OneDrive: `/org-config.json` by path — core standards § Addressing, finding 63) — capture `remote_filesystem.connection.all_members_group` and `resource_ids.members_registry`, and confirm the installing member's `member_hash` appears in the `admins[]` array. If not, halt and instruct the admin to grant org admin role first.
- `aifs_exists("/shared/client-intelligence/collection-state.json")` — if true, read it. `version` below `2.0.0`, and NOT recorded in org-config `installed_collections[client-intelligence].upgrade_reviews_confirmed` already includes `"1-to-2"` (the admin finished the review) → halt and route to `upgrade/1-to-2.md` (Repair Path, first rule). Otherwise halt with the "already installed" message and branch to the repair path on admin confirmation.

### 2. (Removed in 2.0.0)

The 1.x default-permission-policy resolution is retired — no ACL policy exists. `clients_default_visibility` is stored as-is in collection-state and setup-responses.

### 3. Seed `collection-state.json`

Write `/shared/client-intelligence/collection-state.json` via `aifs_write`:

```json
{
  "version": "{collection_version}",
  "installed": "{ISO_TIMESTAMP}",
  "installed_by": "{installing_member_hash}",
  "bootstrap_admins_requested": ["{member_hash_1}", "{member_hash_2}", ...],
  "example_template_installed": {true | false},
  "clients_default_visibility": "{private_first | ask | org_public_first}"
}
```

`{collection_version}` is the `version` from this collection's `collection.json` being installed — never a hard-coded value.

**`bootstrap_admins` is not written here.** No admin grant exists yet, so this step records only who was *requested* (`bootstrap_admins_requested`). Step 9 adds `bootstrap_admins` once step 8 returns `complete` (no access record before the permission-changes procedure returns).

This file is the canonical record that the collection is installed. Its existence is what later setup invocations check to detect "already installed."

### 4. (Removed in 2.0.0)

`config/default-permissions.json` is retired. On upgrades from 1.x, if the file exists it is overwritten with `{"retired": "2.0.0"}` (see `upgrade/1-to-2.md`). The `config/` folder itself remains (admin roster derivation + future config).

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

### 8. Apply ACLs (core permission-changes procedure)

Never call `aifs_share` directly. Per `agent-index-core/standards.md` § "Permission changes: list, confirm, apply, verify", every permission change goes through the core procedure, install time included: the installer sees the exact changes, confirms them, and each is applied under the installer's own credentials and verified.

Follow `/agent-index-core/internal/permission-changes.md` with one table:
- `calling_task: "client-intelligence/install"`, `confirmer: "admin"`
- `purpose`: "Set up client-intelligence access: bootstrap admins edit templates/ and config/; all members read the collection and write public-index/ and instances/"
- `confirmer: "admin"` is correct here: the installer is an org admin (Prerequisites).
- changes (all `share`, `is_folder: true`):

  | Resource | Recipient | Role | Reason | Source |
  |---|---|---|---|---|
  | `/shared/client-intelligence/` | `{all_members_group}` | `reader` | Members read the collection | `org_config` |
  | `/shared/client-intelligence/public-index/` | `{all_members_group}` | `writer` | Members register clients in the discovery index | `org_config` |
  | `/shared/client-intelligence/templates/` | each bootstrap admin's `sharing_identity` | `writer` | Admin role (templates) | `member_request` |
  | `/shared/client-intelligence/config/` | each bootstrap admin's `sharing_identity` | `writer` | Admin role (default policy) | `member_request` |
  | `/shared/client-intelligence/instances/` | `{all_members_group}` | `writer` | Members create org-public clients | `org_config` |

  The all-members group comes from org-config (`org_config`); the bootstrap admins are the people the installer named in the `bootstrap_admins` answer this session (`member_request`), resolved to `sharing_identity` via the members registry.

The procedure skips anything already in effect. On a clean repair run it reports "No permission changes needed".

**Act on the result:**
- **`complete`** → Step 9.
- **Anything else** (declined, cancelled, failed, not verified): surface "Install paused — {the procedure's summary lines}. Files are in place (steps 3–7 completed) but the collection is not yet usable. Re-run the install in repair mode to resume." Halt. A re-run skips whatever was already applied.

### 9. Verify

The procedure has already verified every grant against the backend. Now record the admins: read `/shared/client-intelligence/collection-state.json`, set `bootstrap_admins` to the requested admins (every admin row is `applied` or `skipped_noop`, since step 8 returned `complete`), and write it back. If the read or write fails, surface it and recommend the repair path.

### 10. Write `collection-setup-responses.md`

Write the setup choices to `/client-intelligence/setup/collection-setup-responses.md` in the canonical setup-responses format (core `org-setup` Phase 4b step 8). `apply-updates` machine-parses the `- **Value:**` lines to hand `clients_default_visibility` to members, so a frontmatter-only record reads as drift (finding 14):

```markdown
---
collection: client-intelligence
collection_version: "{collection_version}"
setup_status: complete
completed: "{YYYY-MM-DD}"
installed_by: "{installer.display_name}"
---

## Org-Mandated Parameters

### bootstrap_admins
- **Value:** {JSON array of the bootstrap admins' member hashes}

### clients_default_visibility
- **Value:** {clients_default_visibility}

### install_example_template
- **Value:** {true | false}
```

### 11. Confirm to admin

> *"Client Intelligence is set up. {N} bootstrap admin(s), default visibility prompt: {clients_default_visibility}, example template: {installed | skipped}. Clients are private-by-default in each member's own space, shareable per-person, or org-public in the commons — every client's name is org-discoverable (the visibility floor). Members can start with `@ai:create-client`; admins author templates with `@ai:create-template`."*

**Note:** `collaborative-acls.json` declares **3 of the 5** step-8 rows — the all-members-group grants on the root, `public-index/`, and `instances/` — so `install-collection` Step 5.5 can verify/re-apply them idempotently on upgrades (upgrade-collection Step 6.5 flags it; its `requires_upgrade_review: ["1-to-2"]` holds the `instances/` grant until the gated-instance review is confirmed). The two admin rows on `templates/` and `config/` are per-member, so they live only here and in `add-admin` / `remove-admin`.

---

## Repair Path

**1.x orgs take this path only after the review.** If the existing `collection-state.json` has a `version` below `2.0.0` (or none), unless org-config `installed_collections[client-intelligence].upgrade_reviews_confirmed` already includes `"1-to-2"` (the admin finished the review), halt: *"This org has client-intelligence 1.x installed. Upgrading to 2.x needs the gated-instance review first — follow `upgrade/1-to-2.md` (admin half), then run `@ai:install-collection client-intelligence`."* Do not run step 8: it would grant the all-members group Writer on `instances/` before the review, exposing 1.x instances that were gated per-instance.

If the prerequisites detected an existing 2.x `collection-state.json` and the admin chose to run repair instead of installing fresh:

- Each of steps 3–8 above is preceded by a check of the current state. Operations that already match the expected state are skipped:
  - `aifs_write` calls are preceded by `aifs_read`; if the existing content matches what would be written (modulo timestamps), the write is skipped.
  - Step 8 re-runs the permission-changes procedure, which reads current access and skips any grant already in place.
  - Step 9 sets `bootstrap_admins` only after step 8 returns `complete`; until then the file carries only `bootstrap_admins_requested`.
- The example template is re-installed only if its files are missing. The `_changelog.json` is not rewritten; if any seed entries are missing they are appended with current timestamps and a `repair_pass: true` flag in `details`.
- Verification (step 9) runs unchanged. The final confirmation message indicates "repaired" rather than "installed."

The repair path is safe to re-run multiple times.

---

## Upgrade Behavior

### Preserved Responses

On a collection upgrade within 2.x, these are preserved unchanged:

- `bootstrap_admins` — admin role is maintained by `templates/` and `config/` grants on the filesystem, not by this responses file. Changes to the admin set after install happen via `@ai:add-admin` / `@ai:remove-admin`.
- `clients_default_visibility` — preserved; editable via `@ai:edit-default-permissions`. (It replaced the 1.x `default_permission_policy_preset`, which is retired along with `config/default-permissions.json`; the 1.x → 2.0 mapping is in `upgrade/1-to-2.md`.)
- `install_example_template` — historical after install; not consulted again.

### Reset on Upgrade

None at present.

### Requires Member Attention

None at present.

### Migration Notes

- **1.1.0 (collection 2.4.0):** step 8 applies grants through the core permission-changes procedure (agent-index-core 3.32.0+; the installer confirms the exact table) instead of the retired browser review-page flow; change sources are declared per row (`org_config` for the all-members group, `member_request` for bootstrap admins). `collection-state.json` now records `bootstrap_admins_requested` at seed time and `bootstrap_admins` only after the grants are confirmed and verified; `version` fields are templated from `collection.json`. org-config and the members registry are read by id anchor. A 1.x `collection-state.json` halts the repair path and routes to `upgrade/1-to-2.md`. No action is needed for existing 2.x installs.

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
| Existing `collection-state.json` with `version` < 2.0.0 (1.x org) and `upgrade_reviews_confirmed` lacking `1-to-2` | Halt; route to `upgrade/1-to-2.md`. Never apply step 8 before the gated-instance review. Once the review is recorded, the repair path runs and rewrites `collection-state.json` `version`. |

The setup never silently leaves an inconsistent state. Every failure surfaces a specific, actionable error.
