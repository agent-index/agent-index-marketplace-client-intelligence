---
name: view-client
type: task
version: 2.0.0
collection: client-intelligence
description: Member-facing read task that displays a client's full data and recent changelog. Tier-resolved via the universal-floor pointer — org-public clients read from the commons; private clients read from the owner's space by folder ID (succeeds only for the owner and grantees). Callers without access get the floor view (name, owner, tier) plus how to ask the owner.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
reads_from: "/shared/client-intelligence/public-index/, /shared/client-intelligence/instances/, id:{folder_id}/"
writes_to: null
---

## About This Task

View Client renders a client's full record: template fields, extension fields, metadata, and the most recent changelog entries. Access is tier-mechanical, not probed:

1. Resolve the client via its pointer at `/shared/client-intelligence/public-index/instances/{slug}.json`.
2. `org_public` → read `instance.json` + `changelog.json` from `location.path` (every member can).
3. Private → read via `id:{location.folder_id}/...`. This succeeds for the owner and anyone the owner has granted; for everyone else Drive returns ACCESS_DENIED — which IS the floor working: render the pointer's name/owner/template/status and say *"`{name}` is private to {owner}. Ask them to share it (granting is owner-run: `@ai:grant-permission`)."*
4. `archived` / `archived-moved-private` statuses → say so; for moved stubs, follow the pointer to the current location instead of rendering the stub.

## Workflow

### Step 1: Pre-flight

`aifs_auth_status` (re-auth/halt); collection installed check.

### Step 2: Resolve

Accept slug or name (name → search the pointer index; multiple matches → disambiguate by slug). Pointer missing → *"No client `{input}`. See the directory: `@ai:list-clients`."*

### Step 3: Tier-resolved read

Per the rules above. Read `instance.json` fully; read `changelog.json` and take the most recent 10 entries (offer the rest on request). On private-tier ACCESS_DENIED → floor view + ask-the-owner guidance (never retry via other paths). On org-tier permission-denied → provisioning broken; name the admin fix (`@ai:install-collection client-intelligence`).

### Step 4: Render

Name, slug, tier (+ caller's access: yours / shared with you (read|write) / org-public), owner (+ departed annotation if `owner_departed`), template + version, status, template fields, extension fields, created/last-updated, recent changelog (actor + event + timestamp). End with relevant verbs: edit (if writable), grant/revoke (owner, private tier), transition (owner).

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
