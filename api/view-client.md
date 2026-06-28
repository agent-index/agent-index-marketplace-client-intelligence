---
name: view-client
type: task
version: 2.3.0
collection: client-intelligence
description: Member-facing read task that displays a client's full data and recent changelog. Tier-resolved via the universal-floor pointer — org-public clients read from the commons; private clients read from the owner's space by folder ID (succeeds only for the owner and grantees). Callers without access get the floor view (name, owner, tier) plus how to ask the owner. When a brand-book provider supplies the `client-profile` template, renders a branded, interactive Cowork profile (with Edit / Manage access / Generate brief / Change visibility / Archive) by default; otherwise renders the plain record. Can export the profile to a standalone HTML / PDF / Word artifact on request.
stateful: false
produces_artifacts: true
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
optional_capabilities:
  - "brand-book (get-template/get-element/get-brand-guidelines) — enables the branded interactive profile and standalone exports; degrades to the plain record when absent. Resolution is inlined in Step 4 (members can't read /internal/ at runtime — int4)."
reads_from: "/shared/client-intelligence/public-index/, /shared/client-intelligence/instances/, id:{folder_id}/"
writes_to: null
---

## About This Task

View Client renders a client's full record: template fields, extension fields, metadata, and the most recent changelog entries. Access is tier-mechanical, not probed:

1. Resolve the client via its pointer at `/shared/client-intelligence/public-index/instances/{slug}.json`.
2. `org_public` → read `instance.json` + `changelog.json` from `location.path` (every member can).
3. Private → read via the cross-drive anchor `id:{location.item_drive_id}:{location.folder_id}/...` when the pointer carries `item_drive_id` (a private client lives on the owner's drive — C.1.3 `crossdriveread`), falling back to the bare `id:{location.folder_id}/...` only for older pointers that predate it and for the caller's OWN clients on their own drive. The qualified form is what lets a grantee on OneDrive open a client the owner shared to them — a bare anchor would resolve against the *caller's* drive and 404 even with the grant present; on gdrive the bare anchor already reaches shared items, so the qualified form is OneDrive parity and harmless there. This succeeds for the owner and anyone the owner has granted; for everyone else Drive returns ACCESS_DENIED — which IS the floor working: render the pointer's name/owner/template/status and say *"`{name}` is private to {owner}. Ask them to share it (granting is owner-run: `@ai:grant-permission`)."* Never fall back to an external connector on a 404 — repair `item_drive_id` instead (standards § "Reads go through aifs only").
4. `archived` / `archived-moved-private` statuses → say so; for moved stubs, follow the pointer to the current location instead of rendering the stub.

## Workflow

### Step 1: Pre-flight

`aifs_auth_status` (re-auth/halt); collection installed check.

### Step 2: Resolve

Accept slug or name (name → search the pointer index; multiple matches → disambiguate by slug). Pointer missing → *"No client `{input}`. See the directory: `@ai:list-clients`."*

### Step 3: Tier-resolved read

Per the rules above. Read `instance.json` fully; read `changelog.json` and take the most recent 10 entries (offer the rest on request). On private-tier ACCESS_DENIED → floor view + ask-the-owner guidance (never retry via other paths). On org-tier permission-denied → provisioning broken; name the admin fix (`@ai:install-collection client-intelligence`).

### Step 4: Resolve brand (inline — int4)

Inlined (members can't read `/internal/` at runtime — bug 20260608 int4; `/internal/resolve-brand.md` is authoring docs only):

1. **Find the provider.** `aifs_read("/org-config.json")` → `capability_providers["brand-book"].providers`. Empty/absent/multiple → **no brand**: go to Step 5b (plain record) with a one-line notice.
2. **Provider read base (id-anchored).** Provider collection in `installed_collections[]`; `base = "id:{folder_id}"` if present else `/{provider_collection}`. `brand_book_version` = the provider's registry `capability_version`.
3. **Fetch once (read-only ops from `{base}/api/...`):** `get-template("client-profile", "html")` — `client-profile` is THIS collection's record artifact type; whatever brand template the org published under that name is used (`found:false`/failure → Step 5b). For **each element named in the returned template's `sections`** (no hardcoded element names): `get-element(name, "html")` (tokens resolve against `/shared/brand-book/tokens/*`); per-element `found:false`/`rendering_missing` → native default for that element.
4. **Personal-element precedence** as elsewhere (optional → personal wins; required → org wins except where org-undefined).
5. **Slot value.** `client-logo` for the header badge comes from the instance's `branding/` read at the client's tier; absent → initials badge.

**Failure rule:** any failure degrades to Step 5b — brand resolution never blocks the view.

### Step 5a: Render — branded interactive profile (default in Cowork)

When Step 4 produced a template AND the surface is Cowork: compose the profile from the template's `sections` — its header element (badge/logo, name, kind · tier + status) over its detail sections for the template fields, the extension fields, and recent activity (date → event from the changelog). Fill the header's `marker_fills` actions, mapping each `action_contract` handle to this collection's flow:

| Template handle | Scope | This collection runs |
|---|---|---|
| `edit` | record | **edit-client** for this slug |
| `manage-access` | record | **view-permissions** (who has access) → **grant-permission** / **revoke-permission** to change (owner-run, private tier) |
| `brief` | record | **client-brief** for this slug |
| `transition` | record | **transition-client** (private ⇄ org-public) |
| `archive` | record, `confirm` | **delete-client** `mode: archive` (soft; floor record survives) |

The button emits its handle (+ slug) back into the session; on click, run the mapped task per its own definition and confirmation gate, then re-render. **Gate by authority and tier:** show only actions the caller can perform — e.g. `edit` only when the caller has write access; `manage-access` / `transition` / `archive` for org-public (any member) or, in the private tier, the owner only. On a name-only floor view (private, no access) render the identity block plus the "ask {owner} to share it" guidance and **no record actions**.

### Step 5b: Render — plain record (fallback / on request)

When there is no brand template, the surface isn't Cowork, or the member asks for "plain": Name, slug, tier (+ caller's access: yours / shared with you (read|write) / org-public), owner (+ departed annotation if `owner_departed`), template + version, status, template fields, extension fields, created/last-updated, recent changelog (actor + event + timestamp). End with relevant verbs: edit (if writable), grant/revoke (owner, private tier), transition (owner).

### Step 6: Export on request (standalone artifact)

If the member asks to export/save the profile: render the same composition to the requested format from the template's `surfaces.export_formats` — `html`, `pdf`, `docx`, or `markdown` (the docx/pdf double as a printable client one-pager). Action buttons are omitted in static formats. Save the file as an artifact and link it; never write the commons or change client data. (For a richer narrative brief, `client-brief` remains the dedicated document task.)

## Directives

### Constraints

- **Read-only. Never write; never call permission ops.**
- **Never bypass the floor** — an ACCESS_DENIED private read ends in the floor view, full stop; no side channels.
- One instance at a time; never expose other instances' data.
- Moved stubs (`archived-moved-private`) are never rendered as data — follow the pointer to the live copy.

### Edge Cases

- Pointer exists, org-public `instance.json` missing → data-integrity warning (matches list-clients' integrity pass); suggest admin review.
- Pointer exists, private `id:{folder_id}` read fails FOR THE OWNER → folder moved/deleted outside agent-index; surface plainly and suggest the integrity pass.
- Changelog unparseable → render instance data + a changelog warning; never block the view.
- Multiple name matches → ask for the slug (names aren't unique; creation's duplicate check is informational).
- `archived` status → render with the marker: *"Archived. To unarchive: `@ai:edit-client {slug}`, reset status to active."*
- Caller asks for one field → answer from instance.json without the full render.
