---
name: create-client
type: task
version: 2.1.0
collection: client-intelligence
description: Member-facing task to create a new client instance from a template. Interviews the member for template-defined field values, optional extension fields, and a client name, then asks the visibility question — private (default; stored in the member's own My Drive, shareable per-person later) or org-public (stored in the org commons under /shared, uniformly accessible to all members). Writes the instance, the universal-floor pointer, and the initial changelog. Private-tier per-creation grants go through permission-change-helper with the owner's Accept.
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
  - Remote filesystem access (adapter 2.5.0+ — ID-anchor addressing)
  - permission-change-helper binary 0.4.1+ (private-tier grants only)
reads_from: "/shared/client-intelligence/, id:{member_folder_id}/clients/"
writes_to: "id:{member_folder_id}/clients/, /shared/client-intelligence/"
---

## About This Task

Create Client is the primary member-facing task for adding a new client record. The member picks a template, fills the template-defined fields (mandatory and optional), optionally adds extension fields, names the client, and chooses its **visibility tier**:

- **Private (the default).** The instance is written to the member's own My Drive (`id:{member_folder_id}/clients/{slug}/`) — owned by the member, invisible to everyone else until the owner grants access (at creation or later via `@ai:grant-permission`: share = reader, collaborator = writer). This is the owned-content model (standards.md § "Addressing").
- **Org-public.** The instance is written to the org commons (`/shared/client-intelligence/instances/{slug}/`), where every member has uniform writer access (install-time `collaborative-acls.json` grant). No per-instance permissions exist in this tier — uniform access is the deal; if a client needs gating, it belongs in the private tier. Edits are attributed via mandatory changelog events (task-level guardrail).

Either way, the **universal visibility floor** applies: a pointer record (name, owner, template, status, tier) is written to `/shared/client-intelligence/public-index/instances/{slug}.json` at creation — every member can discover that the client relationship exists and who owns it; the data itself is gated by tier. Use a codename if even the name is sensitive.

There is NO creator-applied ACL step in this task (removed in 2.0.0): the 1.x design had members applying per-instance Drive ACLs under `/shared`, which Drive does not permit for non-Managers (finding F12) — every non-admin create failed. The 1.x "ancestor leak" limitation is structurally dissolved by the two-tier model.

### Inputs

- **`template_slug`** (required, interactive) — template to instantiate.
- **`field_values`** (interactive, per template) — mandatory + chosen optional fields.
- **`extension_fields`** (optional, interactive) — member-added `(name, value)` pairs.
- **`client_name`** (required, interactive) — universally visible name (floor). Codenames recommended for confidential engagements.
- **`visibility`** (interactive) — `private` (default per org setup `clients_default_visibility`) or `org_public`.
- **`per_creation_grants`** (optional, interactive, private tier only) — `(member, level)` tuples; level ∈ `read` (reader) | `collaborate` (writer).

### Outputs

Private tier:
- `id:{member_folder_id}/clients/{slug}/instance.json`, `changelog.json`
- Optional grants on `id:{client_folder_id}` via permission-change-helper (owner Accepts)

Org-public tier:
- `/shared/client-intelligence/instances/{slug}/instance.json`, `changelog.json`

Both:
- `/shared/client-intelligence/public-index/instances/{slug}.json` — universal-floor pointer

## Workflow

### Step 1: Pre-flight

- `aifs_auth_status`; re-auth on failure; halt if unrecoverable.
- `aifs_exists("/shared/client-intelligence/collection-state.json")` — halt with not-installed message if false.
- Read `member-index.json` (local) for `member_hash`, `display_name`, and **`member_folder_id`**. If `member_folder_id` is missing: "Your private member space isn't set up yet — run `@ai:update` (it self-provisions) and retry." Halt. (No admin backfill exists; self-provisioning per core 3.9.0+.)
- Read the org's `clients_default_visibility` from the collection's `setup/collection-setup-responses.md` (org-mandated; default `private_first`).

### Step 2: Resolve template

As 1.x: parse or ask; `aifs_exists("/shared/client-intelligence/templates/{template_slug}/template.json")`; halt with list-templates guidance if missing.

### Step 3: Read the template

`aifs_read(".../templates/{template_slug}/template.json")` — capture `version`, `fields`. (The 1.x read of `config/default-permissions.json` is REMOVED — no ACL policy applies at creation in 2.0.)

### Step 4: Collect field values

Unchanged from 1.x: mandatory fields loop-until-provided; optional fields offered; running summary shown.

### Step 5: Collect extension fields (optional)

Unchanged from 1.x, including the name-collision check against template fields.

### Step 6: Name the client + duplicate check (the floor at work)

Ask: *"What should this client be called? The name is visible to every member regardless of tier — use a codename if the engagement is confidential."*

Duplicate check: `aifs_list("/shared/client-intelligence/public-index/instances/")`; read each pointer; compare `name` (case-insensitive, whitespace-normalized). On match/near-match: *"A client named `{existing_name}` already exists, owned by {owner} ({tier}). View it with `@ai:view-client {existing_slug}`, continue creating a separate record, pick a different name, or cancel?"* Informational — never block on confirm.

### Step 7: Generate the slug

As 1.x (kebab-case; uniqueness against the pointer index; numeric suffix on collision).

### Step 8: The visibility question

Per the org default (`private_first` unless configured otherwise), ask:

> *"Should this client be **private** (default — stored in your own space; only you can see it until you share it with specific people) or **org-public** (stored in the org commons; every member can view and edit it)?"*

Include the custody guidance once per session: *"Rule of thumb: revenue-bearing org relationships belong org-public (the org keeps them if you leave); prospecting and sensitive engagements start private."*

If `private`: optionally collect `per_creation_grants` — *"Anyone you want to give access right away? (read, or collaborate = read+write). Otherwise it's just you, and you can grant later with `@ai:grant-permission`."* Resolve each target against `members-registry.json`; drop unresolvable targets with a notice.

If `org_public`: no grants question (uniform access).

### Step 9: Summary and confirm

Show name/slug/template/fields/extension fields/tier (+ planned grants if any). Proceed only on explicit confirm. **No filesystem writes before this point.**

### Step 10: Structural writes

Compose `instance.json` (as 1.x, plus `"visibility": "private"|"org_public"`) and `changelog.json` (single `created` event, actor = caller).

**Private tier:**
1. `aifs_write("id:{member_folder_id}/clients/{slug}/instance.json", ...)`
2. `aifs_write("id:{member_folder_id}/clients/{slug}/changelog.json", ...)`
3. `aifs_stat("id:{member_folder_id}/clients/{slug}")` → record `client_folder_id`.

**Org-public tier:**
1. `aifs_write("/shared/client-intelligence/instances/{slug}/instance.json", ...)`
2. `aifs_write("/shared/client-intelligence/instances/{slug}/changelog.json", ...)`

On permission-denied in the org tier: the install's provisioning is incomplete — *"The org commons isn't writable; your admin needs to run `@ai:install-collection client-intelligence` (Step 5.5 provisioning)."* Halt.

### Step 11: Write the universal-floor pointer

`aifs_write("/shared/client-intelligence/public-index/instances/{slug}.json", ...)`:

```json
{
  "type": "client", "name": "{client_name}", "slug": "{slug}",
  "template_slug": "{template_slug}", "template_version": {template_version},
  "owner": "{display_name}", "owner_hash": "{member_hash}",
  "status": "active",
  "scope": "private" | "org_public",
  "location": {"folder_id": "{client_folder_id}"} | {"path": "/shared/client-intelligence/instances/{slug}/"},
  "created": "{ISO}", "last_updated": "{ISO}", "owner_departed": false
}
```

For a private client with per-creation grants, `scope` becomes `{"readers": [...], "collaborators": [...]}` — but ONLY after Step 12's gate; write it as `"private"` now.

### Step 12: Per-creation grants (private tier with grants only)

Compose ONE permission-change-helper spec: an `op: "share"` per grant on resource **`id:{client_folder_id}`** (bare folder ID — helper 0.4.1+; the exact folder, least privilege): read → `reader`, collaborate → `writer`. The member (owner) reviews and **Accepts** — the folder is theirs, so the grants apply under their own credentials. Never call `aifs_share` directly.

**HARD GATE (strategy 1.1.3 precedent):** proceed only when the outcome file reports `"outcome": "applied"`, OR — if the file is missing/page-lifecycle-valued despite a confirmed Accept — an independent `aifs_get_permissions("id:{client_folder_id}")` confirms every requested grant. On `applied`: overwrite the pointer with the granted `scope`. On `rejected`/failure: the client stays private with no grants — *"Client created (private, just you). Grants weren't applied; add them anytime with `@ai:grant-permission`."* Never write a scope the Drive state doesn't back.

### Step 13: Confirm to caller

Private: *"Client `{name}` created — private, in your own space{, shared with N people}. Next: `@ai:view-client {slug}`, `@ai:grant-permission {slug}`, or `@ai:transition-client {slug}` to make it org-public."*
Org-public: *"Client `{name}` created in the org commons — every member can view and edit it (edits are attributed in the changelog). Next: `@ai:view-client {slug}`."*

### Optional instance subfolder: branding/ (added in 2.1.0)

Instances MAY carry a `branding/` subfolder (client_display_name + client-logo) supplying slot values for brand-aware documents like `client-brief`. Not created at instance creation — `edit-client` creates it on first use. It inherits the instance's tier structurally.

## Directives

### Behavior

Conversational; adapt depth to the member. The visibility question is the one decision that matters — make the custody trade-off legible without lecturing. Duplicate warnings inform, never block. If unsure about grants, default to just-the-owner.

### Constraints

- **Never call `aifs_share`/`aifs_unshare`/`aifs_transfer_ownership` directly.** Private-tier grants only via permission-change-helper with the owner's Accept.
- **Never apply or attempt per-instance ACLs in the org tier** — uniform access by design; Drive forbids member-applied /shared folder grants anyway (F12).
- **Never write the pointer's granted scope before the Step 12 hard gate passes.**
- **Never block on duplicate names**; never reuse a slug; never write before Step 9's confirm.
- Org-tier writes that fail permission checks indicate broken provisioning — halt and name the admin fix; do not retry as a different identity.

### Edge Cases

- Template edited mid-interview: instance binds to the version read at Step 3 (as 1.x).
- `member_folder_id` missing → Step 1 halt with self-provisioning guidance.
- Unresolvable grant target → dropped with notice, never included in the spec.
- Helper `partial_failure` → pointer gets ONLY the verified grants; offer retry via `@ai:grant-permission`.
- Org-public chosen but org default is `private_first` — fine; the default shapes the prompt, not the choice.
- Floor pointer write fails after instance writes: surface — the client exists but is undiscoverable; re-run guidance (slug check makes re-runs safe).
