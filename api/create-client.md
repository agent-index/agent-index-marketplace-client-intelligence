---
name: create-client
type: task
version: 1.1.0
collection: client-intelligence
description: Member-facing task to create a new client instance from a template. Interviews the member for template-defined field values, optional member-added extension fields, a client name, and optional per-creation permission grants. Writes the instance data, the public-index entry, and the initial changelog entry, then invokes permission-change-helper to apply the union of (creator default + collection-level default policy + per-creation explicit grants) as Drive ACLs on the new instance folder.
stateful: false
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills:
    - permission-change-helper
  tasks:
    - list-templates
    - view-template
external_dependencies:
  - Remote filesystem access (adapter contract 2.0.0 or higher)
  - Permission-change-helper binary (Go) or Node fallback present at mcp-servers/permission-helper-*
reads_from: "/shared/client-intelligence/"
writes_to: "/shared/client-intelligence/"
---

## About This Task

Create Client is the primary member-facing task for adding a new client record. The member picks a template, fills out the template-defined fields (mandatory and optional), optionally adds their own extension fields, names the client, and optionally pre-grants other members permission on the new instance at creation time. The task writes the instance to the remote filesystem and then invokes the permission-change-helper skill to apply the initial ACL — the union of three mechanisms specified in the parent design: creator default (creator unconditionally gets View, Edit, Delete), collection-level default policy (resolved from `config/default-permissions.json`), and per-creation explicit grants (collected interactively).

Any collection member can run this task. The visibility floor lets the new client's name be universally discoverable to all members (via `public-index/instances/{slug}.json`), while the client's data is gated by Drive ACL on the instance folder. See the parent design idea (`agency-workflow/client-intelligence-collection`) for the full rationale.

### Inputs

- **`template_slug`** (required, interactive) — slug of the template to instantiate.
- **`field_values`** (interactive, per template) — values for each mandatory field; values for any optional fields the member chooses to fill.
- **`extension_fields`** (optional, interactive) — additional member-added fields beyond the template definition. Each is a `(name, value)` pair.
- **`client_name`** (required, interactive) — universally visible name for the client. Codenames are recommended for confidential engagements.
- **`per_creation_grants`** (optional, interactive) — list of `(member, permissions)` tuples specifying additional grantees and what permissions they should receive on this new instance.

### Outputs

- `/shared/client-intelligence/instances/{slug}/instance.json` — full instance data (template fields + extension fields + metadata).
- `/shared/client-intelligence/instances/{slug}/changelog.json` — per-instance changelog with a single `created` event.
- `/shared/client-intelligence/public-index/instances/{slug}.json` — visibility-floor record (name, slug, template, status).
- ACL grants on `/shared/client-intelligence/instances/{slug}/` — applied via permission-change-helper.

Confirmation message surfaced to the member with the slug, the count of grants applied, and example invocations for `view-client` and `grant-permission`.

---

## Workflow

### Step 1: Pre-flight checks

- `aifs_auth_status`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt.
- `aifs_exists("/shared/client-intelligence/collection-state.json")`. If false, halt with the not-installed message.
- Confirm permission-helper is installed. Check for `mcp-servers/permission-helper-go/agent-index-show-plan` (preferred) or `mcp-servers/permission-helper/show-plan.sh` (fallback). If neither, halt with: *"The permission-change-helper isn't installed. Run `@ai:update` to complete the core install."*

### Step 2: Resolve template

If the member named the template (e.g., *"create a pharma client"*), parse the slug. Otherwise run `list-templates` first and ask: *"Which template? Pick by slug."*

`aifs_exists("/shared/client-intelligence/templates/{template_slug}/template.json")`. If false: *"No template `{template_slug}` exists. Run `@ai:list-templates` to see what's available."*

### Step 3: Read template and collection-level policy

- `aifs_read("/shared/client-intelligence/templates/{template_slug}/template.json")` — capture `version`, `fields`.
- `aifs_read("/shared/client-intelligence/config/default-permissions.json")` — capture `grants` array.

### Step 4: Collect field values

Walk the member through the template fields one at a time:

- For each mandatory field: ask for the value. Loop on empty/missing input until the member provides one or chooses to cancel.
- For each optional field: ask whether to fill it; if yes, collect the value.

Show a running summary of collected values so the member can verify.

### Step 5: Collect extension fields (optional)

Ask: *"Want to add any fields beyond what the template defines? You can add free-form fields specific to this client."* If yes, collect `(name, value)` pairs one at a time until the member says no more.

Validate that extension field names don't collide with template field names (case-sensitive). If a collision: ask the member to pick a different name or cancel.

### Step 6: Name the client + duplicate-name check

Ask: *"What should this client be called? Use a codename if the engagement is confidential — the name is visible to every collection member."*

Run a duplicate-name check:

1. `aifs_list("/shared/client-intelligence/public-index/instances/")`. For each `{slug}.json`, `aifs_read` and compare `name` field to the proposed name (case-insensitive) and to a normalized form (lowercase, whitespace collapsed).
2. If a match or near-match is found, surface a warning: *"A client named `{existing_name}` already exists (slug `{existing_slug}`). The new one will be a separate record. Continue with the same name, pick a different name, or cancel?"*

The warning is informational — do not block creation if the member confirms.

### Step 7: Generate the slug

Derive a kebab-case slug from the name: lowercase, replace whitespace and non-alphanumeric with hyphens, collapse repeated hyphens, trim leading/trailing hyphens. Example: `"Acme Pharma Inc."` → `acme-pharma-inc`.

Confirm slug uniqueness via `aifs_exists("/shared/client-intelligence/public-index/instances/{slug}.json")`. If a collision exists (rare given the name check, but possible from slug-collapse), append a numeric suffix: `acme-pharma-inc-2`, `acme-pharma-inc-3`, etc.

### Step 8: Collect per-creation grants (optional)

Ask: *"Anyone else you want to give access to this client right at creation? Otherwise just you (and whoever the collection's default policy includes) will have access."*

If yes, for each grant collect:
- Member identifier (email or display name; resolve to member_hash via members-registry).
- Permissions to grant: any combination of `view`, `edit`, `delete`.

### Step 9: Show summary and confirm

Show the full summary:

```
Client: {client_name} (slug: {slug})
Template: {template_slug} v{template_version}

Field values:
- {field_1}: {value_1}
- ...

Extension fields:
- {ext_1}: {value_ext_1}
- ...

Initial grants:
- Creator ({caller.display_name}): view, edit, delete
- {collection_policy_grants}
- {per_creation_grants}
```

Ask: *"Create the client?"* Proceed only on explicit confirm.

### Step 10: Structural writes

Compose the instance payload:

```json
{
  "slug": "{slug}",
  "name": "{client_name}",
  "template_slug": "{template_slug}",
  "template_version": {template_version},
  "created": "{ISO_TIMESTAMP}",
  "created_by": "{caller.member_hash}",
  "template_fields": { "{field_name}": "{value}", ... },
  "extension_fields": { "{ext_name}": "{value}", ... }
}
```

Compose the public-index payload:

```json
{
  "slug": "{slug}",
  "name": "{client_name}",
  "template_slug": "{template_slug}",
  "template_version": {template_version},
  "created": "{ISO_TIMESTAMP}",
  "created_by": "{caller.member_hash}",
  "status": "active"
}
```

Compose the initial changelog with a single `created` event:

```json
{
  "next_id": 2,
  "entries": [
    {
      "id": 1,
      "timestamp": "{ISO_TIMESTAMP}",
      "actor": {"display_name": "{caller.display_name}", "member_hash": "{caller.member_hash}"},
      "event": "created",
      "details": {
        "template_slug": "{template_slug}",
        "template_version": {template_version},
        "template_field_count": {N},
        "extension_field_count": {N}
      }
    }
  ]
}
```

Write all three files in order:

1. `aifs_write("/shared/client-intelligence/instances/{slug}/instance.json", <instance_payload>)`.
2. `aifs_write("/shared/client-intelligence/instances/{slug}/changelog.json", <changelog_payload>)`.
3. `aifs_write("/shared/client-intelligence/public-index/instances/{slug}.json", <public_payload>)`.

If any write returns permission-denied, halt and surface: *"You don't have permission to create client instances on this collection. This usually means the install is broken or your membership in the collection's all-members group has been revoked. Contact your org admin."* Do not proceed to Step 11. Partial state may exist (e.g., the instance file was written but the public-index file wasn't) — recommend re-running, or manually removing the orphan with admin assistance.

### Step 11: Capture pre-state for the helper

The instance folder now exists. Read the current permissions to populate the `before` field of the helper spec:

- `aifs_get_permissions("/shared/client-intelligence/instances/{slug}/")` — capture the current ACL state. Likely contains the inherited all-members Writer grant from the parent `instances/` folder.

### Step 12: Compute the union of initial grants

Build the union from three sources:

1. **Creator default:** the caller gets `view + edit + delete`. Always included.
2. **Collection-level default policy:** for each entry in `default-permissions.json` `grants` array, resolve the `target`:
   - `member:{hash}` → that member's email (from members-registry).
   - `group:admins` → resolve to the set of members with Drive Editor access on `templates/` via `aifs_get_permissions("/shared/client-intelligence/templates/")`.
   - `group:all-members` → resolve to the `all_members_group` address from `org-config.json`.
3. **Per-creation grants:** from Step 8.

Union: combine per (member, permission) pair. The creator's grants are unconditional. For each non-creator (member, permission), check the pre-state from Step 11 — if the member already has the requested role on the folder (via inheritance or explicit grant), filter the operation as a no-op.

### Step 13: Build the permission-change-helper spec

Compose a v1.1 spec per `agent-index-core/api/permission-change-helper.md` (v1.1 added in core 3.7.3 to support the `inherit: false` field). Every per-instance share carries `inherit: false` so the recipient sees only the explicit grant on the instance folder, not the all-members Writer they would otherwise inherit from `instances/`. This is the V1 data-visibility-floor fix activated in client-intelligence 1.1.0:

```json
{
  "version": "1.1",
  "operations": [
    {
      "op": "share",
      "resource": "/shared/client-intelligence/instances/{slug}/",
      "recipient": "{member_email_or_group_address}",
      "role": "{drive_role}",
      "inherit": false,
      "before": { "recipients": <pre_state.permissions> }
    },
    ... (one per non-no-op (member, permission) in the union)
  ],
  "context": {
    "requestor": "{caller.member_hash}",
    "calling_task": "create-client",
    "purpose": "Grant initial access on the new client `{client_name}` ({slug}). {N} share operation(s) with parent-inheritance override: creator gets full access; collection-policy grants apply; {M} per-creation grants from invocation."
  },
  "mode": "fail_soft"
}
```

Permission mapping for the `role` field: `view` → `reader`; `edit` → `writer`; `delete` → `writer` (Drive doesn't model delete separately — delete is enforced at the task layer based on whether the caller has `delete` recorded in the client-intelligence permission model. V1 stores no per-permission record; future versions may add a per-instance `permissions.json` artifact if needed for finer-grained intent.).

**Note on `inherit: false`:** the adapter (gdrive 2.3.0+) implements this via `inheritedPermissionsDisabled: true` on the folder resource, which requires `organizer` role on the Shared Drive (or `owner` on My Drive). The user who clicks Accept on the review page must have that role. If they don't, the helper returns an `AccessDeniedError` with an actionable message before any state changes — clean failure, no partial state. Org admins always have organizer rights; non-admin members who try to create-client will get the error.

**Backward compat:** specs older than client-intelligence 1.1.0 emitted v1.0 specs without `inherit`. Those still validate but produce the pre-1.1.0 additive-on-top-of-parent-inheritance behavior (the V1 limitation). Existing flows that haven't been updated to v1.1 continue to work; only the data-visibility floor is degraded for them.

If the filtered operations list is empty (everyone already has the requested permission via inheritance — possible if collection-policy resolves to the all-members group already inherited as Writer), skip Step 14 entirely.

### Step 14: Invoke the helper

Narrate to the caller before invoking:

> *"Here's a link to open a review page in your browser. It'll show {N} share operations on the new client folder. Click the link, then click Accept on the review page to apply them with your own credentials."*

Invoke `permission-change-helper` with the spec.

**Branch on outcome:**

- **`applied`** — all shares succeeded. Continue to Step 15.
- **`rejected`** — caller clicked Reject. The instance files from Step 10 are already on disk, so the client exists but with only the inherited ACL (which may or may not match the intended grants). Surface: *"Client created, but no permission changes applied. The client folder uses the default `instances/` ACL. To apply intended grants later, run `@ai:grant-permission`. To remove the client, run `@ai:delete-client {slug}`."*
- **`timed_out` / `page_closed`** — review window closed without a decision. Same instance-exists-without-grants outcome as `rejected`. Same guidance to caller.
- **`partial_failure`** — some grants applied. Surface per-failure summary and offer to retry the failed ones via `@ai:grant-permission`.
- **`apply_error` / `verification_failed`** — hard failure. Surface verbatim and the same instance-exists-without-grants guidance.
- **`binary_not_found`** — should have been caught at Step 1 pre-flight. If it surfaces here, the install state is inconsistent. Halt and recommend repair.

### Step 15: Verify post-state (on `applied` only)

`aifs_get_permissions("/shared/client-intelligence/instances/{slug}/")` — confirm the union of intended grants is now in place. The helper's apply-script verifies per-operation; this task-side verification catches the case where Drive's eventual consistency hasn't fully propagated yet (rare but possible).

### Step 16: Confirm to caller

```
Client `{client_name}` created (slug: `{slug}`).

Template: {template_slug} v{template_version}
{field_count} field(s) filled, {extension_count} extension field(s).

Permissions applied:
- {creator}: view, edit, delete
- {grantee_1}: view
- ...

What's next:
- View this client: @ai:view-client {slug}
- Edit this client: @ai:edit-client {slug}
- Grant someone else access: @ai:grant-permission {slug} {member}
```

---

## Directives

### Behavior

Conversational and adaptive to the member's expertise. A non-technical member benefits from gentle explanations of mandatory-vs-optional fields and what "extension fields" mean; an experienced member gets a terser flow.

For the duplicate-name check, do NOT block creation — the design explicitly says the warning is informational. If the member confirms despite the warning, proceed with a separate slug.

For permission grants, be careful about defaults. If the member is unsure whom to grant, default to "just me" (creator-default only) and remind them that they can grant later via `@ai:grant-permission`.

### State Management

Not stateful from the agent's side. The interview is in-memory only. After Step 10 the data is persisted; the helper invocation in Step 14 may leave the client with mismatched grants on non-`applied` outcomes (acceptable degraded state — the client exists, permissions can be fixed via `grant-permission`).

### Constraints

- **Never call `aifs_share`, `aifs_unshare`, or `aifs_transfer_ownership` directly.** All ACL changes go through `permission-change-helper`. Preflight v1.2+ will flag any direct call.
- **Never read or write to another collection's space.** The reads from `members-registry.json` and `org-config.json` are explicitly allowed at the framework level.
- **Never block creation on a duplicate-name warning.** Informational only.
- **Never override the creator-default permissions.** Creator always gets view + edit + delete on the instance, regardless of what the collection policy or per-creation grants say.
- **Never reuse an existing slug.** Step 7's suffix-disambiguation ensures uniqueness.

### Edge Cases

- **Template was edited between Step 3 and Step 10.** The instance's `template_version` is captured in Step 3; if the template's actual version on disk has moved by the time the instance is written, the instance reflects the pre-edit version. This is acceptable — instances bind to a specific version of the template at creation. If the admin chose Migrate on the edit, the instance gets the migration applied separately (per `edit-template`).
- **Collection's `default-permissions.json` is malformed or missing.** Halt at Step 3 with: *"The collection's default permission policy is unreadable. An admin can repair it via `@ai:edit-default-permissions`."*
- **Member tries to grant themselves something at Step 8.** Filter as a no-op (creator already has everything).
- **Per-creation grant target doesn't exist in members-registry.** Surface: *"I can't find a member with that email/name. Pick someone from the registry or cancel the grant."* Do not include unresolvable targets in the helper spec.
- **Helper returns `applied` but post-state verification (Step 15) shows missing grants.** Likely a Drive eventual-consistency lag. Retry the verification once after a 5-second pause; if still missing, surface a warning and proceed (the user can manually re-run `@ai:grant-permission` if anything is wrong).
- **Filesystem becomes unavailable mid-task.** Halt at the affected step with the standard not-authenticated guidance. Re-run starts the interview over; the slug-uniqueness check from Step 7 prevents duplicate clients if Step 10 partially succeeded on a prior run.

---

## V1 Limitations

Two known limitations of V1 that affect this task:

1. **Data visibility floor — PARTIALLY RESOLVED in client-intelligence 1.1.1.** The original V1 limitation was: the per-instance ACL applied by this task was additive on top of the inherited all-members Writer grant from `instances/`, meaning every collection member had Writer access on every new instance regardless of intended grants. As of agent-index-core 3.7.3 + gdrive adapter 2.3.0, per-instance share operations in this task (and `grant-permission`) are emitted with `inherit: false`, which sets `inheritedPermissionsDisabled: true` on the instance folder. This correctly blocks **write** access inheritance from the immediate parent `/shared/client-intelligence/instances/` folder — the all-members Writer grant on that parent no longer propagates to new instances.

   **However, read access via grants rooted higher in the tree** (e.g., a permissive reader grant on `/shared/client-intelligence/` itself) **still flows through**. Drive's `inheritedPermissionsDisabled` mechanism only blocks immediate-parent inheritance, not grandparent. So if your org has any ancestor grant of the form `all-members reader on /shared/client-intelligence/`, that grant continues to reach the instance contents even with `inherit: false` applied at the instance level.

   The full fix is tracked as core-improvements idea `data-visibility-floor-ancestor-leak`. Two real design options exist (apply the override higher in the tree, or restructure the all-members grant location); both require broader access-control-project decisions and ship in 3.8.0 or later.

   **Until the full fix ships, two operational patterns are recommended for confidential client engagements:**

   1. **Codename pattern (moderate confidentiality, V1 default).** Don't put the real client name in the instance folder slug, public-index entry, or other discoverable fields. Use a non-identifying codename for the instance and maintain the codename → real-identity mapping outside agent-index (e.g., in a separately-permissioned password manager, encrypted note, or admin-only doc). Low friction; the workflow inside client-intelligence works normally; only the discoverability surface is opaque to non-grantees.

   2. **Empty-shell + off-platform pattern (highest sensitivity).** Create the instance with only the public-index entry (codename + minimal metadata); keep all sensitive data outside agent-index entirely — for example, in a separately-permissioned Drive folder, an encrypted vault, or a client-managed shared workspace. The instance folder becomes a metadata-only stub that admins use for status tracking and workflow, while the actual sensitive contents never enter the leak surface. Higher friction (workflow benefits inside the instance are reduced) but provides absolute confidentiality regardless of any ancestor-grant leak.

   Choose based on the engagement: codename for the common case (moderately sensitive client work where exposure of contents to other org members is undesirable but not catastrophic); empty-shell for the rare case where any cross-org visibility is a deal-breaker (M&A targets, active security incidents, regulated-industry engagements with strict data-residency requirements, etc.).
2. **Permission audit history requires `aifs_get_audit`.** Drive Activity records the share events from the helper invocation, but no V1 task surfaces that history. `view-client-audit` is deferred to post-V1 pending access-control's `aifs_get_audit` adapter operation.
